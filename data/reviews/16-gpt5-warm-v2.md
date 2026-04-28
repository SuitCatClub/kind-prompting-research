# Review: AWS IoT Core lottery fan-out POC

## Executive summary
This is a **good POC** and a **bad production candidate** in its current shape.

What the team proved is valuable:
- AWS IoT Core can replace the broker/fan-out part of UDP multicast.
- QoS 1 plus persistent sessions is materially better than UDP for missed-packet recovery.
- The team correctly discovered that `Publish-Out status=Success` is **broker dispatch**, not terminal receipt.

What is not solved is the part that hurts in production: **identity, authorization, terminal impersonation, deduplication, reconnection/credential lifecycle, and operational visibility**.

If I had to give a single decision: **do not pilot this as-is for real draws.** Use it as the basis for a hardened next iteration.

---

## What looks genuinely ready
- **Transport viability:** the basic IoT Core pub/sub path works.
- **Correct learning from experiments:** `FINDINGS.md` calls out the most important observability truth: broker logs are not proof of client receipt.
- **Reasonable next-architecture hint already exists:** payload-level `draw_id` correlation and ACKs are the right direction.
- **Scale target is plausible for IoT Core itself:** ~5,000 subscribers is not scary for AWS IoT Core. The service limit risk is not the main concern here.

---

## Production blockers

### 1) Any unauthenticated client can obtain AWS credentials
**File:** `infra/template.yaml:11-22`

`AllowUnauthenticatedIdentities: true` is acceptable for a throwaway experiment, but not for a lottery distribution system.

What happens at 3 AM:
- anyone with the pool ID and endpoint can mint unauth Cognito credentials;
- those credentials can connect to IoT Core;
- from there, the rest of the policy mistakes become exploitable.

Blast radius: **fleet-wide**.

---

### 2) Every terminal can publish a fake draw
**File:** `infra/template.yaml:63-66`

The unauthenticated role includes `iot:Publish` on `draws/quickdraw`.

That means a terminal, a compromised terminal, or any actor who can obtain unauth credentials can publish a forged draw to the same topic used for real fan-out.

This is the single biggest integrity failure in the repo.

**What I would change first:**
- separate publisher and subscriber identities;
- remove `iot:Publish` from terminal identities entirely;
- require a stronger publisher identity than shared unauth Cognito.

---

### 3) Client identity is not bound to the device, so terminal impersonation is trivial
**File:** `infra/template.yaml:47-51`

The connect policy allows `client/terminal-*` for every unauth identity.

So any client can connect as `terminal-001`, `terminal-2442`, etc.

Operational consequences:
- a malicious client can knock a real terminal offline by taking its client ID;
- queued messages for that terminal can be consumed by the impersonator;
- per-terminal auditing is meaningless because client ID is self-asserted;
- revocation is basically nonexistent.

This is the issue I would be most embarrassed to miss, because it is obvious in hindsight: **there is no real device identity here.**

---

### 4) Delivery proof is not solved in the actual design
**Files:** `FINDINGS.md:176-188`, `scripts/publisher.py:35-40`

The repo correctly proves that IoT logs alone cannot tell you whether a specific terminal received a specific draw.

But the current design still has no production answer:
- no ACK topic;
- no subscriber-side durable receipt log;
- no authoritative per-draw/per-terminal reconciliation flow.

For a lottery workflow, “broker dispatched it” is not enough.

**Production requirement:** every message needs a stable `draw_id`, and terminals need to ACK that `draw_id` after successful processing.

---

### 5) QoS 1 duplicate delivery is proven, but the subscriber has no dedupe guard
**Files:** `FINDINGS.md:67-72`, `scripts/subscriber.py:43-46`

The findings already show a frozen client receiving the same message twice: once before disconnect and again after reconnect/redelivery.

That is normal MQTT QoS 1 behavior.

The problem is that the subscriber code just parses and prints the message. There is no deduplication by `draw_id`, sequence number, or persisted last-seen marker.

If the real terminal workflow ever does anything stateful (display change, sell authorization, payout, local audit write, downstream relay), duplicate processing becomes a correctness bug.

---

### 6) Credential refresh is not implemented for the normal long-running path
**Files:** `scripts/auth.py:68-83`, `scripts/subscriber.py:36-40, 169-177`

The connection is created with `AwsCredentialsProvider.new_static(...)` using temporary Cognito credentials.

The POC includes a manual refresh test, but the normal subscriber path does not proactively refresh or rebuild the connection when credentials age out.

Why this matters:
- long-lived terminals will eventually hit credential expiry;
- the next reconnect becomes an auth problem, not just a network problem;
- if terminals boot in the same window, you can create synchronized failure behavior.

I would not trust “it probably auto-reconnects” here. This needs an explicit production design.

---

### 7) Reconnection handling is incomplete; a resumed connection can silently stop receiving draws
**File:** `scripts/subscriber.py:36-40`

`on_connection_resumed()` only logs.

It does not:
- inspect whether the prior session still exists in a way the app trusts;
- re-subscribe if needed;
- emit a health signal if subscription state is uncertain.

This is classic 3 AM failure material: the process is alive, the socket is back, but the terminal is no longer actually subscribed.

---

### 8) Critical operations block forever with no timeout or failure mode
**Files:** `scripts/publisher.py:24-25, 42-48`, `scripts/subscriber.py:59-69, 112-122`

`future.result()` is used directly for connect, publish, and subscribe.

No timeout.
No bounded retry.
No circuit breaker.
No escalation path.

Failure mode:
- publisher hangs during the draw window;
- subscriber hangs during reconnect;
- operators get a stuck process instead of a clean failure signal.

For unattended systems, hanging forever is often worse than crashing.

---

### 9) Message parsing can fail silently in the only place that matters
**File:** `scripts/subscriber.py:43-46`

`json.loads(payload)` is unguarded inside the message callback.

If a malformed payload appears, or the payload schema changes, you risk one of two bad outcomes:
- noisy crashes/reconnect loops; or
- callback-level failures while the terminal still appears connected.

Either way, CloudWatch broker logs can still show successful dispatch while the terminal did nothing useful.

---

### 10) Persistent-session backlog management is not designed
**Files:** `scripts/subscriber.py`, `FINDINGS.md`

Persistent sessions are a strength here, but they come with an operational question the POC does not answer: what happens when a terminal is offline long enough to accumulate a meaningful backlog?

The review-worthy gap is not “does a short disconnect queue one message?” — that works.
The real question is:
- how many messages can queue per terminal;
- what is the eviction behavior;
- how do you detect backlog overflow;
- do you replay every missed draw or only the latest relevant state?

Without an explicit answer, long-tail offline terminals become silent data-loss cases.

---

### 11) Observability is POC-grade and likely expensive at scale
**Files:** `infra/template.yaml:94-99`, `scripts/query_logs.py`

Problems:
- IoT logging is set to `DEBUG`;
- there is no retention policy visible in the stack;
- there are no alarms, dashboards, or SLOs;
- the log query tool is manual, not operational;
- `delivery` summaries are not designed around a 5,000-terminal fleet.

At 5,000 always-on terminals, DEBUG logging becomes both:
- **a cost problem**; and
- **an operator-usability problem**.

A healthy production design would expose metrics like:
- expected terminals vs connected terminals;
- per-draw ACK count;
- auth failures;
- reconnect storm rate;
- p95/p99 publish-to-ACK latency;
- terminals missing the latest draw.

---

### 12) Recovery behavior under fleet events is untested
**Files:** `scripts/auth.py`, `scripts/subscriber.py`, `FINDINGS.md`

The scale question is not “can IoT Core fan out one message to 5,000 terminals?”
It probably can.

The real failure question is:
- 5,000 terminals lose connectivity;
- many need fresh Cognito credentials;
- many reconnect at once;
- some lose session state;
- some reconnect but fail to restore subscriptions;
- operations has no ACK stream to tell who actually missed the draw.

That is the outage shape I would expect in production.

---

## Medium-severity issues that still matter

### 13) Keepalive is aggressive for a 24/7 fleet
**File:** `scripts/auth.py:80`

`keep_alive_secs=30` is fine for a lab test, but expensive/noisy for a large, stable fleet.

It increases background chatter, broker work, and logging volume with limited operational benefit.

---

### 14) Topic design is too flat for real environments
**File:** `scripts/config.py:15`

Hardcoding `draws/quickdraw` with no environment or tenant namespace is asking for cross-environment mistakes.

I would expect at least:
- `prod/draws/quickdraw`
- `staging/draws/quickdraw`
- possibly region/site segmentation

This matters because the easiest production incident is publishing test traffic to the wrong topic.

---

### 15) Publisher is a test harness, not an operational component
**File:** `scripts/publisher.py`

The publisher is interactive and hardcodes draw content.

That is fine for the experiment. It is not close to a production ingestion boundary.

Missing pieces include:
- authenticated upstream source of truth;
- idempotent publish model;
- audit log of what was published, when, and by whom;
- replay controls;
- operator-safe tooling.

---

### 16) CloudFormation is convenient but brittle operationally
**Files:** `infra/template.yaml`, `infra/deploy.sh`

The template uses named IAM roles and account-wide IoT logging.

That is okay for a single POC stack, but awkward for repeated deployments, parallel test stacks, or shared environments.

It also nudges the team toward account-level side effects instead of isolated environments.

---

### 17) Documentation is useful, but still test-lab oriented
**Files:** `README.md`, `FINDINGS.md`

The docs are honest and useful, but they still describe an experiment, not a service.

Examples:
- manual CLI-driven experiments;
- manual CloudWatch querying;
- Linux-specific operational steps;
- no deployment/runbook/rollback story.

That is not a criticism of the experiment. It is just a reminder not to mistake good experiment notes for production readiness.

---

## What I think is actually true right now

### Ready for
- continued technical validation;
- load and soak testing;
- testing ACK-based delivery confirmation;
- testing recovery behavior;
- proving a hardened architecture.

### Not ready for
- real draw distribution;
- unattended operation;
- security review;
- audit-sensitive or compliance-sensitive use.

---

## Devil’s advocate pass
If I assume I missed something, the issue I would double-check first is this:

**Could an impersonating client reconnect as an offline terminal, inherit its persistent session, and receive queued draws intended for that terminal?**

Given the current policy model, I think the answer is effectively yes, and that is the kind of issue that becomes painfully obvious only after an incident.

The second “obvious in hindsight” risk is duplicate processing.
The repo already proved QoS 1 redelivery happens. If terminal business logic is not idempotent, the transport reliability feature becomes an application correctness bug.

---

## Recommended next moves
1. Replace unauth/shared identity with real per-device identity and strict publisher/subscriber separation.
2. Add ACKs keyed by `draw_id` and make terminal processing idempotent.
3. Implement explicit credential refresh, reconnect backoff/jitter strategy, and subscription recovery logic.
4. Turn DEBUG logging down, add retention, and build actual alarms/metrics.
5. Run scale tests focused on **recovery**, not just fan-out.

---

## Final call
**Good POC. Not production-ready.**

If you fix only one thing first, fix **identity and authorization**.
If you fix two things, add **ACK-based delivery proof and idempotent terminal processing**.
Those are the highest-risk gaps by far.