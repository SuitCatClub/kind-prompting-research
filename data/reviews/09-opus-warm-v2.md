# Opus 4.6 — Warm v2 (Depth + Breadth)

I spent real time with this. The experimental methodology is first-rate — the iptables trick to isolate PUBACK behavior from connection liveness is the kind of thing you only design if you actually understand the TCP/MQTT layer boundary. The progression from SIGSTOP (too coarse) to iptables (surgically precise) is documented honestly, dead ends included. That intellectual honesty is what makes the findings trustworthy.

This review is structured as requested: first pass deep, second pass sweep, then devil's advocate, then the full catalog with severity separated from the finding.

---

## First Pass — Deep Dive

### 1. The Static Credentials Time Bomb

This is the most important finding and no amount of emphasis is too much.

**File:** `auth.py` lines 68–72, `subscriber.py` entire normal-mode flow

```python
credentials_provider = AwsCredentialsProvider.new_static(
    access_key_id=credentials['access_key_id'],
    secret_access_key=credentials['secret_access_key'],
    session_token=credentials['session_token'],
)
```

Cognito unauthenticated credentials expire after ~1 hour. `new_static()` bakes them into the MQTT connection at creation time. The CRT SDK's auto-reconnect mechanism (which you observed working beautifully in Experiment 2 — the 74ms reconnect) reuses whatever credentials were provided at connection creation. After credential expiry, the next auto-reconnect attempt uses stale credentials → SigV4 signature fails → 403 → connection silently dies.

Why this matters so much: every experiment ran within a single credential lifetime. The POC never exercised the hour-boundary failure mode because tests are short. In production, 5,000 terminals silently go dark at T+60 minutes. The `--refresh-test` code path demonstrates that manual refresh *works*, but it's a test mode, not the normal code path. The normal path has no refresh mechanism.

The fix is `AwsCredentialsProvider.new_delegate()` with a callback:

```python
def _credential_provider_callback():
    creds = get_cognito_credentials(pool_id, region, identity_id=cached_id)
    return AwsCredentials(
        access_key_id=creds['access_key_id'],
        secret_access_key=creds['secret_access_key'],
        session_token=creds['session_token'],
    )

credentials_provider = AwsCredentialsProvider.new_delegate(
    get_credentials=_credential_provider_callback
)
```

This makes every reconnect transparently fetch fresh credentials. No timer loops, no manual intervention.

**Connection to CRED-01:** The requirement verification table marks CRED-01 as `[x]` (complete). It isn't. The `--refresh-test` proves manual refresh works; it does not prove the production code path handles credential expiry. This should be marked incomplete or with a caveat.

### 2. IAM Architecture — Three Distinct Problems

The single unauthenticated role shared by publishers and subscribers creates three separate attack surfaces, not one:

#### 2a. Draw Result Injection

Any anonymous user can obtain Cognito credentials (no authentication required) and publish to `draws/quickdraw`. The Identity Pool ID is in CloudFormation outputs and would be in any client-side configuration. This means fake draw results can be injected to all 5,000 terminals from the public internet.

This is well-understood and acceptable for a POC. But I want to be precise about why it matters: in a lottery system, a single forged draw result could trigger incorrect payouts, regulatory investigation, or loss of gaming license. This isn't a theoretical risk — it's a compliance blocker.

#### 2b. Client ID Hijacking (Under-discussed)

The Connect permission allows any unauthenticated identity to connect as `terminal-*`. There's no binding between a Cognito identity and a specific client ID. An attacker can:

1. Connect as `terminal-001` with their own Cognito credentials
2. IoT Core disconnects the real terminal-001 (MQTT spec: same clientId = takeover)
3. Attacker receives terminal-001's draw results
4. Real terminal-001 misses draws until it reconnects, at which point the attacker can immediately reconnect and repeat

At scale, an attacker could cycle through all 5,000 client IDs, causing a cascading wave of disconnects. Each disconnect generates CloudWatch events at DEBUG level, amplifying logging costs.

**Production mitigation:** Use IoT Core's `${iot:Connection.Thing.ThingName}` policy variable to bind client IDs to provisioned Things, or use `${cognito-identity.amazonaws.com:sub}` to bind client IDs to specific Cognito identities.

#### 2c. The Publish Permission Exists on a Receive-Only Role

Even setting aside the authentication issue, the architectural statement is wrong. Subscribers should have `Connect + Subscribe + Receive` only. The publisher should be a completely separate principal — either an IAM user, an authenticated Cognito identity, or (best) an IoT certificate. This separation must be explicit in the production requirements, not just implied.

### 3. IoT Core Account-Level Side Effects

This is the category I want to flag most strongly as an infrastructure engineer. Several resources in this stack have account-wide scope.

#### 3a. `AWS::IoT::Logging` Is Account-Wide and Singleton

```yaml
IoTLogging:
  Type: AWS::IoT::Logging
  Properties:
    AccountId: !Ref AWS::AccountId
    DefaultLogLevel: DEBUG
    RoleArn: !GetAtt IoTLoggingRole.Arn
```

This resource configures IoT Core logging for the **entire account** in this region, not just for this POC's topics. If there are other IoT workloads in the same account, they all get DEBUG logging now. If another team or stack has their own `AWS::IoT::Logging` resource, deploying this stack overwrites their configuration.

There can only be one `AWS::IoT::Logging` per account per region. This is a singleton resource masquerading as a stack-scoped resource. Deleting this stack removes the logging configuration for the entire account.

**For the POC:** If this is a dedicated test account, no problem. If it's a shared account, this could be disruptive.

**For production:** IoT Core logging should be managed at the account/organizational level, not per-application.

#### 3b. Orphaned CloudWatch Log Group

The `AWSIotLogsV2` log group is created automatically by IoT Core on the first event. It is NOT part of this CloudFormation stack. When you delete the stack:
- The IAM roles are deleted ✓
- The Cognito pool is deleted ✓
- The IoT logging configuration is deleted ✓
- The `AWSIotLogsV2` log group with all its data **remains** ✗

At DEBUG level during testing, this log group contains connection events, topic names, client IDs, and trace IDs. It's not sensitive data per se, but it's operational metadata that persists after teardown.

**Fix:** Either add an explicit `AWS::Logs::LogGroup` resource (which gives you retention control), or document the manual cleanup step in the teardown instructions.

#### 3c. Named IAM Roles Block Multi-Stack Deployment

```yaml
RoleName: !Sub '${ProjectName}-unauth-role'
RoleName: !Sub '${ProjectName}-iot-logging-role'
```

IAM role names are globally unique within an account. If someone deploys two stacks with the same `ProjectName` (or forgets to change the default), the second deployment fails with a `RoleName already exists` error. This is a known CloudFormation anti-pattern for anything you might deploy more than once.

**Fix (production):** Remove explicit `RoleName` properties and let CloudFormation generate unique names, or use a unique suffix (e.g., stack ID).

### 4. Reliability at 5,000 Terminals — The Rate Limit Wall

This is where "Conditional Go" needs more conditions.

#### 4a. Connection Rate Throttle

AWS IoT Core has a default **connect rate limit** (historically 500 connections/second per account, though this varies by region and can be increased via service quotas). A simultaneous restart of 5,000 terminals (e.g., after a fleet-wide power cycle or software update) would hit this limit hard.

At 500/second, connecting all terminals takes a minimum of 10 seconds. With throttling, exponential backoff, and retry, it could take minutes. During this window, terminals are offline and miss any draws published.

**The CRT SDK handles throttling gracefully** with backoff, so connections will eventually establish. But the time-to-full-fleet-online matters for a system with 4-minute draw cycles.

#### 4b. Cognito GetId Throttle

`GetId` for unauthenticated identities has its own rate limit (~5 RPS by default, though this is not well-documented). On first deployment of 5,000 terminals, each needs to call `GetId` once to obtain an identity. At 5 RPS, that's ~17 minutes to provision all identities. After that, identity IDs are cached and only `GetCredentialsForIdentity` is needed for refreshes.

This is a one-time cost unless the identity cache is lost (e.g., terminal reimaging). But it's worth knowing about.

#### 4c. Persistent Session Queue Depth

IoT Core's persistent session has a limit on queued messages. If a terminal is offline and multiple draws are published, the oldest messages may be dropped when the queue limit is reached. The README notes the 1-hour session expiry, but doesn't mention message queue depth.

I'm flagging this as something to verify against current AWS documentation rather than asserting a specific number. The limit has changed over time and may depend on the IoT Core tier.

#### 4d. Keep-Alive at 30 Seconds

`keep_alive_secs=30` means each terminal sends a PINGREQ every 30 seconds. At 5,000 terminals: ~167 PINGREQs/second sustained. IoT Core can handle this, but it contributes to:
- Message broker load (each PINGREQ is a packet processed by the broker)
- CloudWatch events at DEBUG level (PINGREQ/PINGRESP are logged)
- Network bandwidth on the terminal side (minimal, but non-zero)

For production, 60–300 seconds is more appropriate. The tradeoff is detection time for dead connections — at 300 seconds, it takes ~7.5 minutes to detect an unresponsive terminal (1.5× keep-alive per MQTT spec).

### 5. The Verification Table — Two False Passes

#### 5a. LOGS-03 — Marked Pass, Actually Refuted

The requirement: "traceId correlates a single Publish-In to all corresponding Publish-Out entries."
The finding: "traceId does NOT correlate Publish-In to Publish-Out — each gets unique traceId."

The verification table marks this `[x]`. This is a false pass. Any stakeholder reading the summary table (which is the most likely read path for a go/no-go decision) concludes that traceId correlation works. It doesn't.

The FINDINGS.md prose is honest about this — the finding is clearly documented. But the verification table contradicts the prose. This should be marked `[~]` or `[ ]` with a note.

I understand why it happened — the requirement was "answered" (the POC determined the behavior), even though the answer was negative. The table conflates "we investigated this" with "this works." It needs a clear distinction.

#### 5b. CRED-01 — Implicit Validation Only

FINDINGS.md states: "Credential refresh was implicitly validated during auto-reconnects." The auto-reconnect in Experiment 2 worked because it happened within seconds of the original credential issuance — the credentials were still valid. This doesn't validate the refresh path.

The `--refresh-test` code exists and presumably works, but FINDINGS.md says Experiment 4 was "not yet tested separately." Either run it and document results, or mark CRED-01 as unvalidated.

### 6. The Cost Surprise Nobody Mentions Early Enough

At DEBUG level with 5,000 terminals and 4-minute draw cycles:

**Per draw cycle (minimum):**
- 1 Publish-In event
- 5,000 Publish-Out events
- Overhead: Connect/Disconnect events for churning terminals, Subscribe events on reconnect

**Per day (360 draw cycles):**
- ~1,800,000 Publish-Out events alone
- Plus session lifecycle, keepalive (at DEBUG), and Subscribe events
- Rough estimate: 3–5 million log events per day

**CloudWatch Logs ingestion:** ~$0.50/GB. Each event is roughly 300–500 bytes. At 4M events × 400 bytes = ~1.6 GB/day = ~$0.80/day = ~$24/month just for log ingestion. Storage accrues on top of that if there's no retention policy.

This isn't a showstopper, but it's worth knowing before someone is surprised by the first bill. In production, drop to ERROR or INFO. The CloudFormation template should parameterize `DefaultLogLevel`.

---

## Second Pass — Sweep

Things I noticed on a quick re-read that didn't fit the deep dive:

### S-01. Environment Variable Naming Mismatch

`deploy.sh` uses `AWS_DEFAULT_REGION` (AWS CLI convention):
```bash
REGION="${AWS_DEFAULT_REGION:-us-east-1}"
```

`config.py` and `.env.example` use `AWS_REGION`:
```python
AWS_REGION = os.environ.get('AWS_REGION', 'us-east-1')
```

These are different environment variables. If a user sets `AWS_REGION=us-west-2` in their `.env` file but doesn't set `AWS_DEFAULT_REGION`, `deploy.sh` deploys to `us-east-1` while the scripts connect to a `us-west-2` endpoint. The stack and the scripts end up in different regions. This is a real misconfiguration waiting to happen.

### S-02. `awscrt` Is a Transitive Dependency

`publisher.py` and `subscriber.py` import `from awscrt import mqtt` directly. But `requirements.txt` only lists `awsiotsdk`, which transitively depends on `awscrt`. If `awsiotsdk` ever restructured its dependencies (unlikely but possible), or if someone vendored dependencies differently, the direct import would break.

**Fix:** Add `awscrt>=0.24.0` to `requirements.txt` as an explicit dependency.

### S-03. `connect_future.result()` Returns a Dict, Not a Bool

```python
session_present = connect_future.result()
print(f"Connected as '{args.client_id}', session_present={session_present}")
```

The CRT SDK's `connect()` resolves to `{'session_present': True/False}`. The variable `session_present` holds the entire dict. The print output would show `session_present={'session_present': True}`. More critically, any future `if session_present:` check always evaluates truthy (non-empty dict), regardless of actual session state.

This appears in three places in `subscriber.py` (initial connect, reconnect test, refresh test).

**Fix:**
```python
connect_result = connect_future.result()
session_present = connect_result.get('session_present', False)
```

### S-04. No Timeout on Any `.result()` Call

`connect_future.result()`, `publish_future.result()`, `subscribe_future.result()`, and `disconnect().result()` all block indefinitely. If the broker is unreachable, the script hangs forever with no error message. For a POC this is annoying; for a production terminal it's a hung process that never recovers.

### S-05. `config.py` Raises Bare `KeyError`

```python
IOT_ENDPOINT = os.environ['IOT_ENDPOINT']
IDENTITY_POOL_ID = os.environ['IDENTITY_POOL_ID']
```

If `.env` is missing or incomplete, the user sees `KeyError: 'IOT_ENDPOINT'` with no context. A 1-line improvement:

```python
IOT_ENDPOINT = os.environ.get('IOT_ENDPOINT') or sys.exit("ERROR: IOT_ENDPOINT not set. Copy .env.example to .env and fill in values.")
```

### S-06. `cmd_trace` Ignores `--minutes`

```python
def cmd_trace(args):
    ...
    start_time = end_time - (60 * 60)  # hardcoded 1 hour
```

Every other subcommand uses `args.minutes`. `cmd_trace` hardcodes 60 minutes. Passing `--minutes 120` has no effect on trace queries.

### S-07. SUBACK QoS Not Validated

```python
subscribe_result = subscribe_future.result()
print(f"Subscribed to '{config.TOPIC}' with QoS {subscribe_result['qos']}")
```

The code logs the granted QoS but doesn't check it. IoT Core can return QoS 128 (subscription failure) in the SUBACK — particularly during the 6–8 minute IAM policy propagation window documented in the README. The subscriber would print the failure, then sit in its `while True: sleep(1)` loop forever, believing it's subscribed, receiving nothing.

### S-08. Unhandled Exception in Message Callback

```python
def on_message_received(topic, payload, dup, qos, retain, **kwargs):
    message = json.loads(payload)
```

This runs on a CRT background thread. If `json.loads()` throws (malformed payload, encoding issue, empty bytes), the exception is swallowed by the SDK's callback dispatcher. The draw result is silently dropped. The terminal displays nothing. No error output.

### S-09. No Deduplication

QoS 1 guarantees at-least-once delivery — FINDINGS.md documents that Experiment 2 produced duplicate delivery. The subscriber processes every message without checking `draw_id`. For a display-only terminal this is probably benign (same numbers shown twice). For any downstream action (printing, logging to a database, triggering a payout check), duplicates are a problem.

### S-10. Publisher Has No Graceful Shutdown

`publisher.py`'s `while True: input()` loop has no try/except for `KeyboardInterrupt`. Ctrl+C during `input()` raises `KeyboardInterrupt`, skipping the `disconnect().result()` call. The MQTT connection is abandoned without a clean DISCONNECT packet. IoT Core treats this as an ungraceful disconnect and may queue messages for the publisher's client ID if persistent sessions are involved (they aren't, because `clean_session=True`, but it's still sloppy).

### S-11. `deploy.sh` Missing `--no-fail-on-empty-changeset`

Re-running `deploy.sh` without changes fails with "No changes to deploy." This is a common CloudFormation gotcha. Adding `--no-fail-on-empty-changeset` makes the script idempotent.

### S-12. `on_connection_resumed` Doesn't Handle `session_present=False`

The callback prints session state but takes no action:

```python
def on_connection_resumed(connection, return_code, session_present, **kwargs):
    print(f"Connection resumed: return_code={return_code}, session_present={session_present}")
```

If `session_present` is False on resume, it means the persistent session was lost — subscriptions and queued messages are gone. The terminal needs to re-subscribe. In the normal code path (no test flags), this doesn't happen because the initial subscribe is already done. But after an auto-reconnect where the session was lost, the terminal silently stops receiving messages.

In a production system, this callback should trigger a re-subscribe.

### S-13. `query_logs.py` Query Polling Has No Timeout

```python
while True:
    time.sleep(1)
    result = client.get_query_results(queryId=query_id)
    if result['status'] in ('Complete', 'Failed', 'Cancelled', 'Timeout', 'Unknown'):
        break
```

CloudWatch Insights queries have a 15-minute server-side timeout, so the practical maximum hang is 15 minutes. But the operator has no feedback during this period and no way to abort gracefully.

### S-14. `query_logs.py` Trace ID Injection

```python
f"| filter traceId = '{args.trace_id}'\n"
```

`args.trace_id` is interpolated directly into the CloudWatch Insights query. A crafted value containing `'` and `|` could inject arbitrary query clauses. CloudWatch Insights isn't SQL, but it does have a query language that supports `display`, `stats`, and `filter` — an injected clause could exfiltrate data from other log entries or cause unexpected query behavior.

For a dev-only CLI tool this is low risk. The pattern is dangerous to copy into any API or automation pipeline.

### S-15. `draw_id` Has Second-Precision Collision Potential

```python
'draw_id': f'draw-{int(time.time())}',
```

Two publishes in the same second produce identical `draw_id` values. At 4-minute intervals this is practically impossible. But `draw_id` is the proposed payload-level correlation key for production delivery tracking. If this pattern is carried forward, use `uuid.uuid4()` or at minimum millisecond precision.

### S-16. SIGSTOP Is POSIX-Only

```python
os.kill(os.getpid(), signal.SIGSTOP)
```

Raises `AttributeError` on Windows. The `--freeze-after-subscribe` flag is a test mode and the real testing used iptables anyway, so this is low impact. But it's a crash waiting for someone who runs tests on Windows.

### S-17. `LOG_GROUP` Is Hardcoded

```python
LOG_GROUP = 'AWSIotLogsV2'
```

If IoT Core's log group name changes (unlikely) or if someone configures a custom log group, this silently returns no results. Minor, but worth parameterizing.

### S-18. No `requirements.txt` Upper Bound on boto3

```
boto3>=1.42.0
```

Floor-only version constraint. A future boto3 release with breaking API changes (rare but has happened with service model changes) could break the scripts silently. Consider `boto3>=1.42.0,<2.0.0`.

---

## Devil's Advocate — What Would an Attacker Target?

Setting aside the obvious (unauthenticated publish = draw injection), here's what I'd look at if I were actively trying to cause harm:

### Attack 1: Client ID Takeover Cascade

MQTT spec: if a client connects with a clientId that's already connected, the broker disconnects the existing client. There's no authentication binding between a Cognito identity and a client ID in this architecture.

**Attack:** Connect as `terminal-001`. Real terminal-001 gets kicked. It auto-reconnects (74ms). I reconnect immediately. We enter a reconnection storm. Scale this to all 5,000 client IDs — one attacker process cycling through `terminal-0001` to `terminal-5000` could cause fleet-wide instability.

**Cost amplification:** Every connect/disconnect generates CloudWatch events at DEBUG level. A rapid takeover cycle against 5,000 client IDs could generate millions of log events per minute, inflating CloudWatch costs.

**Mitigation:** Bind client IDs to specific identities using IoT Core policy variables (`${iot:ClientId}` matched against `${cognito-identity.amazonaws.com:sub}`), or use certificate-based auth where the client ID is embedded in the certificate's Common Name.

### Attack 2: Persistent Session Poisoning

1. Connect as `terminal-001` with `clean_session=False`
2. Subscribe to `draws/quickdraw`
3. Disconnect cleanly
4. Real terminal-001 connects — its session might be corrupted or the attacker's subscription state might interfere
5. Messages could be queued for the attacker's session state rather than the real terminal's

The exact behavior depends on how IoT Core handles session ownership transfer, but the attack surface exists because client ID = session ID in MQTT, and client IDs aren't bound to identities.

### Attack 3: Cognito Identity Exhaustion

Repeatedly call `GetId` without caching the identity. Each call creates a new identity in the pool. At sufficient volume, this could hit Cognito's identity count limits or generate unexpected costs. Cognito identity pools don't have a built-in cap on identity count.

**Mitigation:** AWS WAF on the Cognito endpoint, or rate limiting at the application layer. For production, use authenticated identities or certificate-based auth.

### Attack 4: Log Exfiltration via `cmd_raw`

`query_logs.py raw --query` accepts arbitrary CloudWatch Insights queries. If this tool were ever accessible beyond a developer's local machine (e.g., via a web UI, API, or shared automation), an attacker could query any log group the IAM user has access to, not just `AWSIotLogsV2`.

---

## Complete Findings Catalog

Everything I noticed, with severity separated from the finding itself. I'd rather you see everything and decide what matters.

### Findings That Actually Matter

| ID | Category | Finding | Severity | Why It Matters |
|----|----------|---------|----------|---------------|
| F-01 | Credential Lifecycle | `new_static()` bakes credentials at creation time; auto-reconnect after ~1hr uses expired creds → silent terminal death | **Critical** | Every terminal goes dark at T+60min in production |
| F-02 | IAM Architecture | Single unauthenticated role has iot:Publish — anyone can inject fake draws | **Critical** (production) | Regulatory/integrity blocker for lottery system |
| F-03 | Decision Integrity | LOGS-03 marked `[x]` but finding refutes it — traceIds don't correlate | **High** | Misleads architecture decisions for delivery tracking |
| F-04 | Decision Integrity | CRED-01 marked complete but only implicitly tested | **High** | Production readiness gap masked as validated |
| F-05 | Attack Surface | Client ID hijacking — no binding between identity and clientId | **High** (production) | Fleet-wide DoS via connection takeover cascade |
| F-06 | Account Scope | `IoT::Logging` is account-wide singleton — overwrites other IoT logging configs | **Medium** | Breaks other IoT workloads in shared accounts |
| F-07 | Protocol | SUBACK QoS not validated — terminal silently non-subscribed during IAM propagation | **Medium** | 6–8 min window where terminals believe they're subscribed but aren't |
| F-08 | Reliability | `on_connection_resumed` doesn't re-subscribe when `session_present=False` | **Medium** | Terminal silently stops receiving after session loss |
| F-09 | Scale | IoT Core connect rate limit (~500/s default) — fleet restart takes 10+ seconds | **Medium** | Terminals miss draws during reconnection storm |
| F-10 | Cost | DEBUG logging at 5,000 terminals ≈ 3–5M events/day ≈ $24+/month just ingestion | **Medium** | Cost surprise; also no retention policy = unbounded storage |

### Findings That Are Real But Lower Priority

| ID | Category | Finding | Severity | Notes |
|----|----------|---------|----------|-------|
| F-11 | Env Vars | `deploy.sh` uses `AWS_DEFAULT_REGION`, config.py uses `AWS_REGION` — different vars | **Low-Med** | Real cross-region deployment bug for inattentive users |
| F-12 | Robustness | All `.result()` calls block forever with no timeout | **Low** | Hung process on bad config; annoying in dev, dangerous in prod |
| F-13 | Robustness | `json.loads()` in message callback — unhandled exception silently drops messages | **Low-Med** | Silent message loss on malformed payload |
| F-14 | Code | `connect_future.result()` returns dict, not bool — session_present is always truthy | **Low** | Wrong log output; future conditional checks broken |
| F-15 | UX | `config.py` raises bare `KeyError` on missing env vars | **Low** | Bad first-run experience |
| F-16 | Logic | `cmd_trace` hardcodes 1-hour window, ignores `--minutes` | **Low** | Inconsistency; confusing for users |
| F-17 | Security | Query injection in `query_logs.py` trace_id interpolation | **Low** | Dev-only CLI tool; dangerous if pattern is copied |
| F-18 | Dependencies | `awscrt` imported directly but not in requirements.txt (transitive only) | **Low** | Works today; fragile |
| F-19 | Dependencies | `boto3>=1.42.0` has no upper bound | **Low** | Future-proofing |
| F-20 | Code | `draw_id` uses second-precision timestamp — collision potential | **Low** | Not a problem at 4-min intervals; bad pattern to carry forward |
| F-21 | Portability | `signal.SIGSTOP` crashes on Windows | **Low** | Test mode only; iptables is the real approach anyway |
| F-22 | Code | Publisher has no Ctrl+C handling — skips disconnect | **Low** | Sloppy but inconsequential for POC |
| F-23 | Infra | `deploy.sh` missing `--no-fail-on-empty-changeset` | **Low** | Re-deploy without changes fails |
| F-24 | Infra | Named IAM roles block multi-stack deployment with same ProjectName | **Low** | Only matters if someone deploys twice |
| F-25 | Cleanup | Orphaned `AWSIotLogsV2` log group survives stack deletion | **Low** | Operational metadata persists after teardown |
| F-26 | Security | IoT logging role `Resource: '*'` — can write to any log group | **Low** | Standard least-privilege gap; low real-world risk |
| F-27 | Protocol | No duplicate message deduplication (QoS 1 = at-least-once) | **Low** | Display-only is idempotent; matters for downstream actions |
| F-28 | Scale | `keep_alive_secs=30` → 167 PINGREQ/s at 5,000 terminals | **Info** | Not a problem for IoT Core; worth benchmarking |
| F-29 | Config | `LOG_GROUP = 'AWSIotLogsV2'` is hardcoded | **Info** | Minor inflexibility |
| F-30 | Config | `TOPIC` is hardcoded in config.py, not configurable via env | **Info** | Fine for POC |

---

## Assessment of the "Conditional Go" Conclusion

The conclusion is honest, directionally correct, and appropriately scoped. I agree with it — with amendments to the conditions list.

**What the conclusion gets right:**
- IoT Core solves the core fan-out problem (router CPU)
- QoS 1 provides at-least-once delivery guarantees that UDP multicast can't
- Publish-Out `status=Success` ≠ PUBACK — this is the key finding and it's clearly stated
- Payload-level correlation (ACK topic) is the correct production approach
- The POC doesn't oversell

**What the conditions list is missing:**

1. **Credential lifecycle is a production blocker.** The `new_static()` pattern means terminals die after 1 hour. This isn't a "nice to have" — it's a prerequisite for any deployment beyond a demo. The conditions should explicitly require either `new_delegate()` credentials or certificate-based auth.

2. **IAM role separation is a compliance blocker.** The conditions mention "certificate-based auth for production" but don't explicitly call out that the current IAM architecture allows anonymous draw injection. For a lottery system, this is a regulatory issue, not just a security best practice.

3. **Scale testing is not optional.** The gap between 2 terminals and 5,000 is not just a confidence gap — it's where rate limits, throttling, connection storms, and Cognito throughput all become real factors. A 50–100 subscriber scale test should be a condition of the Go, not a "next step."

4. **Session loss handling is unaddressed.** If a terminal's persistent session expires (1-hour default) or is lost due to a reconnect with `session_present=False`, the terminal silently stops receiving messages. The production architecture needs either a re-subscribe mechanism or a fallback (retained message, HTTP endpoint) for terminals to catch up.

**My recommendation:** The "Conditional Go" should be upgraded to "Conditional Go with Hard Prerequisites" — where the prerequisites are:
- Credential lifecycle (delegate provider or certificates)
- IAM role separation (publisher ≠ subscriber)
- Scale test at 50–100 terminals (proves rate limits and Cognito throughput)
- Session loss recovery mechanism

Without these four, you're not going to production. With them, this architecture is sound.

---

## One Thing That Impressed Me

The decision to document the SIGSTOP dead end alongside the iptables success. Most POC documents only show what worked. Showing what didn't work — and *why* it didn't (SIGSTOP kills keepalive, which triggers disconnect before you can test PUBACK) — builds trust in the findings. It tells me the experimenter understood the system deeply enough to explain failure modes, not just observe them.

The traceId finding (Publish-In and Publish-Out have different traceIds) is also genuinely valuable tribal knowledge. AWS's documentation on this is sparse to nonexistent. This finding will save someone hours.

---

## Things I'm Uncertain About

In the interest of honesty:

- **IoT Core persistent session queue depth limits.** I believe there's a per-session limit on queued messages, but I'm not confident in the exact number and it may have changed. Worth verifying against current AWS documentation before sizing the production architecture.

- **Cognito `GetId` rate limits.** I quoted ~5 RPS for unauthenticated identities. This is my best recollection, but I haven't verified it recently. The actual limit matters for fleet provisioning time.

- **`new_delegate()` API availability.** I recommended this as the fix for static credentials. I'm confident the CRT SDK supports delegate credentials providers, but the exact Python API method name should be verified against the current `awscrt` version (0.24.x range given the awsiotsdk 1.28.2 pin).

I'd rather flag my uncertainty than state these as facts.

---

*Review performed against code as provided. No files were executed. Infrastructure assessment based on AWS IoT Core, Cognito, and CloudFormation behavior as of early 2026.*
