# Sonnet 4.6 — Warm Onboarding Prompt (233s)

Now I have everything I need. This was a thorough read — let me give you an honest, detailed review.

---

# Code Review: AWS IoT Core Delivery Validation POC

First, a genuine compliment before the critique: **the experiment design is genuinely good.** The iptables approach to blocking PUBACK (versus the earlier SIGSTOP attempt, which you correctly recognized and documented as flawed) is precise and scientifically valid. The FINDINGS.md is one of the better-written POC reports I've seen — the evidence-to-conclusion chain is clear and the uncertainty is never hidden. The person who built this knows what they're doing. That makes honest feedback worth giving.

---

## 1. Issues Worth Addressing

### 🔴 Critical — Security

---

**[SEC-01] The publisher shares the same unauthenticated identity pool as terminals — any terminal can forge draw results**

In `template.yaml`, the single `UnauthRole` grants both `iot:Publish` on `draws/quickdraw` *and* `iot:Subscribe`/`iot:Receive` to all unauthenticated Cognito identities. This means:

- Any terminal (or anyone who can hit the Cognito endpoint) can publish fake draw results to all 5,000 connected devices
- There is no trust boundary between "the system that produces lottery results" and "the terminals that receive them"

In a lottery system, this is not a technicality. A malicious or compromised terminal publishing a fake draw result that reaches all other terminals is a serious regulatory exposure.

---

**[SEC-02] `iot:Publish` on `draws/quickdraw` is granted to all unauthenticated identities**

Remove `iot:Publish` from the unauthenticated role entirely. If ACK topics are implemented, scope the publish permission tightly:

```yaml
- Effect: Allow
  Action: iot:Publish
  Resource:
    - !Sub 'arn:aws:iot:${AWS::Region}:${AWS::AccountId}:topic/draws/quickdraw/ack/${cognito-identity.amazonaws.com:sub}'
```

That condition variable (`cognito-identity.amazonaws.com:sub`) pins each terminal to publishing *only* to its own ACK topic.

---

**[SEC-03] Any unauthenticated identity can connect as `publisher-tpm`**

The IAM policy allows `iot:Connect` for both `client/terminal-*` and `client/publisher-tpm`. Combined with SEC-01, this is the complete attack path: get credentials → connect as `publisher-tpm` → publish a crafted draw result → all 5,000 terminals receive it.

---

### 🟠 Significant — Architecture

---

**[ARCH-01] The scale gap between POC and production is not bridged — the "Go" conclusion is premature**

The experiments ran with **2 subscribers**. The production target is **5,000**. That's a 2,500× gap. Key unknowns:

- **IoT Core fan-out latency at scale**
- **CloudWatch Logs write throughput:** ~10,000 events per draw cycle approaches CloudWatch rate limits
- **Cognito `GetId` throttling at reconnection storms**
- **IoT Core message broker transactions per second** (one publish to 5,000 subscribers = 5,000 delivery operations)

The "conditional" part of "Conditional Go" needs explicit conditions, not just a list of next steps.

---

**[ARCH-02] QoS 1 at-least-once delivery is not handled at the terminal — double-delivery is a real risk**

Experiment 2 explicitly documents that the frozen terminal received the **same message twice**. The subscriber code has no deduplication logic. The `draw_id` field is already in the payload:

```python
seen_draw_ids = set()

def on_message_received(topic, payload, dup, qos, retain, **kwargs):
    message = json.loads(payload)
    draw_id = message.get('draw_id')
    if draw_id in seen_draw_ids:
        print(f"[DUPLICATE] Ignoring already-processed draw_id={draw_id}")
        return
    seen_draw_ids.add(draw_id)
```

---

**[ARCH-03] DEBUG-level logging at account scope will be operationally unmanageable at production scale**

At 5,000 terminals × every 4-minute draw cycle: Connect, Subscribe, Publish-In, 5,000× Publish-Out, Disconnect events. Cost, query performance, and signal-to-noise all degrade.

---

**[ARCH-04] No CloudWatch log retention policy — logs accumulate forever**

```yaml
IoTLogGroup:
  Type: AWS::Logs::LogGroup
  Properties:
    LogGroupName: AWSIotLogsV2
    RetentionInDays: 30
```

---

**[ARCH-05] Persistent session expiry of 1 hour is too short for production terminals**

Maintenance windows, overnight shutdowns, and network outages can easily exceed 1 hour. When the session expires, queued QoS 1 messages are lost — with no notification. IoT Core supports up to 7 days.

---

### 🟡 Moderate — Code Quality

---

**[CODE-01] Unhandled exception in MQTT message callback will silently kill message processing**

`json.loads()` raises `JSONDecodeError` inside the CRT background thread. An unhandled exception may silently crash the callback.

---

**[CODE-02] `cmd_trace` ignores `--minutes` and hardcodes a 60-minute window**

---

**[CODE-03] `publisher.py` has a hardcoded client ID**

---

**[CODE-04] `connect_future.result()` result assigned to misleadingly named variable**

The awsiotsdk `connect()` future resolves to a `dict` like `{'session_present': True}`, not a bare boolean.

---

**[CODE-05] Minor log injection vector in `cmd_trace` query construction**

---

## 2. Propositions: What and How to Improve

---

### Proposition A: Separate publisher authentication immediately

**How:** IoT Core certificate for the TPM machine:
1. Provision an IoT certificate + private key for the TPM server
2. Create an IoT Policy scoped only to `iot:Connect` for `publisher-tpm` and `iot:Publish` on `draws/quickdraw`
3. Remove `iot:Publish` from the `UnauthRole` entirely
4. Change `publisher.py` to use `mqtt_connection_builder.mtls_from_path()` instead of WebSocket/SigV4

---

### Proposition B: Implement the ACK topic now, not later

1. Subscriber-side ACK publish after receiving a message
2. Scope IAM policy publish permission to `topic/draws/quickdraw/ack/${cognito-identity.amazonaws.com:sub}`
3. Add an IoT Rule that writes ACK messages to DynamoDB keyed by `(draw_id, client_id)`
4. A simple Lambda + DynamoDB query answers "which terminals received draw-X?"

This is the difference between "we think it probably worked" and "we can prove it in a regulatory audit."

---

### Proposition C: Add a scale pre-validation test

```python
# scale_test.py — spawn N subscribers as threads
# Test at 50, 500, 2000, 5000
# Record: time-to-first-message per subscriber
# Measure: p50/p95/p99 delivery latency, any missing deliveries
```

---

### Proposition D: Replace busy-wait with exponential backoff in query_logs.py

---

### Proposition E: Set log retention via CloudFormation

---

## 3. Other Ideas and Observations

---

**The traceId behavior has a regulatory implication that's understated**

IoT Core logs alone cannot constitute a delivery audit trail. If a regulatory body asks "prove that terminal-4832 received draw result 2026-04-09T16:44:45Z," the Publish-Out log is not sufficient. An ACK topic logging to a tamper-evident store (DynamoDB with point-in-time recovery, S3 with Object Lock) would constitute actual proof.

---

**The double-delivery finding needs to be surfaced as a terminal software requirement**

If the current UDP multicast system doesn't have duplicates (UDP is at-most-once), this is a **breaking behavioral change** for terminal developers. It should be explicit in the handoff documentation.

---

**The 74ms auto-reconnect bypassing the queue has an operational monitoring implication**

Absence-based monitoring ("no Queued event → terminal was online") is unreliable — a fast-reconnecting terminal won't have a Queued event. This strengthens the case for the ACK topic pattern.

---

**Consider MQTT 5 over MQTT 3.1.1**

- Correlation data in PUBLISH packets
- User properties
- Request/Response pattern for ACK flows

---

**Cost should be estimated before the production decision**

- Connection minutes: 5,000 × 24h × 60min = 7.2M/day × 30 = 216M/month
- Messages: 1 publish × 5,000 deliveries × 360 draws/day = 1.8M messages/day
- CloudWatch Logs: ~18M log events/day at DEBUG level

---

**Summary Table**

| # | Severity | Category | One-Line |
|---|----------|----------|----------|
| SEC-01 | 🔴 Critical | Security | Publisher uses same unauthenticated pool as terminals |
| SEC-02 | 🔴 Critical | Security | All terminals can publish draw results |
| SEC-03 | 🔴 Critical | Security | Any unauthenticated identity can impersonate publisher-tpm |
| ARCH-01 | 🟠 Significant | Architecture | 2 subscribers ≠ 5,000 — scale test required |
| ARCH-02 | 🟠 Significant | Architecture | No deduplication — terminals will process duplicate draws |
| ARCH-03 | 🟠 Significant | Architecture | DEBUG logging at scale will be expensive and hard to query |
| ARCH-04 | 🟡 Moderate | Architecture | No log retention policy |
| ARCH-05 | 🟡 Moderate | Architecture | 1-hour session expiry too short |
| CODE-01 | 🟡 Moderate | Code | Unhandled exception in MQTT callback |
| CODE-02 | 🟡 Moderate | Code | cmd_trace ignores --minutes |
| CODE-03 | 🟢 Minor | Code | Publisher client ID hardcoded |
| CODE-04 | 🟢 Minor | Code | session_present holds a dict, not a boolean |
| CODE-05 | 🟢 Minor | Code | Raw string interpolation in query from CLI input |

The bones of this POC are solid. The main work ahead is in the security model (before touching production) and in scale validation (before the architecture decision is locked in).
