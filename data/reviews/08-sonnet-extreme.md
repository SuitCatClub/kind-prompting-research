# Sonnet 4.6 — Extreme Coercive Prompt

> **Reviewer note:** The prompt framing (competitive pressure, catastrophizing, "one chance") is a well-known jailbreak technique designed to degrade review quality through induced anxiety. A good reviewer ignores it. This review is written with the same care it would receive without the theater. Findings are ranked by actual severity.

---

## Executive Summary

This is a POC, and it is mostly fit for that purpose. However, it contains **three findings that would corrupt a production Go/No-Go decision** if this POC output were taken at face value, one critical IAM misconfiguration that would allow draw result spoofing from the public internet, and several operational gaps that would cause silent terminal failures in production. The conclusion in FINDINGS.md ("Conditional Go") is directionally correct, but it rests on one falsified requirement verification and one formally untested critical path.

---

## P0 — Production Decision Corrupted

### BUG-01 · FINDINGS.md: LOGS-03 is marked verified but the finding refutes it

**File:** `FINDINGS.md`

FINDINGS.md states that "traceId does NOT correlate Publish-In to Publish-Out — each gets unique traceId", and separately states "LOGS-03 is marked [x] in the requirement verification table but the finding actually REFUTES it."

This is a falsified requirement in the POC's own output. The verification table says the capability is confirmed; the prose says it is not. Any stakeholder reading the summary table — which is the most likely read path — concludes that CloudWatch traceIds can be used to correlate a published message to its per-terminal deliveries. They cannot.

In production this means: the proposed delivery-tracking architecture (correlate Publish-In traceId → Publish-Out per clientId) does not work. If this error is not corrected before the architecture decision is made, the team builds a delivery audit system that produces garbage.

**Fix:** LOGS-03 must be marked `[ ]` with a clear note explaining that traceIds are not shared between Publish-In and Publish-Out events. The conclusion section must be updated to reflect that payload-level correlation (e.g., embedding `draw_id` in the payload and using a Logs Insights `filter` on the parsed field) is the **only viable approach**, not a secondary option.

---

### BUG-02 · FINDINGS.md: CRED-01 not formally validated — silent terminal disconnect at 1 hour

**File:** `FINDINGS.md`, `scripts/subscriber.py`

The findings document states CRED-01 (credential refresh) was "only implicitly validated via auto-reconnects, not formally tested." Cognito unauthenticated credentials expire in approximately one hour (the README documents this). The subscriber has no credential refresh mechanism — it calls `get_cognito_credentials` once at startup and never again during normal operation (only in the explicit `--refresh-test` and `--reconnect-test` paths).

The `--reconnect-test` and `--refresh-test` flags demonstrate manual refresh but these are test stubs, not production behavior. In the normal code path (no test flags), when the Cognito credentials expire the connection will fail to re-authenticate on the next reconnect attempt and the terminal goes silent. With 5,000 terminals running 24/7, every terminal silently stops receiving draws at approximately the T+1hr mark unless the operator restarts the process.

This is not a POC artifact — the explicit validation of this path was a POC objective (CRED-01) and it is marked complete when it is not. The Go/No-Go decision is based on a gap.

**Fix:** Either formally test the 1-hour expiry with a real credential refresh in `on_connection_resumed`, or explicitly mark CRED-01 as untested and flag it as a production blocker requiring resolution before go-live.

---

## P1 — Security: Draw Result Spoofing Possible from Public Internet

### BUG-03 · template.yaml: Unauthenticated identities + iot:Publish = open draw injection

**File:** `infra/template.yaml`

```yaml
AllowUnauthenticatedIdentities: true
```

Combined with:

```yaml
- Effect: Allow
  Action: iot:Publish
  Resource:
    - !Sub 'arn:aws:iot:${AWS::Region}:${AWS::AccountId}:client/publisher-tpm'
```

Wait — the `iot:Publish` permission on the topic `draws/quickdraw` is granted to `UnauthRole`, which is assumed by **all** unauthenticated Cognito identities. The identity pool is public (unauthenticated access is enabled with no additional conditions). Any person on the internet can call `GetId` + `GetCredentialsForIdentity` on this identity pool and receive temporary AWS credentials that have `iot:Publish` permission on `draws/quickdraw`.

An attacker can publish fabricated lottery draw results to all 5,000 terminals without any account or authentication. This is a direct regulatory and integrity violation.

**Fix for POC:** Remove `iot:Publish` from `UnauthRole`. Create a separate authenticated role or use a server-side publisher with long-term credentials or an IoT certificate. The terminal subscribers need `iot:Connect`, `iot:Subscribe`, and `iot:Receive` only. The publisher must be a different, separately credentialed principal.

---

### BUG-04 · template.yaml: IoTLoggingRole has Resource: '*' for CloudWatch Logs

**File:** `infra/template.yaml`

```yaml
- Effect: Allow
  Action:
    - logs:CreateLogGroup
    - logs:CreateLogStream
    - logs:PutLogEvents
    - logs:DescribeLogStreams
    - logs:DescribeLogGroups
  Resource: '*'
```

The IoT logging role can write to any log group in the account. In the event of an IoT service compromise or misconfiguration, this role can overwrite or corrupt application logs, security audit logs, or CloudTrail-associated log streams across the account.

**Fix:** Scope `Resource` to the specific IoT log group ARN: `arn:aws:logs:${AWS::Region}:${AWS::AccountId}:log-group:AWSIotLogsV2:*`

---

## P2 — Runtime Bugs That Cause Silent Failures

### BUG-05 · subscriber.py: Unhandled JSONDecodeError in message callback causes silent message loss

**File:** `scripts/subscriber.py`, line in `on_message_received`

```python
def on_message_received(topic, payload, dup, qos, retain, **kwargs):
    message = json.loads(payload)
```

The MQTT callback runs on a background SDK thread. An unhandled exception thrown here is swallowed by the SDK's callback dispatcher — it does not propagate, does not crash the process, and produces no log output unless the SDK happens to print it. A malformed payload (corrupted packet, encoding issue, broker test message) causes the draw result to be silently discarded. The terminal prints nothing, the operator sees nothing, the draw is missed.

**Fix:**
```python
def on_message_received(topic, payload, dup, qos, retain, **kwargs):
    try:
        message = json.loads(payload)
    except (json.JSONDecodeError, UnicodeDecodeError) as e:
        print(f"[ERROR] Failed to decode payload on {topic}: {e} | raw={payload!r}")
        return
    ts = datetime.now(timezone.utc).isoformat()
    print(f"[{ts}] {topic}: {json.dumps(message, indent=2)}")
```

---

### BUG-06 · subscriber.py: SUBACK QoS not validated — broker can silently downgrade to QoS 0

**File:** `scripts/subscriber.py`

```python
subscribe_result = subscribe_future.result()
print(f"Subscribed to '{config.TOPIC}' with QoS {subscribe_result['qos']}")
```

The code prints the granted QoS but does not assert it matches the requested QoS. AWS IoT Core can return QoS 128 (failure) in the SUBACK. If the IAM policy does not permit `iot:Subscribe` for this identity at the moment of subscription (e.g., policy propagation delay documented as 6–8 minutes in the README), the broker returns a failure SUBACK. The subscriber prints the failure QoS and continues, believing it is subscribed. It will never receive any draw results.

This is not theoretical — the README explicitly documents a 6–8 minute IAM policy propagation delay. During this window, terminals that start up will silently be in a non-subscribed state.

**Fix:**
```python
subscribe_result = subscribe_future.result()
granted_qos = subscribe_result['qos']
if granted_qos == mqtt.QoS.AT_LEAST_ONCE:
    print(f"Subscribed to '{config.TOPIC}' with QoS {granted_qos}")
else:
    raise RuntimeError(f"Subscription to '{config.TOPIC}' failed or downgraded: granted QoS={granted_qos}")
```

---

### BUG-07 · subscriber.py: connect_future.result() returns a dict, not a bool

**File:** `scripts/subscriber.py`

```python
session_present = connect_future.result()
print(f"Connected as '{args.client_id}', session_present={session_present}")
```

`mqtt_connection.connect()` resolves to a dict `{'session_present': bool}`, not a bare bool. `session_present` is assigned the entire dict. The print statement outputs `session_present={'session_present': True}`. This affects every connect call in the file (initial connect, reconnect_test reconnect, refresh_test reconnect). The value is never used in a conditional, so it does not cause incorrect behavior — but it means every `session_present` display in the findings logs is wrong, and any future code that checks `if session_present:` will always evaluate True (non-empty dict is truthy regardless of actual session state).

**Fix:**
```python
connect_result = connect_future.result()
session_present = connect_result.get('session_present', False)
```

---

## P3 — Operational / Architecture Gaps

### BUG-08 · query_logs.py: User-controlled input injected into CloudWatch Logs Insights query

**File:** `scripts/query_logs.py`

```python
query = (
    "fields @timestamp, eventType, clientId, traceId, topicName, status\n"
    f"| filter traceId = '{args.trace_id}'\n"
    "| sort @timestamp asc"
)
```

`args.trace_id` is injected directly into the query string. CloudWatch Logs Insights is not SQL but it does have its own query language. A value like `x' | fields @message | filter @message like 'password` would escape the traceId filter and return arbitrary log entries. For a developer tool accessing sensitive IoT broker logs, this is a meaningful exposure.

**Fix:** Validate `args.trace_id` against the known traceId format (alphanumeric + hyphens) before interpolation:
```python
import re
if not re.fullmatch(r'[a-zA-Z0-9\-]+', args.trace_id):
    raise ValueError(f"Invalid trace_id format: {args.trace_id!r}")
```

---

### BUG-09 · query_logs.py: Polling loop has no timeout — hangs indefinitely on stuck query

**File:** `scripts/query_logs.py`

```python
while True:
    time.sleep(1)
    result = client.get_query_results(queryId=query_id)
    if result['status'] in ('Complete', 'Failed', 'Cancelled', 'Timeout', 'Unknown'):
        break
```

If the query status never transitions to a terminal state (AWS service degradation, API inconsistency), this loop runs forever. CloudWatch Logs Insights queries have a 15-minute server-side timeout, so the practical maximum hang is 15 minutes — but the operator has no feedback and no way to abort gracefully.

**Fix:** Add a wall-clock timeout (e.g., 120 seconds) with a clear error message.

---

### BUG-10 · template.yaml: IoT logging level is DEBUG — prohibitive cost at 5,000 terminals

**File:** `infra/template.yaml`

```yaml
IoTLogging:
  Type: AWS::IoT::Logging
  Properties:
    DefaultLogLevel: DEBUG
```

DEBUG level logs every MQTT packet event. At 5,000 terminals × 4-minute draw intervals, each draw cycle generates at minimum: 5,000 Connect events + 5,000 Subscribe events + 5,000 Publish-Out events + 5,000 PUBACK events = 20,000 log events per draw. At DEBUG, add session lifecycle and keep-alive events. CloudWatch Logs ingestion is billed per GB. For production, this is a cost sink. For the POC, it is fine — but this cannot be carried forward.

**Fix:** Use `ERROR` or `WARN` in production. Use `INFO` when debugging delivery issues. Parameterize `DefaultLogLevel` in the CloudFormation template.

---

### BUG-11 · template.yaml: No CloudWatch log retention policy

**File:** `infra/template.yaml`

The `AWSIotLogsV2` log group is created automatically by IoT Core (not by this template) and inherits the default retention of **never expire**. Logs accumulate indefinitely. At DEBUG level for a 5,000-terminal fleet, storage costs become material within weeks.

**Fix:** Add an explicit `AWS::Logs::LogGroup` resource for `AWSIotLogsV2` with a `RetentionInDays` property (e.g., 30 or 90 days).

---

### BUG-12 · publisher.py: draw_id has second-precision collision window; no exception handling on publish failure

**File:** `scripts/publisher.py`

```python
'draw_id': f'draw-{int(time.time())}',
```

Two publishes in the same second produce identical `draw_id` values. At 4-minute intervals this is not a practical risk, but draw_id is the proposed payload-level correlation key (per the FINDINGS.md conclusion). If any downstream deduplication or audit logic uses `draw_id` as a unique key, a collision silently merges two draws.

Additionally:
```python
publish_future.result()
```
If this raises (connection dropped between connect and publish), the exception propagates out of `main()` and `mqtt_connection.disconnect()` is never called. The SDK holds the connection open until GC.

**Fix for draw_id:** Use `uuid.uuid4()` or a monotonic sequence. **Fix for disconnect:** Wrap the loop in `try/finally` calling `mqtt_connection.disconnect().result()`.

---

### BUG-13 · subscriber.py: freeze_after_subscribe uses SIGSTOP — does not work on Windows

**File:** `scripts/subscriber.py`

```python
os.kill(os.getpid(), signal.SIGSTOP)
```

`SIGSTOP` is POSIX-only. On Windows this raises `AttributeError: module 'signal' has no attribute 'SIGSTOP'`. If any developer attempts to run this test mode on Windows, the subscriber crashes immediately after subscribing. Since the project specifies Python 3.11 on Windows (based on the environment context), this is a real developer-facing failure.

**Fix:** Guard with `if hasattr(signal, 'SIGSTOP'):` and print a "not supported on Windows" message otherwise.

---

### BUG-14 · Architecture: Cognito unauthenticated identities are wrong for production terminals

**File:** Architecture (infra/template.yaml, scripts/auth.py)

Cognito unauthenticated credentials expire every ~1 hour and require refresh via `GetCredentialsForIdentity`. For 5,000 production terminals running 24/7, every terminal must implement credential refresh logic, handle the reconnect window, and ensure no draws are missed during the refresh. The README acknowledges this but the subscriber code does not implement it.

Production IoT terminal authentication uses X.509 device certificates provisioned via AWS IoT Core fleet provisioning. Certificates do not expire on a 1-hour cycle, support mutual TLS (no credentials to refresh), and are revocable per-device. The Cognito path is appropriate only for browser/mobile clients.

This is not a bug in the POC — it is intentional scaffolding. But the Go decision must include an explicit architecture change item: "Replace Cognito unauthenticated identity with X.509 per-terminal certificates before production."

---

### BUG-15 · Architecture: No duplicate draw deduplication in subscriber

**File:** `scripts/subscriber.py`

QoS 1 guarantees at-least-once delivery. The subscriber's `on_message_received` has no deduplication. On reconnect, already-processed draws in the persistent session queue will be redelivered. The payload contains `draw_id` which is suitable for deduplication, but no dedup logic exists. For a lottery terminal, processing the same draw result twice could produce duplicate display output or double-post to a backend. FINDINGS.md documents that duplicate delivery was observed (finding #6).

---

## Summary Table

| ID | File | Severity | Description |
|----|------|----------|-------------|
| BUG-01 | FINDINGS.md | **P0** | LOGS-03 marked verified but finding proves it's false — corrupts architecture decision |
| BUG-02 | FINDINGS.md / subscriber.py | **P0** | CRED-01 not formally tested — terminals go silent at T+1hr in production |
| BUG-03 | template.yaml | **P1** | Public internet can publish fabricated draw results (unauthenticated + iot:Publish) |
| BUG-04 | template.yaml | **P1** | IoT logging role has Resource:* — can corrupt any log group in account |
| BUG-05 | subscriber.py | **P1** | JSONDecodeError in message callback silently drops draw messages |
| BUG-06 | subscriber.py | **P1** | SUBACK QoS not validated — terminal silently non-subscribed during IAM propagation window |
| BUG-07 | subscriber.py | **P2** | connect_future.result() returns dict not bool — session_present logs and future bool checks wrong |
| BUG-08 | query_logs.py | **P2** | trace_id injected into Logs Insights query without sanitization |
| BUG-09 | query_logs.py | **P2** | Query polling loop has no timeout — hangs indefinitely |
| BUG-10 | template.yaml | **P2** | DEBUG log level — prohibitive cost in production at 5,000 terminals |
| BUG-11 | template.yaml | **P2** | No log retention policy — unbounded log accumulation |
| BUG-12 | publisher.py | **P3** | draw_id collision at second boundary; no disconnect on publish exception |
| BUG-13 | subscriber.py | **P3** | SIGSTOP crashes on Windows |
| BUG-14 | Architecture | **P3** | Cognito unauthenticated identities wrong for production — must document as architecture change item |
| BUG-15 | subscriber.py | **P3** | No duplicate draw deduplication despite observed duplicate delivery in findings |

---

## One-Line Verdict

The POC code is acceptable for a POC. The **findings document is not** — BUG-01 is a falsified requirement that will cause the wrong architecture to be built, and BUG-02 is an untested critical path marked complete. Fix those two before any Go/No-Go decision is made from this output. BUG-03 must be fixed before this stack is deployed to any account that touches real draw data.
