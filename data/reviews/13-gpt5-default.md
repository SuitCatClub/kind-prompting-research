# AWS IoT Core POC Review

## Executive summary
This POC demonstrates that AWS IoT Core can fan out MQTT messages and that CloudWatch logs are useful for broker-side observability. However, it is **not production-ready** in its current form. The biggest blockers are **unauthenticated access**, **overly broad IoT permissions**, **no trustworthy end-to-end delivery acknowledgement**, and **missing operational controls** for 5,000 terminals.

**Overall assessment:**
- **Security:** High risk
- **Scalability:** Promising for the stated load, but unproven operationally
- **Production readiness:** Low

## What is good
- Uses managed MQTT infrastructure instead of router-stressing multicast.
- Uses QoS 1 and exercises reconnect / persistent-session behavior.
- Captures an important finding: IoT Core `Publish-Out` logs are **not** client PUBACK confirmation.
- Keeps the publish payload small, which is good for large fan-out.
- Infrastructure is simple enough for fast experimentation.

## Highest-priority findings

| Severity | Area | Finding | Why it matters |
|---|---|---|---|
| Critical | Security | `AllowUnauthenticatedIdentities: true` with broad IoT permissions | Anyone who can reach Cognito can obtain AWS credentials and connect to IoT Core. |
| Critical | Security | Unauthenticated role can `iot:Publish` to `draws/quickdraw` | Any attacker can inject fake lottery draws. |
| Critical | Security | Unauthenticated role can connect as `publisher-tpm` or any `terminal-*` client ID | Client impersonation and session hijacking are possible. |
| High | Security | No per-device identity or authorization boundary | You cannot isolate, revoke, or audit a single terminal cleanly. |
| High | Reliability | No application-level ACK path | Broker dispatch is not proof that the terminal processed the draw. |
| High | Operations | IoT logging is set to `DEBUG` account-wide | High cost, noisy logs, and possible sensitive metadata exposure in production. |
| Medium | Scalability | No explicit strategy for offline terminals / queued-session buildup | Persistent sessions can accumulate queued QoS 1 traffic and hit limits/cost. |
| Medium | Robustness | Long-lived MQTT connections use static credentials only | Real terminals need automatic credential refresh before expiry. |

## Security review

### 1) Identity model is not acceptable for production
The CloudFormation template creates a Cognito identity pool with unauthenticated identities enabled:
- `AllowUnauthenticatedIdentities: true`
- an unauthenticated IAM role that can connect, subscribe, receive, and publish

That is fine for a lab POC, but not for a lottery distribution system. In production, every terminal needs a strong identity and least-privilege authorization.

### 2) IoT policy scope is too broad
The unauth role allows:
- `iot:Connect` to `client/terminal-*` and `client/publisher-tpm`
- `iot:Publish` to the live draw topic
- `iot:Subscribe` / `iot:Receive` on the live draw topic

Consequences:
- A malicious client can publish fraudulent results.
- A client can impersonate another terminal by reusing its client ID.
- A client can knock a real device offline by causing duplicate-client-ID disconnects.

### 3) No message authenticity beyond transport auth
Even if transport access were fixed, terminals currently trust any valid publisher on the topic. There is no payload signature, sequence number validation, replay protection, or TTL enforcement.

For lottery outcomes, terminals should validate at least:
- `draw_id`
- monotonic sequence / event number
- issuance timestamp + expiry window
- cryptographic signature from the authoritative backend

### 4) Observability configuration is unsafe for steady-state production
`AWS::IoT::Logging` is set to `DEBUG`. That is useful during testing, but too noisy and expensive for production. It should be reduced to `ERROR` or `WARN`, with targeted debug windows when investigating incidents.

## Scalability review

### 1) AWS IoT Core is a plausible fit for 5,000 terminals
A single small QoS 1 message every ~4 minutes to ~5,000 terminals is modest for AWS IoT Core. The fan-out profile itself is not the concern.

The real scaling questions are operational:
- reconnect storms after network or power events
- duplicate sessions / client-ID conflicts
- large numbers of intermittently offline terminals
- confirmation traffic if every terminal ACKs every draw

### 2) Delivery confirmation design is incomplete
The repo correctly concludes that CloudWatch `Publish-Out` success is only broker dispatch. If the business requires proof that each terminal received and processed a draw, you need an **application-level ACK design**.

Recommended pattern:
- Publish draw result on a shared topic such as `draws/quickdraw/v1`
- Each terminal publishes an ACK to `draws/acks/{terminalId}`
- ACK includes `draw_id`, terminal ID, software version, receive timestamp, and validation status
- Backend aggregates ACKs and alerts on missing terminals after a deadline

At 5,000 terminals every 4 minutes, the ACK volume is still manageable.

### 3) Topic design is too POC-oriented
A single hard-coded topic is fine for a demo, but production should at least version and namespace topics, e.g.:
- `lottery/quickdraw/results/v1`
- `lottery/quickdraw/acks/{terminalId}`
- `lottery/quickdraw/control/{terminalId}`

This makes future migrations and authorization boundaries much easier.

### 4) Persistent session behavior needs policy and limits
The subscriber defaults to `clean_session=False`, which is good for testing queued QoS 1 delivery, but production needs explicit rules for:
- how long sessions persist
- what happens when a terminal is offline for hours
- whether stale draws should be discarded instead of delivered
- maximum queued messages per terminal

For draw results, stale delivery can be worse than no delivery unless terminals check message freshness.

## Code / implementation review

### `scripts/auth.py`
- Uses static temporary credentials with the websocket signer, but there is no automatic refresh path for long-lived clients.
- Uses Cognito identity-pool credentials, but the surrounding infrastructure currently allows unauthenticated terminal access.
- No retry / backoff / timeout handling around Cognito calls.

### `scripts/publisher.py`
- Hard-coded `client_id='publisher-tpm'` is unsafe in shared environments.
- Interactive publisher is fine for testing only.
- Payload lacks schema version, signature, TTL, and ordering metadata.

### `scripts/subscriber.py`
- Blind `json.loads(payload)` with no exception handling; malformed payloads can break processing.
- `--client-id` is fully caller-controlled, reinforcing impersonation risk.
- Manual reconnect / refresh tests are useful for the POC, but not a production terminal implementation.
- `SIGSTOP` logic is Unix-specific and not portable.

### `scripts/query_logs.py`
- Useful investigation utility, but it polls once per second without backoff or timeout control.
- Logs Insights is not a production delivery-tracking system.
- The `raw` command allows arbitrary queries; acceptable for operator tooling, but should be access-controlled.

### `infra/template.yaml`
- Biggest risk surface in the repository.
- Missing separate roles/policies for publisher vs subscriber.
- Missing per-device principal mapping, device registry, cert provisioning, alarms, dashboards, and quotas.
- Logging IAM policy is wildcarded and broader than necessary.

## Production-readiness gaps
To move this toward production, the next iteration should include:

1. **Replace unauthenticated Cognito access**
   - Prefer X.509 device certificates with AWS IoT Thing policies, or authenticated Cognito identities with strict per-device policy variables.
   - Do not let field terminals publish to the results topic.

2. **Separate trust domains**
   - Backend publisher uses a server-side IAM role.
   - Terminals only subscribe to results and publish ACKs on their own terminal-scoped topic.

3. **Add payload-level integrity**
   - Signed messages, sequence numbers, issued-at timestamp, expiry, and schema version.

4. **Add real delivery tracking**
   - Application ACK aggregation in DynamoDB / Kinesis / SQS / Lambda, not log scraping.

5. **Harden terminal behavior**
   - Automatic credential refresh
   - Exponential backoff with jitter
   - Idempotent draw processing
   - Stale-message rejection
   - Local buffering / health reporting

6. **Reduce logging blast radius**
   - Default IoT logs to WARN/ERROR
   - Emit explicit application metrics and alarms
   - Add dashboards for connected clients, publish failures, ack lag, and missing terminals

7. **Run scale and failure testing**
   - 5,000+ concurrent terminals
   - reconnect storm test
   - partial regional/network impairment
   - duplicate client-ID scenarios
   - offline backlog / stale-draw behavior

## Bottom line
**Recommendation: do not promote this implementation as-is.**

The architecture direction (AWS IoT Core for fan-out) is reasonable, and the POC already surfaced an important truth about broker logs vs actual receipt. But the current implementation should be treated as a **prototype only**. Before production, the system needs a proper device identity model, least-privilege authorization, application-level acknowledgements, payload integrity controls, and operational instrumentation.


