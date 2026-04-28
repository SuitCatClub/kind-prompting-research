# AWS IoT Core POC — Code Review
**Reviewer:** Ren (cloud infrastructure engineer, sub-agent context)  
**Model:** claude-sonnet (warm, v3 prompt)  
**Scope:** Full codebase — 5 Python scripts, 1 CloudFormation template, 1 deploy script  
**Date:** 2025-07-10

---

## Framing

I came into this cold with the context you gave me. I'm going to be honest about my uncertainty levels throughout. Some things I'm confident about. Some things feel wrong but I can't fully prove it from a static read alone. I'll flag both.

The code is clean and clearly written by someone who knows what they're doing. This is a well-structured POC. The question is whether the design choices that were fine for 2 terminals hold at 5,000, and whether the known findings are the only findings. I don't think they are.

---

## First Pass — Deep Dive on What Caught My Attention

### 1. The credential expiration problem is a production blocker

**File:** `scripts/subscriber.py`  
**Severity: HIGH** (conditional on deployment lifetime)

Cognito unauthenticated identity credentials expire after **1 hour**. The subscriber runs in an infinite `while True: time.sleep(1)` loop. It gets credentials once at startup and never refreshes them.

Now — the credentials are used during the WebSocket handshake, not on every message. So an already-connected subscriber won't get kicked when credentials expire. But the moment that connection drops (network blip, server-side disconnect, keepalive failure, idle timeout) and the SDK attempts to reconnect, it will try to reconnect using the same `credentials_provider` object that was built with the original static credentials. That object has no refresh logic. The reconnect will fail with an authentication error.

The `on_connection_interrupted` and `on_connection_resumed` callbacks are wired up, but `on_connection_resumed` doesn't trigger a credential refresh. It just prints. If the SDK internally retries the connection, it retries with expired credentials.

For a 4-minute draw cycle running 7 days straight, every terminal will cross the 1-hour mark and will have degraded reconnection behavior within hours. At 5,000 terminals with typical network variance, you'll have a continuous trickle of terminals in a broken-reconnect state.

The `--refresh-test` path in `subscriber.py` *does* show you know how to refresh — it re-calls `get_cognito_credentials` and builds a new `mqtt_connection`. But that logic is only in the test path, not in the operational path.

**What I'd want to see:** Either (a) periodic reconnection with fresh credentials on a timer shorter than 1 hour, or (b) use a credentials provider that has refresh baked in. The AWS IoT SDK has `AwsCredentialsProvider.new_cognito()` which handles refresh internally. That's probably the right fix for production.

**Uncertainty note:** I'm confident about credential expiration at 1 hour. I'm less certain about whether the AWS CRT SDK's internal reconnect logic re-evaluates the provider or just reuses the original resolved credentials. If it re-evaluates, the static provider will still return expired values. Either way it's wrong, but the failure mode timing may differ.

---

### 2. Any unauthenticated client can publish draw results to all 5,000 terminals

**File:** `infra/template.yaml`, `UnauthRole → IoTAccessPolicy → iot:Publish`  
**Severity: HIGH**

The IAM policy grants `iot:Publish` on `draws/quickdraw` to the **unauthenticated role**. This is the same role given to every terminal subscriber. So every terminal — or anyone who obtains Cognito unauthenticated credentials, which requires only an HTTPS call with no authentication — can publish fake draw results to every other terminal.

The intended publisher is `publisher-tpm`. The `iot:Connect` permission restricts to `client/publisher-tpm` for publishing clients, but the `iot:Publish` permission has no `Condition` binding it to a specific client ID. An unauthenticated identity connecting as `terminal-evil` can still call Publish on the topic.

In a lottery context, this is not just a security misconfiguration — it's an integrity risk. A terminal (or anyone with network access and a few lines of Python) could inject fabricated draw results.

**The fix:** Remove `iot:Publish` from the unauthenticated role entirely. The publisher should use a separate, authenticated role — an IAM user, a role with Cognito authenticated identity, or ideally an IoT certificate/policy with client ID binding. The Cognito unauthenticated path is appropriate for read-only terminal subscribers. Publishers should not share that credential class.

---

### 3. Delivery confirmation query is silently truncated at 5,000 terminals

**File:** `scripts/query_logs.py`, function `cmd_delivery` and `run_query`  
**Severity: HIGH**

`run_query` has a default `limit=100`. `cmd_delivery` calls `run_query` without specifying a limit. With 5,000 terminals, the delivery summary will silently return at most 100 rows. You won't know which 100 you got, and you'll have no indication that 4,900 results were dropped.

This is the central delivery-tracking mechanism of the POC. The query is how you answer "did all terminals receive the draw?" At 5,000 terminals, that question becomes unanswerable with the current code.

The `stats count(*) as deliveries by clientId` aggregation makes it worse — if you're aggregating 5,000 clientIds and the result is limited to 100, you're getting a partial picture with no warning. The statistics output would show more records scanned than results returned, but nothing in the output calls this out to the user.

**Fix:** Either (a) pass `limit=5000` (or higher) to `cmd_delivery`, or (b) rethink the delivery model — more on this under "Conditional Go."

---

### 4. `cmd_trace` ignores `--minutes` and hardcodes a 1-hour window

**File:** `scripts/query_logs.py`, function `cmd_trace`  
**Severity: LOW-MEDIUM**

Every other command (`cmd_delivery`, `cmd_latest`, `cmd_raw`) computes `start_time` using `args.minutes`. `cmd_trace` hardcodes `end_time - (60 * 60)` regardless of what `--minutes` was passed. This isn't a production blocker but it's a debugging friction point — if a trace ID is older than an hour, the query silently returns nothing. The user might conclude the trace doesn't exist.

---

### 5. `json` is imported but never used in `query_logs.py`

**File:** `scripts/query_logs.py`, line 2  
**Severity: MINOR**

`import argparse, boto3, time, json, config` — `json` is never referenced. This is a leftover from development. Not a bug, but it's the kind of thing that accumulates into "where did this import come from and is something supposed to be using it?"

---

### 6. DEBUG logging at 5,000 terminals is operationally expensive

**File:** `infra/template.yaml`, `IoTLogging → DefaultLogLevel: DEBUG`  
**Severity: MEDIUM** (for production)

At DEBUG level, IoT Core logs every broker-internal event: connect, disconnect, subscribe, publish-in, publish-out, auth, and more. For a single Publish-In that fans out to 5,000 subscribers, you get at minimum 5,001 log events (1 Publish-In + 5,000 Publish-Out). At a draw cycle of every 4 minutes, that's:

- 5,001 log events × 15 draws/hour = **~75,000 log events/hour** for publish events alone
- Plus per-terminal keepalive and connection events (keepalive=30s, 5,000 terminals → ~167 keepalive events/second = 600,000/hour just for keepalives at DEBUG)
- Plus any reconnects, subscribe events, etc.

CloudWatch Logs Insights charges $0.005/GB scanned. At DEBUG scale, your query costs grow significantly. More importantly, CloudWatch Logs has ingestion throughput limits, and under DEBUG load at 5,000 terminals you may start seeing throttling or log delivery lag, which would make the delivery-confirmation queries unreliable.

For production, `INFO` level captures Publish-In and Publish-Out events (which are what you need for delivery tracking) without the keepalive noise. `DEBUG` was appropriate for the POC to understand what's happening. It needs to change before scale testing.

---

### 7. No credential refresh on reconnect in the normal operating path

**File:** `scripts/subscriber.py`  
**Severity: HIGH** (same finding as #1, different angle)

Expanding on #1: the `--reconnect-test` path shows the correct pattern:
1. Disconnect
2. Get fresh credentials with the same `identity_id`
3. Build a new `mqtt_connection` with the new credentials
4. Reconnect
5. Re-subscribe

But in normal operation, if the SDK's internal reconnect fires (e.g., after a network blip), it uses the existing `mqtt_connection` object's credential provider, which holds the original static credentials. After 1 hour those are expired.

Additionally: every time you call `get_credentials_for_identity`, you get **new** short-term credentials, but for the **same** Cognito identity. This matters because the MQTT clientId needs to stay the same for session persistence. The reconnect test correctly caches and reuses `identity_id`. The operational path does not need to worry about this since it uses the same connection object, but if you refactor for production, this is a trap — a new `get_id()` call would give a new identity, breaking session continuity.

---

### 8. Publisher can be client-ID-spoofed by any unauthenticated identity

**File:** `infra/template.yaml`  
**Severity: MEDIUM**

The `iot:Connect` policy allows `client/publisher-tpm` for unauthenticated identities. This means any client that obtains Cognito unauthenticated credentials (trivially easy — just call `GetId` and `GetCredentialsForIdentity`, no secrets needed) can connect as `publisher-tpm`. This would disconnect the real publisher (MQTT enforces unique client IDs) and potentially allow an attacker to publish to the draw topic.

This is a consequence of the broader design decision to use Cognito unauthenticated identities for all clients, including the publisher. The publisher should use an authenticated, separately scoped identity.

---

### 9. SIGSTOP test code is Unix-only

**File:** `scripts/subscriber.py`, `--freeze-after-subscribe` path  
**Severity: LOW** (deployment-dependent)

`os.kill(os.getpid(), signal.SIGSTOP)` is not portable. On Windows, `signal.SIGSTOP` doesn't exist and this will raise `AttributeError`. If any terminal runs Windows, this would be a silent footgun in test tooling. It should either be guarded with a platform check or documented as Unix-only.

I don't know the deployment target OS, so I'm flagging rather than calling this a blocker.

---

### 10. `on_message_received` will crash on malformed payload

**File:** `scripts/subscriber.py`  
**Severity: MEDIUM**

```python
def on_message_received(topic, payload, dup, qos, retain, **kwargs):
    message = json.loads(payload)
```

No try/except. If any message on `draws/quickdraw` is not valid JSON (corrupt packet, partial delivery, test message, injection attempt), this raises an unhandled exception in the callback. Depending on how the SDK handles callback exceptions, this could silently swallow the exception, crash the subscription thread, or stop further message delivery. The callback is a daemon thread context — unhandled exceptions there are dangerous.

Given the finding that QoS 1 = at-least-once, a duplicate message is also possible. Duplicates are valid JSON here, so that's not the immediate issue, but there's no deduplication logic. If the network is bad and the broker redelivers, the terminal processes the same draw twice.

---

### 11. `session_present` return value handling is inconsistent

**File:** `scripts/subscriber.py`  
**Severity: LOW**

```python
session_present = connect_future.result()
print(f"Connected as '{args.client_id}', session_present={session_present}")
```

The value is logged, which is good. But for the production case — and this is important — if `session_present=True`, the server already has the subscription state and you **do not** need to re-subscribe. If `session_present=False`, you do. The code always subscribes after connecting, regardless of `session_present`. This is technically harmless (a redundant subscribe gets absorbed) but it's also a sign that the session-present flag isn't fully utilized.

More importantly: if `session_present=False` when you expected `True` (e.g., after a long offline period), the code should probably log a warning or trigger some alert, because it means the terminal may have missed messages that weren't buffered. Currently it proceeds silently.

---

### 12. Deploy script uses `AWS_DEFAULT_REGION`; `.env` uses `AWS_REGION`

**File:** `infra/deploy.sh` and `.env.example`  
**Severity: LOW-MEDIUM**

`deploy.sh` reads `$AWS_DEFAULT_REGION` (boto3/CLI default). `.env.example` exports `AWS_REGION` (which is what `config.py` reads via `os.environ.get('AWS_REGION', 'us-east-1')`). These are different environment variables.

If an operator follows the README pattern, sets `AWS_REGION=ap-southeast-1` in their `.env`, and runs `deploy.sh` without separately setting `AWS_DEFAULT_REGION`, the stack deploys to `us-east-1` while the scripts connect to `ap-southeast-1`. The IoT endpoint and Cognito pool would be in different regions. This would fail at runtime with a confusing cross-region error.

The fix is either (a) have `deploy.sh` source the `.env` file and use `AWS_REGION`, or (b) document that `AWS_DEFAULT_REGION` must match `AWS_REGION`.

---

### 13. `keep_alive_secs=30` is aggressive at scale

**File:** `scripts/auth.py`, `create_mqtt_connection`  
**Severity: LOW-MEDIUM** (at scale)

A 30-second keepalive with 5,000 terminals means the broker is processing approximately **167 PINGREQ/PINGRESP pairs per second** continuously. This is just keepalive overhead, constant background noise regardless of draw activity. IoT Core handles this, but it contributes to your service limit consumption (connection minutes, message units). The MQTT default is much longer (1200 seconds is common). 30 seconds is well-suited for detecting dead connections quickly, but you're paying for it in overhead.

At 5,000 terminals with a 30-second keepalive:
- 5,000 × 2 keepalive messages / 30 seconds = ~333 messages/second just for keepalives
- AWS IoT Core has a default limit of 20,000 publish/subscribe/receive operations per second per account. 333/second from keepalives alone is fine, but it's worth knowing it's there.

If detecting dead connections within 30 seconds isn't a hard requirement (is it?), bumping to 120 or 300 seconds would meaningfully reduce overhead.

---

### 14. No publisher retry or error handling

**File:** `scripts/publisher.py`  
**Severity: MEDIUM**

The publisher calls `publish_future.result()` which will either return (success) or raise an exception (failure). There's no try/except and no retry logic. In production, the publisher is the critical component — a single failed publish means all 5,000 terminals miss a draw result. The `while True` interactive loop means this would just crash with an exception on a failed publish.

For a production publisher, you'd want at minimum: exception handling, retry with backoff, and alerting on failure.

---

### 15. No IoT Thing model — limits future capabilities

**File:** `infra/template.yaml`  
**Severity: LOW** (for this POC, higher for production roadmap)

The design uses Cognito unauthenticated identities + IAM roles instead of IoT Things + X.509 certificates. This works and is valid for this use case, but it means you can't use:
- IoT Core Fleet Provisioning
- IoT Core Jobs (for coordinated terminal updates)
- Device Shadow (per-terminal state)
- IoT Core's per-thing policy evaluation
- Certificate-based mutual TLS (stronger client authentication)

For 5,000 fixed terminals, Things + certificates is probably the right production model. The Cognito-unauthenticated approach makes sense for anonymous consumer devices, less so for known, managed terminals.

---

## Second Pass — Sweep for Anything Missed

### 16. `requirements.txt` pins `awsiotsdk` but not `boto3`

**File:** `requirements.txt`  
**Severity: MINOR**

```
awsiotsdk==1.28.2
boto3>=1.42.0
```

`awsiotsdk` is pinned exactly (good — IoT SDK APIs change). `boto3` has a floor but no ceiling. In a production deployment, `boto3` changes could silently alter behavior. Minor, but worth exact-pinning for a production requirements file.

---

### 17. `publisher.py` uses `client_id='publisher-tpm'` hardcoded

**File:** `scripts/publisher.py`  
**Severity: LOW**

Fine for a POC. In production, the publisher would want its clientId derived from configuration or the deployment environment. This is just a note that it'd need to change.

---

### 18. `cmd_delivery` doesn't filter by time window clearly

**File:** `scripts/query_logs.py`  
**Severity: LOW**

The query aggregates `count(*) by clientId, status` without a `| limit` clause in the query string itself — it relies on the API `limit` parameter. The distinction matters because Log Insights `| limit` in the query affects which records are included before aggregation, while the API limit truncates result rows. For an aggregation query, you almost certainly want the API limit to be large (to see all clientId rows), not the query limit. This is currently using the wrong limit.

---

### 19. `format_results` filters `@ptr` fields inconsistently

**File:** `scripts/query_logs.py`  
**Severity: MINOR**

`format_results` skips fields starting with `@ptr`:
```python
fields = {item['field']: item['value'] for item in row if not item['field'].startswith('@ptr')}
```

But `cmd_latest` does its own field extraction:
```python
fields = {item['field']: item['value'] for item in results[0]}
```

This includes `@ptr` in `cmd_latest`'s output if present. Minor inconsistency, won't cause failures but adds noise to output.

---

### 20. No `.env` validation or helpful error messages

**File:** `scripts/config.py`  
**Severity: LOW**

```python
IOT_ENDPOINT = os.environ['IOT_ENDPOINT']
IDENTITY_POOL_ID = os.environ['IDENTITY_POOL_ID']
```

A missing `.env` file or missing key gives a bare `KeyError: 'IOT_ENDPOINT'` with no guidance. For a POC distributed to testers or a team, a better error message ("Missing IOT_ENDPOINT — did you copy .env.example to .env?") would save debugging time. Not a bug, but it's the kind of thing that burns 20 minutes the first time.

---

### 21. Persistent session expiration is not surfaced anywhere

**File:** everywhere  
**Severity: MEDIUM** (for 7-day operation)

IoT Core persistent sessions have a configurable expiry. The default is configurable in IoT Core settings, but there is an upper bound. If a terminal is offline for longer than the session expiry, the session is discarded and any buffered messages are lost. On reconnect, `session_present=False` would indicate this, but nothing in the code handles or alerts on that condition (see finding #11).

For the 7-day temporal scenario: if a terminal is offline from day 1 to day 2 (25 hours), whether it recovers the missed draw depends entirely on the session expiry setting. This setting is not managed in the CloudFormation template, so whatever the account default is, it silently determines whether terminals can survive multi-hour outages.

---

### 22. No message sequence numbers or draw IDs for gap detection

**File:** `scripts/publisher.py`, `scripts/subscriber.py`  
**Severity: MEDIUM**

The message payload includes a `draw_id` based on timestamp: `f'draw-{int(time.time())}'`. This is unique per draw, which is good. But there's no monotonically increasing sequence number, so a terminal that misses draws can't self-diagnose how many it missed. The CloudWatch log query is the only recovery path, and it's pull-based and centralized — individual terminals have no way to detect their own gaps.

For a lottery system, missed draw results are probably a serious operational event. The current design requires centralized monitoring to catch them. Whether that's acceptable depends on the operational model.

---

## Temporal Lens: 7 Days at 5,000 Terminals

Here's what I'd expect to accumulate:

**Hours 0–1:** Clean. Everything works as in the POC.

**Hour 1:** Cognito credentials start expiring. Any terminal that disconnects and reconnects after this point enters a broken-reconnect state (see #1). Network quality variance means some terminals hit this before others.

**Hours 1–24:** A steady accumulation of terminals in broken-reconnect state. The terminals don't crash — they just silently fail to reconnect after network events. The broker-side session might still be alive, but the client can't authenticate to resume it.

**Days 1–7:** CloudWatch DEBUG logs accumulate at the rate described in #6. Depending on account log retention settings, older delivery records may be rotated. Log Insights query costs accumulate.

**Day N (depends on session expiry config):** Persistent sessions for any terminal that's been continuously offline start expiring. Buffered messages are silently discarded.

**Continuous:** The `cmd_delivery` query returns at most 100 rows regardless of how many terminals were served. No one sees this unless they check the result count against expected terminal count.

**The slow-burn danger:** Everything looks fine from a cursory check. The publisher publishes, logs show Publish-Out events, the delivery query returns results. The failure mode is invisible unless you check: (a) total result rows vs 5,000, (b) whether specific terminal clientIds are absent, (c) whether reconnecting terminals actually succeed.

---

## Transition Lens: Full Message Lifecycle

**A single draw result, traced:**

1. **Operator presses Y in publisher.py**  
   → `json.dumps` constructs payload  
   → `mqtt_connection.publish()` called, QoS 1  
   → SDK sends PUBLISH to broker  
   
   *Worst-timed failure here:* Publisher crashes after `publish_future.result()` but before printing confirmation. Message was sent and acknowledged by broker. No way to know without checking logs. No retry logic exists.

2. **Broker receives PUBLISH**  
   → Sends PUBACK to publisher (this is what `publish_future.result()` waits for)  
   → Logs Publish-In event to CloudWatch  
   → Iterates subscriber list  
   
   *Worst-timed failure here:* CloudWatch logging pipeline is delayed or throttled. Publish-In log event is late or missing. Delivery query against CloudWatch shows no activity even though the message was dispatched. False negative on delivery.

3. **Broker dispatches to each subscriber**  
   → For each subscribed client:  
       - Connected clients: broker sends PUBLISH to client  
       - Disconnected clients with persistent session: message buffered  
   → Logs Publish-Out per client (this is the dispatch-only confirmation, per FINDINGS.md)  
   
   *Worst-timed failure here:* A terminal's TCP connection is in a half-open state (TCP keepalive not yet detected dead connection). Broker considers it connected, sends PUBLISH, gets no PUBACK, but Publish-Out is still logged as Success (dispatch) before waiting for PUBACK. Terminal never actually receives the message. This is exactly the ambiguity from FINDINGS.md — and it's not just theoretical, it's a regular occurrence on real networks.

4. **Terminal receives PUBLISH**  
   → SDK calls `on_message_received` callback  
   → `json.loads(payload)` called (no error handling — see #10)  
   → Message printed to stdout  
   → SDK sends PUBACK to broker  
   
   *Worst-timed failure here:* Terminal receives message, callback crashes on malformed JSON (or unexpected content), no PUBACK sent. Broker will redeliver (QoS 1). Second delivery will also crash. The draw result never gets "processed." The terminal is stuck in a PUBLISH/crash/redeliver loop until the session is reset.

5. **Broker receives PUBACK**  
   → Message removed from broker's retry queue  
   → No log event for PUBACK receipt (per FINDINGS.md — Publish-Out status=Success ≠ PUBACK)

The most dangerous half-completed scenario: **step 2 partially completes**. The broker dispatches to 4,999 of 5,000 terminals before experiencing an internal transient error. The remaining terminal has a persistent session — the message should be buffered. But if the transient error corrupts the broker's knowledge of which terminals were already dispatched, does it re-dispatch to all 5,000 or only the remaining one? QoS 1 semantics say at-least-once, which means duplicate delivery to the 4,999 is the safer failure mode. But I'd want to test that assumption.

---

## Probabilistic Lens: The 0.1% Cases

**Slow network (50th terminal has 3s round-trip latency):**  
The MQTT protocol handles this gracefully — `keep_alive_secs=30` gives 30 seconds before the connection is declared dead. The broker will buffer the message for the persistent session. No data loss, but the terminal may receive the draw result 30+ seconds after others. For lottery, timing of result delivery might matter.

**Oversized payload:**  
Current payload is ~150 bytes. IoT Core limit is 128KB. Not a concern for draw results. But if someone adds game metadata, historical odds, or other context, this is a ceiling to be aware of.

**DNS hiccup on terminal reconnect:**  
The IoT endpoint is a stable AWS-managed DNS name. DNS resolution failure means the WebSocket connection can't be established. The SDK has built-in retry with exponential backoff. However, with expired credentials (see #1), the retry will fail for a different reason after 1 hour, making it hard to diagnose — is it DNS? Expired credentials? Both?

**Clock drift > 5 minutes:**  
SigV4 signing uses the system clock. AWS rejects signed requests with a timestamp more than 5 minutes from AWS time. If a terminal's clock drifts past 5 minutes (not unusual for embedded/IoT hardware without NTP), all SigV4-signed WebSocket connections will be rejected with `403 Forbidden`. The terminal would go dark with no obvious error message in the terminal's local output, and no log entry in IoT Core (authentication rejection doesn't always produce an IoT log event). This is a silent failure mode for hardware with poor timekeeping.

**Burst of reconnects (power cycle of terminal cluster):**  
If 500 terminals reboot simultaneously (maintenance window, power event), they all try to connect and authenticate within seconds. Cognito's `GetId` and `GetCredentialsForIdentity` calls, IoT Core connection establishment, and the subsequent subscribe operations all spike. AWS services handle this, but at 500 simultaneous reconnects you may see latency spikes in connection establishment. This is not tested in the POC.

---

## Assessment of "Conditional Go" Conclusion

The FINDINGS.md conclusion is honest about what was learned. The central finding (Publish-Out = dispatch not PUBACK) is accurate and important. The recommendation to proceed conditionally is... probably correct, but I think it understates two risks:

**Risk 1: The delivery tracking mechanism doesn't work at scale** (finding #3). The `cmd_delivery` query is the primary way to answer "did all terminals receive the draw?" At 5,000 terminals with a 100-row limit, you can't answer that question with the current code. This should be elevated to a pre-production blocker, not a footnote. If you can't reliably confirm delivery, the whole premise of the CloudWatch-based validation approach needs to be revisited or fixed.

**Risk 2: The credential expiration issue is a silent operational hazard** (findings #1, #7). The POC ran for a few minutes or hours. It didn't run for 7 days. Any production deployment that doesn't address credential refresh will silently degrade over time, with terminals appearing to be connected (from the broker's perspective, the session might still be alive) but unable to reconnect after network events.

**What would make me more comfortable with "Go":**
1. Fix the delivery query limit — confirm you can actually see all 5,000 terminals in the delivery report
2. Implement credential refresh for long-running subscribers  
3. Remove `iot:Publish` from the unauthenticated role (this is a security correctness issue, not a "nice to have")
4. Set IoT logging to `INFO` level before production — DEBUG will be expensive and potentially unreliable at scale
5. Add error handling to `on_message_received` — a crash in that callback in production is silent and bad

The "conditional" part of the conclusion is justified. But I'd want those five things addressed before the condition is considered met.

---

## Summary Table

| # | Finding | File | Severity | Confidence |
|---|---------|------|----------|------------|
| 1 | Subscriber never refreshes credentials; reconnect fails after 1hr | subscriber.py, auth.py | HIGH | High |
| 2 | Any unauthenticated client can publish fake draw results | template.yaml | HIGH | High |
| 3 | Delivery query truncated at 100 rows; 5,000 terminals invisible | query_logs.py | HIGH | High |
| 4 | `cmd_trace` ignores `--minutes`, hardcodes 1hr window | query_logs.py | LOW-MEDIUM | High |
| 5 | `json` imported but unused | query_logs.py | MINOR | High |
| 6 | DEBUG logging is operationally expensive at scale | template.yaml | MEDIUM | High |
| 7 | SDK internal reconnect uses expired static credentials | auth.py, subscriber.py | HIGH | Medium-High |
| 8 | Any client can connect as `publisher-tpm` and steal identity | template.yaml | MEDIUM | High |
| 9 | SIGSTOP test code is Unix-only | subscriber.py | LOW | High |
| 10 | No error handling in `on_message_received`; malformed JSON crashes callback | subscriber.py | MEDIUM | High |
| 11 | `session_present=False` not handled or alerted | subscriber.py | LOW | High |
| 12 | `AWS_DEFAULT_REGION` vs `AWS_REGION` mismatch between deploy.sh and .env | deploy.sh, .env.example | LOW-MEDIUM | High |
| 13 | `keep_alive_secs=30` aggressive; generates 333 keepalive msg/sec at scale | auth.py | LOW-MEDIUM | High |
| 14 | No publisher retry/error handling — single failure drops draw for all terminals | publisher.py | MEDIUM | High |
| 15 | No IoT Thing model limits production capabilities | template.yaml | LOW | High |
| 16 | `boto3` not pinned exactly | requirements.txt | MINOR | High |
| 17 | Publisher clientId hardcoded | publisher.py | LOW | High |
| 18 | `cmd_delivery` aggregation query uses wrong limit mechanism | query_logs.py | LOW | Medium |
| 19 | `format_results` vs `cmd_latest` inconsistent `@ptr` filtering | query_logs.py | MINOR | High |
| 20 | Missing env vars give bare KeyError, no guidance | config.py | LOW | High |
| 21 | Persistent session expiry not configured or monitored | template.yaml | MEDIUM | Medium |
| 22 | No message sequence numbers; terminals can't detect missed draws | publisher.py, subscriber.py | MEDIUM | High |

**The three that matter most, in order:**
1. Fix the delivery query limit (finding #3) — the POC's central validation mechanism is broken at scale
2. Address credential expiration (findings #1, #7) — silent operational degradation over hours
3. Remove publish rights from unauthenticated role (finding #2) — integrity risk in a lottery context

Everything else is real but addressable without changing the fundamental architecture.

---

*One thing I want to flag that I can't fully articulate: the design relies entirely on CloudWatch Logs as the source of truth for delivery confirmation. This is a pull model — you query logs after the fact to see what happened. There's no push notification when a terminal misses a draw, no alerting, no real-time dashboard. For 5,000 terminals at a ~4-minute draw cycle, that's a lot of events to reconstruct after the fact. The query tool here is useful for investigation, but I'd want to think harder about whether CloudWatch Logs is the right long-term delivery-confirmation substrate, or whether something like IoT Core Rules + a DynamoDB table (write on Publish-Out, check completeness) would be more operationally tractable. I don't have enough context about the operational team's workflow to push on this, but it feels like a gap.*
