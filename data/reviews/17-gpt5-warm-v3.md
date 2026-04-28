# AWS IoT Core POC Review

**Reviewer:** GPT-5 via Copilot CLI  
**Date:** 2026-04-28  
**Scope:** Static review of the provided POC code, CloudFormation, README, findings, and planning notes  
**Caveat:** I do not know the real terminal hardware, network topology, or existing monitoring stack. Where that missing context could change the recommendation, I call it out.

## Executive Summary

This is a **good POC** and a **not-yet-safe production design**.

What I trust from this repo:
- AWS IoT Core can probably replace the current UDP multicast fan-out path.
- The team answered the most important ambiguous question: **`Publish-Out status=Success` is broker dispatch, not proof of terminal receipt**.
- QoS 1 plus persistent sessions give you a much better recovery story than multicast.

What I do **not** think is ready for production:
- publisher trust and terminal identity binding
- end-to-end proof that terminal X processed draw Y
- unattended long-running reconnect and credential lifecycle
- fleet-scale observability and alerting
- scale validation anywhere near 5,000 terminals

If you forced me to summarize the risk in one sentence: **today this design can probably fan out messages, but it cannot yet safely prove integrity or receipt.**

## What Looks Good

1. The code is small and understandable.
2. The POC findings are honest and useful, especially the Publish-Out/PUBACK conclusion.
3. `draw_id` already exists in the payload, which is the right anchor for deduplication and ACKs later.
4. The experiments around reconnect, queueing, and refresh are the right experiments.
5. The repo documents limitations instead of hiding them.

## Ready Now vs Not Ready

### Ready for POC use
- proving basic MQTT fan-out through IoT Core
- validating QoS 1 retry behavior
- validating short offline windows with persistent sessions
- manual CloudWatch log inspection during experiments

### Not ready for production use
- authentication/authorization model
- tamper-resistant publisher path
- terminal identity model
- receipt confirmation / reconciliation
- reconnect storm handling
- 24x7 credential refresh lifecycle
- monitoring, alarming, and operator workflow
- scale claims about 5,000 terminals

## Findings That Matter

| Severity | Area | Finding | Why it matters |
|---|---|---|---|
| Critical | Security / integrity | Unauthenticated Cognito identities can publish to the live draw topic (`infra/template.yaml:15`, `infra/template.yaml:63-66`) | Any party that can obtain unauth credentials can inject fake draws unless strong external controls exist |
| Critical | Identity | Same unauth trust path can connect as `publisher-tpm` or any `terminal-*` client (`infra/template.yaml:47-51`) | MQTT client IDs are not identity; impersonation or duplicate-ID disconnect loops can take out real terminals |
| High | Delivery semantics | There is still no application-level proof that a terminal processed a draw | Broker dispatch is not business confirmation |
| High | Reliability | `auth.py` uses static temporary credentials (`scripts/auth.py:68-72`) and normal subscriber flow has no automatic refresh | Long-lived terminals will eventually fail on reconnect after credentials age out |
| High | Reliability | `subscriber.py` logs reconnect state but does not heal if session state is lost (`scripts/subscriber.py:36-40`, `scripts/subscriber.py:112-166`) | A terminal can look connected but stop receiving draws |
| High | Observability | `query_logs.py` is manual and `delivery` inherits a default limit of 100 rows (`scripts/query_logs.py:12`, `scripts/query_logs.py:78-91`) | At 5,000 terminals, the default reporting path is misleading |
| Medium | Scale | No fleet reconnect jitter, no quota strategy, no scale test | One WAN flap can become a herd event |
| Medium | Operations / cost | IoT logging is account-wide DEBUG with no retention or alarms in the template (`infra/template.yaml:94-99`) | Noisy, expensive, and risky if this AWS account hosts other IoT workloads |
| Medium | Correctness | Subscriber accepts any JSON with no schema, dedup, version, or signature checks (`scripts/subscriber.py:43-46`) | Duplicate, malformed, stale, or malicious payloads have undefined behavior |
| Medium | Publish path | Publisher is interactive and blocks indefinitely on futures (`scripts/publisher.py:24-25`, `scripts/publisher.py:43-48`) | A hung publish path at draw time becomes an outage |

## Detailed Review

### 1. `infra/template.yaml` is the biggest production risk

This template is doing lab-friendly things that would make me very nervous in production:

- `AllowUnauthenticatedIdentities: true` (`infra/template.yaml:15`)
- the unauthenticated role can `iot:Publish` to `draws/quickdraw` (`infra/template.yaml:63-66`)
- the same role can `iot:Connect` as `publisher-tpm` (`infra/template.yaml:47-51`)
- terminals are authorized by a broad `terminal-*` wildcard (`infra/template.yaml:50`)
- account-wide IoT DEBUG logging is enabled by default (`infra/template.yaml:94-99`)

For a POC, that is understandable. For a lottery-result distribution system, it is not.

My strongest recommendation is to split principals immediately:
- **publisher principal**: may publish to `draws/quickdraw`; should not be the same trust path as terminals
- **terminal principal**: may connect with only its own client ID, subscribe/receive on the draw topic, and publish only to an ACK topic if you add one

If the production environment already has strong outer controls — private connectivity, egress filtering, store-to-core VPN, device attestation, or network allowlists — that lowers exposure. But nothing in this repo proves those controls exist, so I would not assume them.

### 2. `scripts/auth.py` has the classic week-one failure mode

`scripts/auth.py` creates a static credentials provider from temporary Cognito credentials (`scripts/auth.py:68-72`). That is fine for a short experiment. It is not fine for unattended terminals.

The failure mode is simple:
1. terminal connects successfully
2. credentials expire later
3. existing WebSocket stays up, so nobody notices
4. some ordinary network wobble forces reconnect
5. reconnect now depends on expired credentials unless the app refreshes and rebuilds the connection

The repo does have refresh logic, but only in explicit test branches in `subscriber.py` (`scripts/subscriber.py:127-166`). That is good validation work, not a production lifecycle.

### 3. `scripts/subscriber.py` can create “connected but dark” terminals

This is the failure I would expect to hurt you at 3 AM.

The callbacks log interruption/resume (`scripts/subscriber.py:36-40`), but they do not repair state. If the session is gone, or if the SDK reconnect behavior differs from what you expect:
- reconnect can succeed
- the process can still look alive
- the terminal may no longer have the subscription you think it has
- and you may not notice until stores complain about missing draws

Other important gaps in this file:
- no timeout handling around connect/subscribe/disconnect futures
- `json.loads(payload)` is unguarded (`scripts/subscriber.py:44`)
- no schema validation
- no deduplication for QoS 1 duplicates
- no durable local receipt record
- no ACK back to a central system

One nuance: the AWS CRT may auto-resubscribe in some reconnect cases. If your actual terminal client will rely on that behavior, verify it explicitly under your chosen SDK version and fault scenarios. I would still want the application to detect and alarm on `session_present=False` rather than merely print it.

### 4. `scripts/query_logs.py` is a canary for production readiness

This script is useful for experiments. It is not a production reporting system.

Most obvious code issue: `run_query()` defaults to `limit=100` (`scripts/query_logs.py:12`) and `cmd_delivery()` does not override it (`scripts/query_logs.py:78-91`). At 5,000 terminals, a “delivery summary” that silently caps at 100 rows is dangerous because it can look legitimate while omitting most of the fleet.

Even beyond that:
- it is manual only
- it is post-hoc
- it has no alarming path
- it cannot answer “which terminals processed draw X?” because IoT logs do not carry payload identity through Publish-Out
- it relies on logs that the team already proved are insufficient as receipt confirmation

This is why I agree with the repo’s own conclusion: if you need operational proof, you need a payload-level confirmation model.

### 5. `scripts/publisher.py` is a demo tool, not an operational publish path

The publisher is interactive and blocks indefinitely on futures (`scripts/publisher.py:24-25`, `scripts/publisher.py:43-48`). That is perfectly reasonable for a POC. It is not the shape of something I would trust to publish live lottery draws.

Missing production properties:
- timeout and retry policy
- durable input/output record
- idempotent publish behavior
- stronger sequencing than `draw-{int(time.time())}` (`scripts/publisher.py:36`)
- payload versioning and integrity protection
- explicit integration contract from the upstream draw-generation system

For this domain, I would want a durable source-of-truth record for each draw and a controlled publish pipeline, not a prompt loop.

### 6. Logging and operations are still experimental

The template enables IoT logging at DEBUG account-wide (`infra/template.yaml:98`). That is useful during discovery; it is not a steady-state operating posture.

Risks here:
- unnecessary log volume
- harder operator signal-to-noise
- larger Logs Insights scans
- accidental coupling to other IoT workloads in the same account
- no retention policy in this template
- no alarms or dashboards

If this account is dedicated and low-volume, the cost problem may be mild. If it is shared, DEBUG-at-account-scope is the kind of thing that surprises teams later.

### 7. Small issues that are canaries

These are not headline failures, but they matter:

1. `query_logs.py delivery` effectively caps results at 100 rows.
2. Multiple `.result()` calls have no timeout.
3. Subscriber trusts any JSON payload.
4. There is no dedup behavior despite QoS 1 duplicates being explicitly observed.
5. `draw_id` is second-based, which is weak if two publishes can happen close together.
6. No message schema version, signature, or anti-replay protection.
7. No explicit deployment separation for environments.

Small issues like these are often the smoke that tells you the operational model is still a prototype.

## Category View

### Security

The biggest issue is **publisher trust**.

Before any pilot, I would require:
1. publisher and terminal principals split cleanly
2. terminals made read-only except for a deliberate ACK topic
3. binding between principal and allowed client ID
4. payload integrity protection or another tamper-evident mechanism
5. replay/duplicate handling rules

### Scalability

This repo does not prove 5,000-terminal behavior.

The missing scale questions are:
- reconnect storm behavior after a shared outage
- Cognito quota behavior during refresh bursts
- end-to-end fan-out latency percentiles, not just anecdotal “it arrived”
- persistent-session backlog behavior for offline subsets
- whether your network causes correlated reconnect waves around draw time

### Reliability

Right now the design gives you **at-least-once broker delivery**, not **reliable terminal processing**.

That gap matters. QoS 1 helps with transport. It does not tell you whether the application processed the draw, displayed it, or accidentally processed it twice.

### Observability

Today the answer to “which terminals missed draw 184223?” is basically “infer it from ad hoc logs.”

That is not enough for production. You need one of these:
- ACK topic keyed by `draw_id`
- durable terminal-side receipt logs collected centrally
- or a control-plane reconciliation store that tracks expected terminals vs confirmed terminals per draw

### Operational

I do not see:
- alarms
- retention controls
- runbooks
- an automated credential rotation/reconnect model
- safe environment separation
- rollout / rollback discipline

That is normal for a POC. It also means there is still real engineering work between “promising” and “operable.”

### Cost

The cost risk is less about broker traffic and more about **using CloudWatch Logs Insights as the operating model**.

If operators end up querying DEBUG logs repeatedly to answer delivery questions, you can burn money and time on the wrong telemetry path. Exact cost depends heavily on your real fleet behavior and account layout, which I do not know.

## Lens 1: The Temporal Lens

**Imagine this has been running for 7 days. What breaks first?**

My best guess is not a total outage. It is a **partial, quiet fleet degradation**.

Likely sequence:
1. terminals stay connected long enough that credential expiry remains latent
2. some subset experiences ordinary WAN or site instability
3. those terminals now need a real refresh/reconnect lifecycle
4. some reconnect cleanly, some lose session state, some become “connected but dark”
5. the first alert is not an infrastructure alarm; it is a business symptom: “some stores did not get the draw”

The most likely 2 AM day-7 alert, if you have any alerting at all, is probably one of these:
- spike in disconnect/reconnect events for a subset of terminals
- unexplained drop in expected per-draw confirmations once you add them
- or, more realistically with the current repo, a human report that a subset of stores missed a draw

The other week-one degradation is observability itself:
- DEBUG logs accumulate
- manual queries get noisier
- the `delivery` summary is already structurally wrong at fleet scale because of the 100-row limit
- operators still lack a direct “dark terminal” signal

If your terminals run on flaky WAN links, cellular, or resource-constrained hardware, I would raise this risk substantially.

## Lens 2: The Transition Lens

**Trace one draw from generation to confirmed receipt.**

1. **Draw generated upstream**  
   This repo does not show a durable upstream record or sequencing contract. If the publishing process is down when the draw appears, there is no shown outbox or replay path.

2. **Publisher serializes the message** (`scripts/publisher.py:35-40`)  
   Failure modes: weak ID generation, no schema version, no signature, no anti-replay marker.

3. **Publisher connects and publishes to IoT Core** (`scripts/publisher.py:24-25`, `scripts/publisher.py:43-48`)  
   Failure modes: indefinite hang, expired credentials on reconnect, no bounded retry, no durable audit tying business event to broker publish.

4. **IoT Core accepts Publish-In**  
   Good: broker accepted the message.  
   Bad: this is still not proof of terminal processing.

5. **IoT Core fans out Publish-Out to subscriber sessions**  
   Failure modes:
   - healthy connected terminal: success path
   - half-broken connection: broker logs success but terminal may not process promptly
   - offline terminal with persistent session: queued for later
   - session gone or backlog exceeded: message loss risk
   - duplicate client ID: real terminal displaced by another connection

6. **Subscriber callback runs** (`scripts/subscriber.py:43-46`)  
   Failure modes: malformed JSON raises, duplicate message processed twice, stale draw accepted, application failure after transport-level success.

7. **Terminal processes/displays draw**  
   This transition is not observable from the current design.

8. **All 5,000 terminals confirmed receipt**  
   This transition does not exist today. That is the core architectural gap.

The heart of the transition problem is simple: **the system currently proves broker activity better than business completion.**

## Lens 3: The Probabilistic Lens

At the stated scale:
- 5,000 terminals
- 1 draw every 4 minutes
- 15 draws/hour
- 360 draws/day
- **1,800,000 terminal-delivery opportunities per day**

That changes how I think about “rare.”

- If the true per-terminal per-draw miss rate is **0.1%**, you should expect about **1,800 missed deliveries per day**.
- If it is **0.01%**, that is still about **180 missed deliveries per day**.
- If duplicate processing happens at **0.01%**, that is also **180 duplicate terminal events per day** unless dedup is explicit.

At this scale, the 0.1% case is not exotic; it is operationally routine.

Correlated failures worry me more than independent ones:
- regional DNS issue
- WAN/MPLS flap
- store power events
- coordinated device reboot
- AWS-side reconnect pressure

Even if only **10%** of terminals are affected by a shared network event, that is **500 devices** trying to reconnect around the same time. If reconnect also requires Cognito refresh, you need to know your quota posture and you probably want jitter. I cannot size that precisely without your actual network and AWS quota settings, but I would treat it as a primary design input.

A smaller but very practical probabilistic issue: if your reporting tool silently caps at 100 rows, then on a 5,000-terminal draw you are not measuring a fleet; you are sampling one small slice of it and risking false confidence.

## Devil’s Advocate Pass

**What would I be embarrassed to have missed?**

Two things:

1. **The obvious-in-hindsight issue:** the design still does not answer the real operational question, which is “did terminal X actually process draw Y?”  
   That is the question operators and auditors care about, and the current system does not answer it.

2. **The integrity issue:** in this repo, publisher trust is too weak for the domain.  
   If an unauthorized client can publish or impersonate a terminal/publisher, everything else becomes secondary.

A smaller but still embarrassing miss would be overlooking that the current `delivery` query path effectively falls over at 100 rows. That is exactly the kind of “minor” issue that becomes a major source of false reassurance.

## Recommended Next Steps

1. **Fix trust first**: separate publisher from terminals; remove terminal publish permission on the live draw topic.
2. **Choose a receipt model**: ACK topic keyed by `draw_id`, or another durable confirmation path.
3. **Implement the real reconnect lifecycle**: automatic credential refresh, explicit handling for lost session state, bounded retries, and jitter.
4. **Add idempotence and integrity**: dedup on `draw_id`, schema versioning, and tamper evidence.
5. **Stop treating log forensics as operations**: add metrics, alarms, retention, and a per-draw reconciliation view.
6. **Run a soak test**: long-lived connections plus forced faults over days, not minutes.
7. **Run a scale test**: even if you cannot hit 5,000 immediately, push far enough to learn where reconnect, logging, and Cognito start bending.

## Final Judgment

As a POC, this is solid work.

As a production candidate for lottery-result fan-out, my answer is **not yet**.

The good news is that the remaining work is not mysterious. It is mostly hardening work:
- trust
- confirmation
- recovery
- visibility
- scale discipline
