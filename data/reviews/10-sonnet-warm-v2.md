# AWS IoT Core POC — Code Review
**Reviewer:** Ren (cloud infrastructure, AWS IoT)  
**Model:** claude-sonnet-4.x (warm context, second version)  
**Date:** 2025-07-24  
**Scope:** Full codebase — security, scale, code quality, "Conditional Go" assessment

---

## How I Read This

I went two passes. First pass was slow — I traced through the code as if I were going to operate this in production, not just skim it. Second pass was a sweep for anything I underweighted or skipped. Then I spent time in adversarial mode asking what someone trying to break this would go after.

One honest note before diving in: the architecture is clean and the code is readable. This is a well-structured POC. The findings below are real, but they're the kind of findings that exist precisely *because* the project is at decision stage — the question "should we productionize this?" requires asking harder questions than "does it work?"

---

## Pass 1 — Deep Findings

### F01 — Any Terminal Can Publish Fake Draw Results
**File:** `infra/template.yaml`, IAM policy under `UnauthRole`  
**Severity: HIGH**

The `iot:Publish` permission is granted to the same role that all terminals use. That means any terminal — or anyone who obtains a Cognito unauthenticated credential — can publish to `draws/quickdraw` and inject fake lottery results. There's no IAM separation between "publisher" and "subscriber" identities. The `client_id='publisher-tpm'` naming convention is just cosmetic; it has no enforcement weight.

```yaml
# This single role covers both terminals AND publisher
- Effect: Allow
  Action: iot:Publish
  Resource:
    - !Sub 'arn:aws:iot:${AWS::Region}:${AWS::AccountId}:topic/draws/quickdraw'
```

For lottery results this isn't a theoretical risk — it's a fraud vector. The fix requires either: (a) separate IAM roles for publisher vs. subscribers enforced at the Cognito level, (b) X.509 device certificates with per-client IoT Core policies, or (c) a backend Lambda that's the only permitted publisher and never exposes those credentials to terminals.

---

### F02 — IoT Core Logging Is Set Account-Wide at DEBUG
**File:** `infra/template.yaml`, `AWS::IoT::Logging` resource  
**Severity: HIGH (cost) / MEDIUM (ops)**

`AWS::IoT::Logging` is an account-level resource, not stack-level. Deploying this changes IoT logging for the *entire AWS account*, not just this POC. Setting `DefaultLogLevel: DEBUG` at 5,000 terminals will generate:

- Per draw: 1 Publish-In + ~5,000 Publish-Out + connect/disconnect/subscribe churn at DEBUG = easily 6,000–10,000+ log events per draw cycle
- At 4-minute intervals: ~90,000–150,000 log events/hour
- CloudWatch Logs ingestion is priced per GB — this adds up quickly

More critically: deleting this stack does *not* restore previous logging configuration. It leaves the account in whatever state CloudFormation last wrote. If someone had logging configured before this deployment, that config is gone.

For production, this needs to be INFO or ERROR, and the account-level side effect needs to be documented explicitly in the README.

---

### F03 — `session_present` Variable Holds the Wrong Type
**File:** `scripts/subscriber.py`, lines after `connect_future.result()`  
**Severity: MEDIUM (misleading output, latent logic bug)**

The `awsiotsdk` `connect()` future resolves to a `dict`, not a `bool`. Specifically `{'session_present': True/False}`. So when the code does:

```python
session_present = connect_future.result()
print(f"Connected as '{args.client_id}', session_present={session_present}")
```

It prints something like:
```
Connected as 'terminal-001', session_present={'session_present': True}
```

This is cosmetically wrong but more importantly, if any future code tried to branch on `session_present` as a boolean (e.g., `if session_present: skip_resubscribe()`), it would always evaluate `True` because a non-empty dict is truthy, even when the actual session was *not* present. The current code doesn't branch on it, so it doesn't fail today — but it's a trap.

---

### F04 — `json` Is Imported but Never Used in `query_logs.py`
**File:** `scripts/query_logs.py`, line 4  
**Severity: LOW (unused import)**

```python
import json  # Never referenced anywhere in this file
```

Minor, but it's worth noting because it suggests either (a) the module was refactored and this import was left behind, or (b) there was intent to add JSON output formatting that was dropped. If (b), the absence of machine-readable output from `query_logs.py` is itself a production readiness concern — log queries that return only human-readable stdout don't compose well with alerting or dashboards.

---

### F05 — Log Query Limit of 100 Silently Truncates at Scale
**File:** `scripts/query_logs.py`, `run_query()` default `limit=100`  
**Severity: MEDIUM**

```python
def run_query(query_string, start_time, end_time, region, limit=100):
```

The `cmd_delivery` subcommand queries delivery status per `clientId`. At 5,000 terminals, this query needs `limit=5000` at minimum. With `limit=100`, you silently get delivery data for 100 terminals and have no indication the rest were dropped. This makes the delivery confirmation mechanism — the *core value proposition* of this POC — unreliable at the scale it's being evaluated for.

The `cmd_delivery` call doesn't override the limit, and there's no CLI argument to set it. This needs to be configurable, and the default should not silently hide missing data.

---

### F06 — No Credential Expiry Handling
**File:** `scripts/auth.py`, `scripts/subscriber.py`, `scripts/publisher.py`  
**Severity: MEDIUM (at scale: HIGH)**

Cognito unauthenticated credentials expire after 1 hour. The `expiration` field is returned in the credentials dict but never checked or acted on anywhere. For a POC with two terminals and short test runs, this doesn't matter. For 5,000 terminals running 24/7:

- Credentials issued at session start expire ~1 hour later
- The MQTT connection may stay alive past credential expiry (keepalive keeps the TCP session up) — but the *next* reconnection attempt after a drop will fail with an auth error
- If 5,000 terminals were all deployed at roughly the same time (common in a rollout), they may all hit credential expiry at roughly the same time — thundering herd on Cognito

The credential refresh test mode exists and works, but it's manual and opt-in. Production systems need automatic proactive refresh before expiry, not reactive refresh on connection failure.

---

### F07 — `signal.SIGSTOP` Is Linux/macOS Only
**File:** `scripts/subscriber.py`, `--freeze-after-subscribe` path  
**Severity: LOW (dev/test tool, non-cross-platform)**

```python
os.kill(os.getpid(), signal.SIGSTOP)
```

`signal.SIGSTOP` is not defined on Windows. This will raise `AttributeError` on Windows hosts. For a development testing tool this is minor, but worth noting if team members are on Windows.

---

### F08 — No Error Handling in Message Callback
**File:** `scripts/subscriber.py`, `on_message_received` callback  
**Severity: MEDIUM**

```python
def on_message_received(topic, payload, dup, qos, retain, **kwargs):
    message = json.loads(payload)
```

If `payload` is not valid JSON — malformed message, encoding issue, or injected garbage (see devil's advocate section) — this raises an unhandled exception inside the CRT callback thread. The behavior of an unhandled exception in an `awsiot` callback is implementation-defined and could silently kill message delivery for that subscriber without any visible error. A try/except with a logged error is the minimum production requirement here.

---

### F09 — `ClientId` Is Untrusted and Unenforced
**File:** `infra/template.yaml`, IAM policy Connect ARN  
**Severity: MEDIUM**

```yaml
- !Sub 'arn:aws:iot:${AWS::Region}:${AWS::AccountId}:client/terminal-*'
```

The IAM policy allows any client ID matching `terminal-*`. Nothing prevents terminal hardware at `terminal-001` from connecting as `terminal-002`. Nothing prevents a rogue client from connecting as `terminal-9999` and receiving draw results (or publishing fake ones). With unauthenticated Cognito identities, there is no cryptographic binding between hardware identity and MQTT clientId.

For delivery confirmation this matters: if you're using `clientId` in CloudWatch logs to verify "terminal-001 received the draw," you can't actually verify it was the *real* terminal-001.

---

### F10 — `cmd_trace` Ignores the `--minutes` Argument
**File:** `scripts/query_logs.py`, `cmd_trace` function  
**Severity: LOW (minor inconsistency)**

The `--minutes` argument is registered at the top-level parser and available to all subcommands. `cmd_trace` hardcodes its own window:

```python
end_time = int(time.time())
start_time = end_time - (60 * 60)  # always 60 minutes, ignores --minutes
```

The comment explains the reasoning ("traceId is already specific"), and the reasoning is fine. But it silently ignores a flag the user explicitly set. At minimum, the help text for `trace` should document this behavior. A reasonable alternative would be `max(args.minutes, 60) * 60`.

---

### F11 — `AWS_REGION` vs `AWS_DEFAULT_REGION` Mismatch
**File:** `scripts/config.py` vs `infra/deploy.sh`  
**Severity: LOW (but operationally confusing)**

`deploy.sh` reads `AWS_DEFAULT_REGION` (the standard AWS CLI env var):
```bash
REGION="${AWS_DEFAULT_REGION:-us-east-1}"
```

`config.py` reads `AWS_REGION` (a project-specific var, set in `.env`):
```python
AWS_REGION = os.environ.get('AWS_REGION', 'us-east-1')
```

These can diverge. If a user has `AWS_DEFAULT_REGION=us-west-2` in their shell (a common global config) but doesn't set `AWS_REGION` in `.env`, the stack deploys to `us-west-2` while the Python scripts connect to `us-east-1`. The deploy script could explicitly emit the region it used and recommend adding it to `.env`, which it currently does not.

---

### F12 — Cognito Identity Proliferation
**File:** `scripts/auth.py`, `get_cognito_credentials`  
**Severity: LOW (cost, hygiene)**

Every call to `get_cognito_credentials(identity_pool_id, region)` *without* a cached `identity_id` creates a new Cognito identity. The code handles this correctly *when the caller caches `identity_id`* — subscriber.py does this for its reconnect and refresh tests. But `publisher.py` never caches the identity:

```python
credentials = get_cognito_credentials(config.IDENTITY_POOL_ID, config.AWS_REGION)
# identity_id from credentials is never stored or reused
```

Each publisher restart creates a new identity. Not catastrophic for a POC, but at scale (or with automated restarts), this accumulates orphaned identities that cost money and pollute the identity pool. The fix is a one-liner: persist `identity_id` to disk or env on first run.

---

### F13 — Publish as the Only Point of Failure
**File:** `scripts/publisher.py`  
**Severity: MEDIUM (architecture)**

The publisher is a single script with no redundancy. `publish_future.result()` blocks indefinitely if the broker is unreachable (no timeout argument). There's no retry logic, no dead-letter mechanism, no watchdog. For a 4-minute draw cycle where a missed publish means 5,000 terminals don't get results, this is a reliability gap. Not a code bug, but a production readiness gap.

---

### F14 — IoT Core Persistent Session Queue Depth
**File:** architecture concern  
**Severity: MEDIUM**

IoT Core queues messages for offline persistent session clients, but the default queue depth is **100 messages per session**. If a terminal is offline for multiple draw cycles, messages beyond 100 are dropped silently. With 4-minute draw cycles, a terminal offline for 7+ hours would start losing messages, and the subscriber would have no indication of the gap when it reconnects. Production systems need either: (a) a heartbeat check that detects offline terminals and alerts, (b) message IDs with gap detection at the subscriber, or (c) a separate reconciliation mechanism.

---

## Pass 2 — Quick Sweep

Things I caught on second look that didn't warrant full sections:

- **S01:** `cmd_delivery` doesn't print query statistics (recordsScanned), unlike `cmd_trace` and `cmd_raw`. Minor inconsistency in the tooling.
- **S02:** `.env.example` shows `IOT_ENDPOINT=your-endpoint-ats.iot.us-east-1.amazonaws.com` with `us-east-1` baked into the placeholder. Users in other regions copying this literally will have a subtly wrong endpoint format.
- **S03:** `deploy.sh` outputs the IoT endpoint but doesn't offer to append it to `.env`. This is a manual copy-paste step that's easy to get wrong.
- **S04:** `requirements.txt` pins `awsiotsdk==1.28.2` (good) but leaves `boto3` and `python-dotenv` as minimum-version ranges, which could introduce regressions if those packages release breaking changes.
- **S05:** `keep_alive_secs=30` is hardcoded in `create_mqtt_connection`. At 5,000 terminals, the keepalive traffic itself is meaningful. Each client pings every 30 seconds = ~167 pings/second inbound to the broker. Configurable is better.
- **S06:** The reconnect test in subscriber.py creates a new `mqtt_connection` object but assigns it to the same variable name. The old object is still connected (disconnected explicitly before) so there's no leak, but it relies on the explicit `disconnect()` call having worked. If `disconnect()` failed, you'd have two connections from the same clientId, and IoT Core would kick the first one. This would *probably* surface as expected behavior in testing but could confuse debugging.
- **S07:** No CloudWatch log retention policy is set for `AWSIotLogsV2`. Logs accumulate indefinitely. At the DEBUG volume described in F02, this becomes expensive quickly.
- **S08:** The `ProjectName` parameter defaults to `iot-poc`. Two separate CloudFormation stacks using the default project name will conflict on IAM role names since they use `!Sub '${ProjectName}-unauth-role'`. The role names are deterministic, not unique to the stack.

---

## Devil's Advocate — Attack Surfaces

What would someone trying to break this look for?

### Attack 1: Fake Results Injection
Covered in F01, but worth restating in adversarial terms: the barrier to injecting fake lottery results is obtaining a Cognito unauthenticated identity — which requires knowing only the `IDENTITY_POOL_ID`. That ID is visible in CloudFormation stack outputs, in `.env` files on terminal hardware, and in any CloudWatch logs that include it. Anyone with physical access to one terminal, or network access to observe the Cognito handshake, can obtain credentials and publish fake results. No exploit required.

### Attack 2: Subscriber DoS via Malformed Payload
A permitted publisher (or attacker with F01 credentials) can publish a non-JSON payload to `draws/quickdraw`. Every subscriber's `on_message_received` callback calls `json.loads(payload)` without try/except (F08). Depending on how `awsiotsdk` handles callback exceptions, this could crash each subscriber's message processing silently. One malformed message, sent to one topic, would affect all 5,000 terminals simultaneously.

### Attack 3: Session Exhaustion
IoT Core persistent sessions have storage limits. An attacker with `terminal-*` credentials could connect 5,000 fake clientIds, subscribe with `clean_session=False`, and disconnect — forcing IoT Core to maintain 5,000 persistent sessions and queue future messages for all of them. This consumes IoT Core session storage and could delay or drop real message delivery to real terminals. Rate limits exist but the default thresholds are generous.

### Attack 4: ClientId Squatting
An attacker who knows terminal clientId naming conventions (`terminal-001` through `terminal-5000`) could connect as a legitimate terminal's clientId before the real terminal does, preventing the real terminal from connecting (IoT Core kicks the old session when a new one connects with the same clientId). This would require timing, but during a deploy or mass reconnect event, the window exists.

### Attack 5: Credential Harvest via Log Exfiltration
At DEBUG logging level, IoT Core logs include client metadata. If `AWSIotLogsV2` CloudWatch permissions are misconfigured or exfiltrated, an attacker could enumerate all active clientIds and their connection patterns — useful for targeted attacks or for understanding the system topology before a more sophisticated attack.

---

## Severity Matrix

| ID | Finding | Severity | Blocks Production? |
|----|---------|----------|--------------------|
| F01 | Any terminal can publish fake results | **HIGH** | Yes |
| F02 | Account-wide DEBUG logging, no retention policy | **HIGH** | Yes (cost) |
| F03 | `session_present` holds dict not bool | **MEDIUM** | Latent |
| F04 | Unused `json` import in query_logs.py | LOW | No |
| F05 | Delivery query limit=100 silently truncates | **MEDIUM** | Yes (for 5k) |
| F06 | No credential expiry handling | **MEDIUM** | Yes (for 5k) |
| F07 | SIGSTOP not cross-platform | LOW | No |
| F08 | No error handling in message callback | **MEDIUM** | Yes |
| F09 | ClientId untrusted, unenforced | **MEDIUM** | Depends on threat model |
| F10 | `cmd_trace` ignores `--minutes` | LOW | No |
| F11 | `AWS_REGION` vs `AWS_DEFAULT_REGION` mismatch | LOW | Operationally risky |
| F12 | Cognito identity proliferation | LOW | No |
| F13 | Publisher is single point of failure | **MEDIUM** | Depends |
| F14 | IoT Core session queue depth limit | **MEDIUM** | Edge case |
| S01 | Inconsistent stats printing in query_logs | LOW | No |
| S02 | `.env.example` has hardcoded region in endpoint | LOW | No |
| S03 | `deploy.sh` doesn't write to `.env` | LOW | No |
| S04 | Unpinned boto3/dotenv in requirements.txt | LOW | No |
| S05 | Hardcoded keepalive | LOW | No |
| S06 | mqtt_connection variable reassignment | LOW | No |
| S07 | No log retention policy | LOW | Cost |
| S08 | Deterministic IAM role names conflict on redeploy | LOW | Edge case |

---

## Assessment: Is the "Conditional Go" Justified?

### What the POC Got Right

The core question — *does Publish-Out status=Success mean client receipt or broker dispatch?* — was correctly answered and correctly documented. That's the most important thing this POC was supposed to figure out, and it did. The architecture is well-chosen for the problem: MQTT QoS 1 over WebSocket with SigV4 is a legitimate replacement for UDP multicast. Persistent sessions are the right answer for terminals that reconnect.

### Where "Conditional Go" Is Premature

The "Conditional Go" recommendation needs to be more conditional than it currently is. There are findings here that are not just engineering debt — they're blockers for a lottery result delivery system:

**F01 is a real blocker.** The current architecture has no mechanism preventing any terminal from injecting fake draw results. For a lottery system, this is fraud exposure, not a hardening-later concern. The recommendation should call out that the authentication architecture needs to change before production, not be noted as a future improvement.

**F05 is a functional blocker** specifically for the delivery confirmation feature. If the delivery query silently truncates at 100 when you have 5,000 terminals, the entire premise of using CloudWatch logs for delivery confirmation is broken at scale. The POC was validated at 2 terminals; it hasn't been validated at the scale it's being recommended for.

**The POC hasn't been load tested.** There's a meaningful difference between "works with 2 terminals" and "works with 5,000 terminals." IoT Core has per-second connection rate limits (default 500/second), publish throttles, and session storage limits. A mass reconnect event or a deployment rollout could exceed these limits. The recommendation should require a load test, not assume scale will be linear.

**F02 is a cost blocker.** DEBUG logging at production scale with no retention policy is not a hardening-later item — it's a budget item that will surprise someone.

### Revised Recommendation

If I were writing the conclusion, it would read: **Conditional Go on the architecture, Not Go on this codebase.** The MQTT-over-WebSocket approach is sound. The specific implementation needs: (a) role separation for publishers vs. subscribers, (b) either X.509 certs or a hardened Cognito architecture that prevents terminal-to-publisher privilege escalation, (c) logging level and retention configuration before production, (d) a load test at 10–20% of target scale before committing, and (e) credential lifecycle management. These aren't polish items — several of them are prerequisites for operating a system that delivers lottery results.

---

## One Thing That Feels Off But I Can't Fully Prove

The `traceId` finding in FINDINGS.md says traceId does not correlate Publish-In to Publish-Out. This is correct as documented. But I notice the POC doesn't explore an alternative: **IoT Core Rules Engine**. A Rule on `draws/quickdraw` could forward messages to a Lambda or DynamoDB with full payload context, giving you true end-to-end tracking without depending on CloudWatch Logs Insights at all. The CloudWatch Insights approach has the query latency, the truncation problem (F05), and the payload-correlation gap that FINDINGS.md calls out. I'm not saying it's the wrong call — it may be out of scope — but if delivery confirmation is the production requirement, there may be a cleaner architecture than the one being evaluated, and it's worth a note.

---

*Review complete. Happy to dig deeper on any finding or defend/revise any severity call.*
