# Opus 4.6 — Coercive Prompt

# AWS IoT Core Delivery Validation POC — Code Review

**Scope:** All source files, infrastructure, deploy script, documentation, planning artifacts
**Method:** Full read of codebase + four prior independent reviews (01–04) to calibrate and identify gaps

---

## Preamble: On the Prompt

This review was solicited with a high-pressure framing — "5,000 terminals," "regulatory consequences," "people's jobs depend on this." That framing incentivizes inflated severity and finding quantity over calibration. I'm going to resist that incentive and give you something more useful: findings sorted by whether they actually matter, with honest severity ratings, and explicit callouts where prior reviewers over-inflated or missed things.

This is a completed POC. It has already done its job — it produced clear empirical evidence on Publish-Out semantics and delivered a "Conditional Go" verdict. The code will never run in production as-is; its value is in the findings it generated. I'll review it on those terms while flagging what matters for the production system it informs.

---

## 1. Findings That Actually Matter for Production

These are the issues that, if carried forward into the production design, would cause real outages or security exposure.

### F-01 — Static Credentials Provider Breaks Auto-Reconnect After ~1 Hour 🔴

**File:** `auth.py:68`
```python
credentials_provider = AwsCredentialsProvider.new_static(
    access_key_id=credentials['access_key_id'],
    secret_access_key=credentials['secret_access_key'],
    session_token=credentials['session_token'],
)
```

Cognito temporary credentials expire after ~1 hour. `new_static()` bakes the credential snapshot into the CRT SDK at connection creation time. When a network blip triggers auto-reconnect after credential expiry, the SDK reuses the stale credentials and receives a 403. The connection silently dies — no exception, no callback, no recovery.

The POC's own experiments validate the individual pieces — Experiment 2 proves auto-reconnect works (within the credential window), and Experiment 4 proves manual credential refresh works — but the *combination* (auto-reconnect + expired credentials) is untested and broken.

**Fix:** `AwsCredentialsProvider.new_delegate()` with a callback that calls `get_cognito_credentials()` using the cached `identity_id`. This makes every reconnect fetch fresh credentials automatically.

**Credit:** Found by Opus Warm (Review 03). No other reviewer caught this. It is the single most production-critical finding across all reviews.

---

### F-02 — Any Anonymous User Can Publish Fake Draw Results 🔴

**File:** `template.yaml:62–66`

The single `UnauthRole` grants `iot:Publish` on `draws/quickdraw` to all unauthenticated Cognito identities. The attack path is trivial: call `GetId` → `GetCredentialsForIdentity` → connect as `publisher-tpm` → publish a crafted draw result → all subscribers receive it.

The Identity Pool ID is in CloudFormation outputs and `.env`. It is not a secret.

For the POC, this is a deliberate simplification. For the production system, the publisher must use a completely separate authentication mechanism — either a dedicated IAM role behind authenticated Cognito identities, or (better) an IoT certificate provisioned specifically for the TPM machine via `mqtt_connection_builder.mtls_from_path()`. Terminals should have zero publish permissions on the draw topic.

**Consensus finding:** All 4 prior reviewers identified this. It's real and important.

---

### F-03 — No Deduplication = Double-Processing of Draw Results 🟠

**File:** `subscriber.py:43–46`

The FINDINGS prove (Experiment 2) that QoS 1 delivers the same message twice in reconnect scenarios. The subscriber has no deduplication logic. The `draw_id` field is already in the payload — it just needs a `seen_draw_ids` set.

This is also a **breaking behavioral change from UDP multicast**. UDP is at-most-once — a missed packet is gone. MQTT QoS 1 is at-least-once — duplicates are guaranteed under certain conditions. Terminal developers need to know this before the migration. If the current terminal software assumes "one message = one draw," double-processing could have operational consequences.

**Thread safety note:** The `on_message_received` callback runs on the CRT background thread. Any state tracking (deduplication set, counters) must use a lock or thread-safe data structure. The current callback is pure side-effect (print), so this isn't a problem yet, but it will be as soon as deduplication or ACK logic is added.

---

### F-04 — Persistent Session Expiry Creates a Silent Message Loss Window 🟠

**File:** Not in code — architectural concern

IoT Core persistent sessions expire after 1 hour by default (documented in README line 9). If a terminal loses connectivity for >1 hour, its session expires and all queued messages are discarded. The terminal reconnects with `session_present=False`, re-subscribes, and resumes — but any draws published during the outage are permanently lost with no notification.

For 5,000 terminals with 4-minute draw cycles, a 1-hour outage means ~15 missed draws. IoT Core supports persistent session expiry up to 7 days (account-level setting via `UpdateAccountAuditConfiguration`). The production system should extend this and implement a "catch-up" mechanism — a retained message or HTTP endpoint where a terminal can request the latest draw result on reconnect.

---

### F-05 — Publish-Out Cannot Serve as a Regulatory Delivery Audit Trail 🟠

**File:** `FINDINGS.md:172–188`

The FINDINGS correctly identify that Publish-Out `status=Success` means broker dispatch, not client receipt, and that traceId doesn't link Publish-In to Publish-Out. The conclusion recommends payload-based ACK tracking.

The production implication is stronger than stated: for a lottery system, regulators may require provable per-terminal delivery confirmation for each draw. IoT Core logs alone cannot provide this. The ACK topic pattern (`draws/quickdraw/ack/{clientId}` with `draw_id`) written to a tamper-evident store (DynamoDB with point-in-time recovery, or S3 with Object Lock) is not a nice-to-have — it's a compliance requirement. This should be prototyped in the POC, not deferred to production.

---

## 2. Real Issues in the POC Code

These won't cause production outages (the POC code won't go to production), but they affect the validity of the POC's findings or the developer experience.

### F-06 — Environment Variable Name Mismatch Between deploy.sh and Python Scripts 🟡

**File:** `deploy.sh:4` vs `config.py:12`

```bash
# deploy.sh
REGION="${AWS_DEFAULT_REGION:-us-east-1}"
```
```python
# config.py
AWS_REGION = os.environ.get('AWS_REGION', 'us-east-1')
```

The deploy script reads `AWS_DEFAULT_REGION`. The Python scripts read `AWS_REGION`. The `.env.example` defines `AWS_REGION`. If someone sets `AWS_REGION=eu-west-1` in their `.env` and runs `deploy.sh`, the stack deploys to `us-east-1` (the default) while the Python scripts try to connect to `eu-west-1`. The IoT endpoint, Cognito pool, and scripts would all target different regions.

Both happen to default to `us-east-1`, so this never bit anyone during the POC. But it's a genuine mismatch that would cause confusion in a different region.

**Not found by any prior reviewer.**

---

### F-07 — `session_present` Receives a Dict, Not a Bool 🟡

**File:** `subscriber.py:60, 113, 156`

```python
session_present = connect_future.result()
```

The awsiotsdk `connect()` future resolves to `{'session_present': bool}`, not a bare boolean. The variable `session_present` holds the entire dict. The print output would be `session_present={'session_present': True}` — but the FINDINGS.md shows `session_present=True`, suggesting either the SDK behavior differs from documentation, or the findings output was cleaned up.

Either way, the fix is trivial: `result = connect_future.result(); session_present = result['session_present']`. This applies to all three connect paths and to `publisher.py:25`.

**Consensus finding:** All 4 prior reviewers identified this. The Sonnet Default review rates it 🔴 HIGH, which is inflated — it's a display bug in a POC that only affects print output. 🟡 MEDIUM.

---

### F-08 — LOGS-03 Requirement Marked Complete When the Finding Refutes It 🟡

**File:** `REQUIREMENTS.md:33`, `FINDINGS.md:172`

```markdown
- [x] **LOGS-03**: traceId correlates a single Publish-In to all corresponding Publish-Out entries
```

The FINDINGS prove this is false — each Publish-Out gets its own traceId. The requirement is checked off as "complete" in the verification table, but the finding actually **invalidates** the requirement. The checkbox should indicate that the requirement was tested and refuted, not that it was satisfied.

Similarly, **CRED-01** is marked `[x] Complete` but FINDINGS.md says "Not yet tested separately" — it was only implicitly validated via auto-reconnects that happened to occur within the credential validity window. Given that F-01 (static credentials) means the auto-reconnect path is broken after expiry, the implicit validation is weaker than it appears.

**Found by Sonnet Default (Review 02).** Real documentation integrity issue.

---

### F-09 — No Exception Handling on Future `.result()` Calls 🟡

**Files:** `publisher.py:25,48`, `subscriber.py:59,69,89,112,156,165,177`

Every `future.result()` call — connect, subscribe, publish, disconnect — is unguarded. A transient network failure raises an unhandled exception with no cleanup. The subscriber's `disconnect()` is behind a `try/except KeyboardInterrupt` but the publisher has no such protection — Ctrl+C during `input()` or `publish_future.result()` skips `disconnect()`.

For a POC that runs interactively on a developer's machine, this is 🟡 MEDIUM. For production, it's 🔴.

---

### F-10 — `cmd_trace` Ignores the `--minutes` Argument 🟡

**File:** `query_logs.py:70–71`

```python
end_time = int(time.time())
start_time = end_time - (60 * 60)  # hardcoded 60 minutes
```

All other subcommands use `args.minutes`. `cmd_trace` hardcodes a 60-minute window. The planning decision notes say "60-minute window for trace queries since traceId is already specific" — so this is intentional, but the `--minutes` flag still appears on the parent parser and silently does nothing for the `trace` command. Either remove `--minutes` from the trace subparser's inheritance or use it.

**Consensus finding:** Found by most prior reviewers.

---

### F-11 — `run_query` Polling Loop Has No Timeout 🟢

**File:** `query_logs.py:36–40`

```python
while True:
    time.sleep(1)
    result = client.get_query_results(queryId=query_id)
    if result['status'] in ('Complete', 'Failed', 'Cancelled', 'Timeout', 'Unknown'):
        break
```

If the CloudWatch API returns an unexpected status string (or the query is Running forever), this loops indefinitely. A `for _ in range(120)` with an `else: raise` would cap it at 2 minutes.

---

### F-12 — `on_message_received` Has No Error Handling 🟢

**File:** `subscriber.py:43–44`

```python
def on_message_received(topic, payload, dup, qos, retain, **kwargs):
    message = json.loads(payload)
```

`json.loads()` on the CRT background thread. If a non-JSON payload arrives, the exception behavior depends on the CRT SDK's error handling — it may silently swallow the exception and continue, or it may kill the callback thread. A `try/except` is cheap insurance.

For this POC, the only publisher is the POC's own `publisher.py` which always sends valid JSON, so this is 🟢 LOW. The Sonnet Default review rates it 🔴 HIGH — that's inflated.

---

### F-13 — `dup` Flag Received But Ignored 🟢

**File:** `subscriber.py:43`

```python
def on_message_received(topic, payload, dup, qos, retain, **kwargs):
```

The MQTT `dup` flag indicates whether this message is a redelivery. The callback receives it but doesn't log or act on it. For the POC's own experiments — where duplicate delivery was explicitly studied (Experiment 2) — logging `dup=True` would have provided additional evidence directly in the subscriber output rather than requiring cross-referencing with CloudWatch logs.

**Not found by any prior reviewer.**

---

### F-14 — Unused `json` Import in `query_logs.py` 🟢

**File:** `query_logs.py:4`

```python
import json
```

`json` is imported but never used anywhere in `query_logs.py`. All data handling uses string formatting.

**Not found by any prior reviewer.**

---

### F-15 — `draw_id` Uses Second-Precision Timestamp 🟢

**File:** `publisher.py:36`

```python
'draw_id': f'draw-{int(time.time())}',
```

Two publishes within the same second get the same `draw_id`. Since the POC is interactive (human pressing Y), this is nearly impossible in practice. For the production system where `draw_id` is the recommended correlation key, it needs to be globally unique (UUID or a sequence from the draw management system).

---

## 3. Calibration Notes on Prior Reviews

### Things Prior Reviewers Got Right

- **Static credentials (F-01):** Only Opus Warm found this. It's the most impactful finding. The warm onboarding prompt appears to have enabled deeper temporal reasoning (thinking about what happens at hour 2, not just minute 1).
- **IAM role separation (F-02):** All 4 found this. Consensus is reliable.
- **LOGS-03/CRED-01 documentation inconsistency (F-08):** Only Sonnet Default found this. Good catch — documentation integrity matters for a POC whose entire value is its documented findings.
- **Persistent session expiry (F-04):** Only Sonnet Warm flagged the 1-hour expiry as a production risk with concrete scenario (terminal offline 90 minutes → messages lost). Correct and important.

### Things Prior Reviewers Over-Inflated

- **"Query injection" in `cmd_trace` (rated 🔴 HIGH by Opus Default, 🟡 MED by others):** CloudWatch Logs Insights is not SQL. The injection surface is negligible — you can modify the query filter, but you can't exfiltrate data, execute commands, or escalate privileges. The attacker also already has CLI access to the machine (since it's a local script), which means they already have the AWS credentials. This is 🟢 LOW at most.

- **Hardcoded publisher `client_id` (rated 🔴 HIGH by Sonnet Default):** The publisher is a single interactive script for POC testing. A hardcoded client ID matching the IAM policy is appropriate. This is 🟢 LOW.

- **`json.loads()` exception in CRT thread (rated 🔴 HIGH by Sonnet Default):** The only data source is the POC's own publisher, which always sends valid JSON. For this to trigger, the security issue (F-02) would need to be exploited first. 🟢 LOW in context.

- **`session_present` dict vs bool (rated 🔴 HIGH by Sonnet Default):** Only affects print output in a POC. 🟡 MEDIUM.

- **DEBUG logging "captures full message payloads" (rated 🔴 HIGH by Sonnet Default):** IoT Core DEBUG logging captures metadata (clientId, traceId, topicName, eventType, status). It does **not** capture message payloads unless you configure IoT Rules to route messages to CloudWatch. The Sonnet Default review's claim is factually incorrect for the default `AWSIotLogsV2` log group. 🟢 LOW — the real cost concern is log volume at scale, not payload exposure.

### Things All Prior Reviewers Missed

- **F-06 — Environment variable name mismatch** (`AWS_DEFAULT_REGION` vs `AWS_REGION`): None of the 4 reviews caught this. It's a concrete bug that would manifest in any non-us-east-1 deployment.
- **F-13 — `dup` flag ignored:** The duplicate delivery finding is central to the POC's conclusions, yet the callback doesn't log the one piece of client-side evidence the SDK provides for free.
- **F-14 — Unused `json` import:** Minor but real.
- **Thread safety of the message callback for future deduplication work:** Mentioned in passing by none. The callback runs on the CRT thread; any mutable state added for deduplication or ACK publishing needs synchronization.

---

## 4. Infrastructure Review

### CloudFormation Template (`template.yaml`)

| Finding | Severity | Notes |
|---------|----------|-------|
| Single IAM role for pub + sub | 🔴 | F-02. Must separate for production. |
| `IoTLoggingRole` uses `Resource: '*'` | 🟡 | Should scope to `arn:aws:logs:${Region}:${AccountId}:log-group:AWSIotLogsV2:*` |
| No `AWS::Logs::LogGroup` with retention | 🟡 | Logs accumulate indefinitely. Add `RetentionInDays: 7` for POC, 30 for production. |
| Named IAM roles (`RoleName: !Sub ...`) | 🟢 | Prevents stack updates that require replacement. Remove `RoleName` or append region. |
| DEBUG log level at account scope | 🟢 | Appropriate for POC. Production should use INFO or WARN. Cost at scale: 5,000 terminals × events/cycle × 24h × $0.50/GB ingestion. |

### Deploy Script (`deploy.sh`)

| Finding | Severity | Notes |
|---------|----------|-------|
| `AWS_DEFAULT_REGION` vs `AWS_REGION` mismatch | 🟡 | F-06. Should use `AWS_REGION` to match `.env.example`. |
| No `--no-fail-on-empty-changeset` | 🟢 | Re-running on unchanged stack fails with `set -euo pipefail`. |
| No prerequisite check | 🟢 | AWS CLI and credentials assumed present. |

---

## 5. Scale Gap Assessment

The POC tested with 2 subscribers. Production targets 5,000. The "Conditional Go" conclusion is appropriately hedged but the conditions should be explicit:

| Dimension | POC | Production | Risk |
|-----------|-----|------------|------|
| Subscribers | 2 | 5,000 | Fan-out latency unknown at 2,500× scale |
| Concurrent connections | 3 | 5,001+ | IoT Core limit is 500K — fine |
| Messages/draw | 1 publish → 2 deliveries | 1 publish → 5,000 deliveries | IoT Core TPS limit is relevant |
| Keepalive traffic | Negligible | ~167 pings/sec (30s interval) | IoT Core handles this; network may not |
| Cognito `GetId` calls | 2 | 5,000 on cold start | Cognito default rate limit is 5 TPS → takes 17 minutes to provision |
| CloudWatch events/draw | ~6 | ~10,000+ | Ingestion cost + query performance degrade |
| Credential refresh storm | N/A | 5,000 simultaneous refreshes at ~T+60min | Cognito throttling risk |

The most dangerous unknowns are **Cognito rate limits** (cold-start provisioning and hourly credential refresh storms) and **fan-out latency** (does IoT Core deliver to subscriber #5,000 within the 4-minute draw window?). These should be tested at 500 terminals before committing to the architecture.

---

## 6. Cost Estimate (Production Scale)

| Component | Monthly Estimate |
|-----------|-----------------|
| IoT Core connectivity | 5,000 terminals × 730h × $0.08/million min = ~$17.50 |
| IoT Core messaging | 360 draws/day × 5,001 msgs × 30 days × $1.00/million = ~$54 |
| CloudWatch Logs (DEBUG) | ~18M events/day × 30 days × ~0.5KB avg = ~270GB × $0.50/GB = **~$135/mo** |
| CloudWatch Logs (INFO) | ~90% reduction → **~$13.50/mo** |
| Cognito | Free tier covers 50K MAU |
| **Total (DEBUG)** | **~$207/mo** |
| **Total (INFO)** | **~$85/mo** |

CloudWatch logging at DEBUG is the largest cost component and provides the least value at scale (since Publish-Out can't serve as a delivery audit trail anyway). Dropping to INFO and relying on the ACK topic for delivery tracking is both cheaper and more reliable.

---

## 7. Summary Table

| # | Severity | Category | Finding | New? |
|---|----------|----------|---------|------|
| F-01 | 🔴 Critical | Architecture | Static credentials break auto-reconnect after ~1hr | No (Opus Warm) |
| F-02 | 🔴 Critical | Security | Any anonymous user can publish fake draw results | No (all 4) |
| F-03 | 🟠 Significant | Architecture | No deduplication + breaking change from UDP semantics | No (Opus/Sonnet Warm) |
| F-04 | 🟠 Significant | Architecture | 1-hour session expiry = silent message loss | No (Sonnet Warm) |
| F-05 | 🟠 Significant | Architecture | Publish-Out cannot serve as regulatory audit trail | No (Sonnet Warm) |
| F-06 | 🟡 Moderate | Bug | deploy.sh `AWS_DEFAULT_REGION` ≠ config.py `AWS_REGION` | **Yes** |
| F-07 | 🟡 Moderate | Bug | `session_present` receives dict, not bool | No (all 4) |
| F-08 | 🟡 Moderate | Documentation | LOGS-03 marked complete when finding refutes it; CRED-01 only implicit | No (Sonnet Default) |
| F-09 | 🟡 Moderate | Robustness | No exception handling on any `.result()` call | No (all 4) |
| F-10 | 🟡 Moderate | Bug | `cmd_trace` ignores `--minutes` | No (most reviewers) |
| F-11 | 🟢 Minor | Robustness | `run_query` polling has no timeout | No (Sonnet Default) |
| F-12 | 🟢 Minor | Robustness | `on_message_received` has no error handling | No (Sonnet Default) |
| F-13 | 🟢 Minor | Observability | MQTT `dup` flag received but not logged | **Yes** |
| F-14 | 🟢 Minor | Code quality | Unused `json` import in `query_logs.py` | **Yes** |
| F-15 | 🟢 Minor | Correctness | `draw_id` second-precision, not unique within 1s | No (Sonnet Default) |

**New findings in this review:** 3 (F-06, F-13, F-14)
**Over-inflated findings corrected:** 5 (query injection, hardcoded client_id, json.loads exception, session_present severity, DEBUG payload exposure claim)

---

## 8. Verdict

The POC accomplished what it set out to do. The experimental methodology is rigorous, the findings are honest, and the "Conditional Go" conclusion is appropriately hedged. The code quality is good for a throwaway validation prototype.

**For the production system it informs, three things must happen before any terminal traffic touches IoT Core:**

1. **Separate publisher auth** (F-02) — certificate-based, not Cognito
2. **Delegate credentials provider** (F-01) — or switch terminals to certificates too
3. **ACK topic with tamper-evident storage** (F-05) — the only approach that satisfies regulatory delivery confirmation

**And two things must be validated before committing the architecture:**

4. **Scale test at 500 terminals** — validates fan-out latency, Cognito rate limits, and CloudWatch costs
5. **Extended runtime test (>2 hours)** — validates credential refresh under auto-reconnect (F-01)

Everything else is engineering work that can be addressed incrementally.
