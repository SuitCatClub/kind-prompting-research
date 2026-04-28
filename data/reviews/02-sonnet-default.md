Agent completed. agent_id: sonnet-default-review, agent_type: explore, status: completed, description: Sonnet 4.6 default code review, elapsed: 226s, total_turns: 0, model: claude-sonnet-4.6, duration: 222s

Now I have a thorough picture of the entire codebase. Here is the full report.

---

# AWS IoT Core POC — Code Review Report

**Scope:** All Python scripts, CloudFormation template, deploy script, and planning/documentation files
**Date of review:** Based on codebase state as of 2026-04-09
**Project maturity:** Completed POC / proof-of-concept — all Phase 1 experiments run and findings documented

---

## Table of Contents

1. [Security Issues](#1-security-issues)
2. [Bugs & Correctness Issues](#2-bugs--correctness-issues)
3. [Architectural & Design Issues](#3-architectural--design-issues)
4. [Code Quality & Robustness Improvements](#4-code-quality--robustness-improvements)
5. [Infrastructure (CloudFormation)](#5-infrastructure-cloudformation)
6. [Documentation & Planning Observations](#6-documentation--planning-observations)
7. [Summary Table](#7-summary-table)

---

## 1. Security Issues

### 🔴 HIGH — All Unauthenticated Identities Can Publish to the Draw Topic

**File:** `infra/template.yaml`, lines 63–66
```yaml
- Effect: Allow
  Action: iot:Publish
  Resource:
    - !Sub 'arn:aws:iot:${AWS::Region}:${AWS::AccountId}:topic/draws/quickdraw'
```
The single IAM role is assigned to **all** Cognito unauthenticated identities. This means every terminal (subscriber) also has `iot:Publish` permission on `draws/quickdraw`. In production, a compromised or misbehaving terminal could flood the draw topic with fabricated results. The publisher should use a completely separate authentication path (a different IAM role, a separate identity pool, or certificate-based auth), and terminal roles should only have `iot:Subscribe` + `iot:Receive`.

---

### 🔴 HIGH — IoT Core Logging at DEBUG Level Captures Full Message Payloads

**File:** `infra/template.yaml`, line 97
```yaml
DefaultLogLevel: DEBUG
```
DEBUG logging captures message content in CloudWatch. For a lottery system, draw results are commercially sensitive. If this template is ever promoted toward production, DEBUG must be reduced to `WARN` or `ERROR`. There is no explicit callout in README or FINDINGS warning against this for production use.

---

### 🟡 MEDIUM — Unauthenticated Cognito Identity Pool — Any AWS Account Can Obtain Credentials

**File:** `infra/template.yaml`, line 15
```yaml
AllowUnauthenticatedIdentities: true
```
There is no restriction on which AWS accounts/principals can call `GetId` against this pool. Anyone who can enumerate the Identity Pool ID (it appears in CloudFormation stack outputs, printed to the terminal, and stored in `.env`) can obtain temporary AWS credentials scoped to the `UnauthRole`. While the IAM permissions are restricted to IoT operations, this is still a meaningful exposure. For production, authenticated identities with Cognito User Pools or a custom authorizer should be used.

---

### 🟡 MEDIUM — Potential Query Injection in `cmd_trace`

**File:** `scripts/query_logs.py`, lines 64–68
```python
query = (
    "fields @timestamp, eventType, clientId, traceId, topicName, status\n"
    f"| filter traceId = '{args.trace_id}'\n"
    "| sort @timestamp asc"
)
```
The `--trace-id` CLI argument is interpolated directly into the CloudWatch Logs Insights query string with no sanitization. A crafted input like `x' | fields secretField` could alter query behavior. While this is a local developer tool, it is worth sanitizing with a UUID regex check:
```python
import re
if not re.fullmatch(r'[0-9a-f\-]{36}', args.trace_id):
    raise ValueError(f"Invalid trace_id format: {args.trace_id}")
```

---

### 🟡 MEDIUM — `IoTLoggingRole` CloudWatch Policy Uses `Resource: '*'`

**File:** `infra/template.yaml`, lines 86–92
```yaml
Resource: '*'
```
The IoT logging role can write to **any** CloudWatch log group in the account, not just `AWSIotLogsV2`. This should be scoped to the specific ARN:
```yaml
Resource: !Sub 'arn:aws:logs:${AWS::Region}:${AWS::AccountId}:log-group:AWSIotLogsV2:*'
```

---

## 2. Bugs & Correctness Issues

### 🔴 HIGH — `session_present` Receives a Dict, Not a Bool

**File:** `scripts/subscriber.py`, lines 59–61, 112–114, 155–157
```python
connect_future = mqtt_connection.connect()
session_present = connect_future.result()
print(f"Connected as '{args.client_id}', session_present={session_present}")
```
The `awsiotsdk` `connect()` Future resolves to a **dict** `{'session_present': bool}`, not to a bare boolean. `session_present` here is the entire dict. The print output will be `session_present={'session_present': True}` (misleading but not crashing). More importantly, in the `--reconnect-test` flow, `session_present` is used purely for printing — but if any downstream code ever added `if session_present:` it would always be truthy (any non-empty dict is truthy in Python).

**Fix:**
```python
result = connect_future.result()
session_present = result.get('session_present', False)
```
This applies to all three connect calls in `subscriber.py`.

---

### 🔴 HIGH — No Exception Handling on Any Async Future `.result()` Calls

**Files:** `scripts/publisher.py` (lines 25, 48), `scripts/subscriber.py` (lines 59, 68–69, 88, 112, 155–156, 163–165, 177)

Every `future.result()` call — for `connect()`, `subscribe()`, `publish()`, `disconnect()` — is unguarded. A transient network failure, auth error, or IoT Core throttle will cause an unhandled exception with a raw traceback. The `disconnect()` calls are especially dangerous: if a preceding operation raises, the disconnect is never reached, leaving a dangling MQTT connection.

**Minimum recommended pattern:**
```python
try:
    connect_future.result()
except Exception as e:
    print(f"Connection failed: {e}")
    sys.exit(1)
```
And use a `try/finally` to ensure `disconnect()` is always called.

---

### 🟡 MEDIUM — `on_message_received` Has No Error Handling for Malformed Payloads

**File:** `scripts/subscriber.py`, lines 43–46
```python
def on_message_received(topic, payload, dup, qos, retain, **kwargs):
    message = json.loads(payload)
    ...
```
`json.loads()` raises `json.JSONDecodeError` if the payload is not valid JSON. This callback executes on the SDK's background CRT thread. An unhandled exception here may silently crash the background thread, causing the subscriber to stop processing messages without any visible error. A `try/except` with a fallback print is essential:
```python
try:
    message = json.loads(payload)
except json.JSONDecodeError as e:
    print(f"[{ts}] {topic}: received non-JSON payload: {payload!r} ({e})")
    return
```

---

### 🟡 MEDIUM — `draw_id` Is Insufficiently Unique

**File:** `scripts/publisher.py`, line 36
```python
'draw_id': f'draw-{int(time.time())}',
```
`time.time()` has second-level precision. If the user presses `y` twice within the same second (easy in an interactive loop), two messages get the same `draw_id`. Since FINDINGS.md recommends `draw_id` as the primary correlation key for production delivery tracking, uniqueness is critical. Use `uuid.uuid4()`:
```python
import uuid
'draw_id': str(uuid.uuid4()),
```
Or at minimum `time.time_ns()` for nanosecond uniqueness.

---

### 🟡 MEDIUM — `cmd_delivery` Query Has a Redundant `fields` Clause

**File:** `scripts/query_logs.py`, lines 82–87
```python
query = (
    "fields @timestamp, clientId, status, traceId\n"
    "| filter eventType = 'Publish-Out'\n"
    f"| filter topicName = '{config.TOPIC}'\n"
    "| stats count(*) as deliveries by clientId, status\n"
    "| sort clientId"
)
```
In CloudWatch Logs Insights, when you use a `stats` aggregation, any preceding `fields` projection is ignored — the `stats` command produces its own output columns. The `fields @timestamp, clientId, status, traceId` line is a no-op here and may confuse future maintainers. It should be removed:
```
| filter eventType = 'Publish-Out'
| filter topicName = 'draws/quickdraw'
| stats count(*) as deliveries by clientId, status
| sort clientId
```

---

### 🟢 LOW — `cmd_latest` Parses `@ptr` Fields Into Its Dict

**File:** `scripts/query_logs.py`, lines 107–108
```python
fields = {item['field']: item['value'] for item in results[0]}
trace_id = fields.get('traceId', 'N/A')
```
Unlike `format_results()` which filters `@ptr` fields with `if not item['field'].startswith('@ptr')`, `cmd_latest` includes the internal CloudWatch `@ptr` pointer in its dict. This is harmless but inconsistent — future accidental use of `fields` for display would show the pointer. Apply the same filter as `format_results`.

---

### 🟢 LOW — `run_query` Polling Loop Has No Timeout

**File:** `scripts/query_logs.py`, lines 36–41
```python
while True:
    time.sleep(1)
    result = client.get_query_results(queryId=query_id)
    if result['status'] in ('Complete', 'Failed', 'Cancelled', 'Timeout', 'Unknown'):
        break
```
While CloudWatch Insights queries rarely hang, a transient AWS API issue returning an unexpected status string would cause an infinite loop. Add a maximum iteration count:
```python
for _ in range(120):  # max 2 minutes
    time.sleep(1)
    ...
    if result['status'] in (...):
        break
else:
    raise RuntimeError("Query timed out after 120 seconds")
```

---

## 3. Architectural & Design Issues

### 🔴 HIGH — Publisher `client_id` Is Hard-Coded

**File:** `scripts/publisher.py`, line 17
```python
client_id='publisher-tpm',
```
The publisher has no CLI argument for `--client-id`. The value `publisher-tpm` is hard-coded and must match the IAM policy `client/publisher-tpm` resource ARN exactly. If you ever want to run multiple publisher instances, test with a different publisher ID, or rename the client, you must edit the source file **and** the CloudFormation template. The subscriber correctly uses `--client-id` via argparse; the publisher should do the same (defaulting to `publisher-tpm` for backward compatibility).

---

### 🟡 MEDIUM — Significant Code Duplication in `subscriber.py` Test Flows

**File:** `scripts/subscriber.py`, lines 83–167

The `--reconnect-test` (lines 83–125) and `--refresh-test` (lines 127–167) blocks are nearly identical: both disconnect, obtain fresh credentials with `cached_identity_id`, create a new `mqtt_connection`, call `connect()`, and re-subscribe. The only semantic difference is what they print. This could be extracted into a helper:

```python
def reconnect_with_fresh_credentials(client_id, identity_id, callbacks, clean_session=False):
    credentials = get_cognito_credentials(config.IDENTITY_POOL_ID, config.AWS_REGION, identity_id=identity_id)
    connection = create_mqtt_connection(...)
    session_present = connection.connect().result().get('session_present', False)
    return connection, credentials, session_present
```

---

### 🟡 MEDIUM — `config.py` Raises Bare `KeyError` on Missing Env Vars

**File:** `scripts/config.py`, lines 10–11
```python
IOT_ENDPOINT = os.environ['IOT_ENDPOINT']
IDENTITY_POOL_ID = os.environ['IDENTITY_POOL_ID']
```
If a user forgets to copy `.env.example` to `.env` or leaves a value blank, the script crashes with `KeyError: 'IOT_ENDPOINT'` — no guidance on what to do. A more helpful pattern:
```python
def _require_env(key: str) -> str:
    val = os.environ.get(key)
    if not val:
        raise RuntimeError(f"Missing required env var '{key}'. Copy .env.example to .env and fill in values.")
    return val

IOT_ENDPOINT = _require_env('IOT_ENDPOINT')
IDENTITY_POOL_ID = _require_env('IDENTITY_POOL_ID')
```

---

### 🟡 MEDIUM — `draw_id` Is the Recommended Correlation Key, But It Isn't Logged on Receipt

**File:** `scripts/subscriber.py`, lines 43–46; `FINDINGS.md` lines 184–188

FINDINGS.md recommends `draw_id` payload-based tracking as the production solution. However, the `on_message_received` callback just pretty-prints the full payload. For the POC to validate the ACK topic pattern properly, the callback should log `draw_id` explicitly and demonstrate the ACK pattern (even as a stub), rather than leaving it as documentation only:
```python
draw_id = message.get('draw_id', 'unknown')
print(f"[{ts}] Received draw_id={draw_id} on {topic}")
# Stub: mqtt_connection.publish(f"draws/quickdraw/ack/{client_id}", ...)
```

---

### 🟢 LOW — `boto3` Not Version-Pinned in `requirements.txt`

**File:** `requirements.txt`, line 2
```
boto3>=1.42.0
```
`awsiotsdk==1.28.2` is pinned, but `boto3` is only lower-bounded. A future `boto3` major version bump could silently break auth behavior. For reproducible POC environments, pin to a specific version or range:
```
boto3>=1.42.0,<2.0.0
```

---

### 🟢 LOW — `awscrt` Is a Direct Import But Not an Explicit Dependency

**File:** `scripts/auth.py` line 3, `scripts/publisher.py` line 5, `scripts/subscriber.py` line 8
```python
from awscrt import mqtt
from awscrt.auth import AwsCredentialsProvider
```
`awscrt` is imported directly but is only a transitive dependency of `awsiotsdk`. If `awsiotsdk` ever changed its packaging, `awscrt` could stop being installed automatically. Add it explicitly:
```
awscrt>=0.22.0
```

---

## 4. Code Quality & Robustness Improvements

### 🟡 MEDIUM — `get_cognito_credentials` Creates a New `boto3` Client on Every Call

**File:** `scripts/auth.py`, line 21
```python
client = boto3.client('cognito-identity', region_name=region)
```
Every call to `get_cognito_credentials()` (including every credential refresh) creates a new boto3 client and a new HTTPS connection. For a POC this is fine, but for the production credential-refresh flow running against 5,000 terminals, client reuse matters. Accept an optional `client` parameter or cache at module level.

---

### 🟡 MEDIUM — No Retry Logic for Transient AWS API Errors

Neither `auth.py` nor `query_logs.py` handles transient Cognito/CloudWatch errors (e.g., `ThrottlingException`, `ServiceUnavailableException`). For a POC these are rare, but the `query_logs.py` tool is likely to be run repeatedly during experiments. A simple exponential backoff wrapper or use of `botocore`'s built-in retry config would make the tool more reliable:
```python
from botocore.config import Config
client = boto3.client('logs', region_name=region, config=Config(retries={'max_attempts': 5, 'mode': 'standard'}))
```

---

### 🟡 MEDIUM — Publisher Has No Graceful Exit on Exception

**File:** `scripts/publisher.py`, lines 24–52

If `connect_future.result()` raises, the script exits without calling `disconnect()`. While the broker will eventually time out the connection, this leaves a stale persistent connection entry on IoT Core. The main block should be wrapped in `try/finally`:
```python
connect_future.result()
try:
    # main loop
finally:
    mqtt_connection.disconnect().result()
```

---

### 🟢 LOW — Static Test Payload Numbers Are Never Randomized

**File:** `scripts/publisher.py`, lines 37–38
```python
'numbers': [7, 14, 21, 35, 42],
```
The lottery numbers are always identical across all test publishes. While it doesn't affect correctness for the POC, randomizing them would make CloudWatch log analysis less ambiguous (you can distinguish which payload corresponds to which Publish-In event by content, not just timestamp):
```python
import random
'numbers': random.sample(range(1, 81), 20),  # Quick Draw uses 1-80
```

---

### 🟢 LOW — `subscriber.py` Doesn't Expose `clean_session` Status in Print Output

**File:** `scripts/subscriber.py`, line 61
```python
print(f"Connected as '{args.client_id}', session_present={session_present}")
```
It would be useful to also print `clean_session={args.clean_session}` at startup since this is the primary variable affecting persistent session behavior and is easy to forget which mode you're in.

---

### 🟢 LOW — No `__init__.py` in `scripts/` — Relative Imports Are Implicit

**Files:** `scripts/publisher.py` line 6, `scripts/subscriber.py` line 9
```python
from auth import get_cognito_credentials, create_mqtt_connection
import config
```
These are bare-name imports that only work because the scripts are run from the `scripts/` directory (via `cd scripts && python publisher.py`). If run from the project root (`python scripts/publisher.py`), they fail with `ModuleNotFoundError`. Adding `scripts/__init__.py` and adjusting to relative imports, or using `sys.path` insertion at the top of each script, would make them location-independent:
```python
import sys, os
sys.path.insert(0, os.path.dirname(__file__))
```

---

## 5. Infrastructure (CloudFormation)

### 🟡 MEDIUM — No CloudWatch Log Retention Policy on `AWSIotLogsV2`

**File:** `infra/template.yaml`

`AWSIotLogsV2` is automatically created by IoT Core when the `AWS::IoT::Logging` resource is provisioned. Without a retention policy, logs are retained **indefinitely** at standard CloudWatch pricing. For a POC this accumulates cost quietly. Add a `AWS::Logs::LogGroup` resource to set retention:
```yaml
IotLogGroup:
  Type: AWS::Logs::LogGroup
  Properties:
    LogGroupName: AWSIotLogsV2
    RetentionInDays: 7
```
Note: CloudFormation can manage this group if it doesn't already exist.

---

### 🟡 MEDIUM — IAM `RoleName` Is Hard-Coded to `ProjectName`

**File:** `infra/template.yaml`, lines 27, 72
```yaml
RoleName: !Sub '${ProjectName}-unauth-role'
```
Hard-coded role names in CloudFormation conflict if the same template is deployed to multiple regions or accounts under the same name. Consider removing `RoleName` entirely and letting CloudFormation auto-generate names, or scope them with the account/region:
```yaml
RoleName: !Sub '${ProjectName}-unauth-role-${AWS::Region}'
```

---

### 🟢 LOW — `deploy.sh` Doesn't Validate Prerequisites

**File:** `infra/deploy.sh`

The script assumes AWS CLI v2, proper `~/.aws` credentials, and `bash` are all available. It would be more robust with a preflight check:
```bash
command -v aws >/dev/null 2>&1 || { echo "AWS CLI not found"; exit 1; }
aws sts get-caller-identity >/dev/null 2>&1 || { echo "AWS credentials not configured"; exit 1; }
```

---

### 🟢 LOW — No `--no-fail-on-empty-changeset` in `deploy.sh`

**File:** `infra/deploy.sh`, line 6–10

`aws cloudformation deploy` exits with a non-zero code if there are no changes to deploy. With `set -euo pipefail`, running `deploy.sh` a second time on an unchanged template will fail the script. Add the flag:
```bash
aws cloudformation deploy \
  --template-file "$(dirname "$0")/template.yaml" \
  --stack-name "$STACK_NAME" \
  --capabilities CAPABILITY_NAMED_IAM \
  --region "$REGION" \
  --no-fail-on-empty-changeset
```

---

## 6. Documentation & Planning Observations

### 🟡 MEDIUM — `REQUIREMENTS.md` LOGS-03 Status Is Misleading

**File:** `.planning/REQUIREMENTS.md`, line 33
```markdown
- [x] **LOGS-03**: traceId correlates a single Publish-In to all corresponding Publish-Out entries
```
LOGS-03 is marked `[x]` (complete/verified) but the actual finding documented in `FINDINGS.md` is the **opposite**: traceId does **not** correlate Publish-In to Publish-Out. Each Publish-Out gets its own unique traceId. The checkbox status implies the requirement was satisfied when it was in fact **refuted**. The status should be `[x] INVALIDATED` or the requirement reworded to "Determine traceId correlation behavior between Publish-In and Publish-Out" (which was the real intent).

---

### 🟡 MEDIUM — README Experiment 2 Instructions Use `iptables` (Linux-Only)

**File:** `README.md`, lines 134–151

The frozen subscriber experiment relies on `iptables`, `ss`, and `SIGSTOP`/`SIGCONT` — all Linux-specific commands. The README gives no guidance for macOS (`pfctl`/`pf`) or Windows. Since the toolchain uses Python and the project directory is under a Windows workspace path (`C:\Users\dev\...`), the developer may be running scripts on WSL or a remote Linux box, but this is never stated. Adding a note like "Experiment 2 requires a Linux environment (bare metal, VM, or WSL2)" would prevent confusion.

---

### 🟢 LOW — `STATE.md` Progress Percentage Is Inconsistent

**File:** `.planning/STATE.md`, lines 11–15
```yaml
completed_plans: 4
percent: 75
```
`completed_plans: 4` out of 4 total plans should be 100%, but `percent: 75` is shown. ROADMAP.md also shows `3/4` plans complete for Phase 1. One of these is stale and was not updated after the final plan was completed. The `stopped_at` field says "Completed 01-04-PLAN.md" which implies all 4 plans are done, but the percentage and ROADMAP table weren't updated.

---

### 🟢 LOW — `.planning/` Is Tracked in Git Without `.gitignore` Exclusion

**File:** `.gitignore`

The `.planning/` directory contains internal project planning artifacts (ROADMAP, STATE with velocity metrics, etc.) that are likely not intended for external consumers of this repo. Consider adding `.planning/` to `.gitignore` or moving it to a non-tracked location, depending on whether these docs are considered part of the shareable artifact.

---

### 🟢 LOW — `CRED-01` Is Marked Validated But Not Formally Tested

**File:** `FINDINGS.md`, lines 166–168
> **Experiment 4: Credential Refresh — Status: Not yet tested separately.**

`REQUIREMENTS.md` marks `CRED-01` as `[x] Complete`, but FINDINGS documents it as only "implicitly validated." For a production decision, implicit validation of the credential refresh cycle is insufficient — the `--refresh-test` experiment should be run explicitly and the findings documented. This is especially true since the `awsiotsdk` credential refresh behavior (whether the SDK auto-refreshes credentials on the existing connection vs. requiring a new connection) is a nuance worth validating.

---

## 7. Summary Table

| # | File(s) | Severity | Issue |
|---|---------|----------|-------|
| 1 | `template.yaml` | 🔴 HIGH | Terminals have `iot:Publish` on draw topic — should be publisher-only |
| 2 | `template.yaml` | 🔴 HIGH | DEBUG log level captures message content — production risk |
| 3 | `template.yaml` | 🟡 MED | `AllowUnauthenticatedIdentities: true` with no access restriction |
| 4 | `query_logs.py` | 🟡 MED | `trace_id` injected into query string without sanitization |
| 5 | `template.yaml` | 🟡 MED | `IoTLoggingRole` CloudWatch policy scoped to `Resource: '*'` |
| 6 | `subscriber.py` | 🔴 HIGH | `connect_future.result()` returns dict, not bool — `session_present` is always a dict |
| 7 | `publisher.py`, `subscriber.py` | 🔴 HIGH | No exception handling on any `.result()` calls; no `finally` for disconnect |
| 8 | `subscriber.py` | 🔴 HIGH | `json.loads()` in message callback unguarded — crashes CRT background thread on bad payload |
| 9 | `publisher.py` | 🟡 MED | `draw_id` uses second-precision timestamp — not unique within same second |
| 10 | `query_logs.py` | 🟡 MED | `cmd_delivery` has a no-op `fields` clause before a `stats` aggregation |
| 11 | `query_logs.py` | 🟢 LOW | `cmd_latest` includes `@ptr` fields in dict unlike `format_results` |
| 12 | `query_logs.py` | 🟢 LOW | `run_query` polling loop has no timeout |
| 13 | `publisher.py` | 🔴 HIGH | `client_id='publisher-tpm'` is hard-coded, not CLI-configurable |
| 14 | `subscriber.py` | 🟡 MED | Reconnect + refresh test blocks are near-identical (DRY violation) |
| 15 | `config.py` | 🟡 MED | `os.environ['KEY']` raises unhelpful `KeyError` on missing env vars |
| 16 | `subscriber.py`, `publisher.py` | 🟡 MED | `draw_id` is recommended production key but not logged explicitly on receipt |
| 17 | `requirements.txt` | 🟢 LOW | `boto3` not version-pinned; `awscrt` not listed as explicit dependency |
| 18 | `auth.py` | 🟡 MED | New boto3 client created on every credential refresh call |
| 19 | `publisher.py`, `subscriber.py` | 🟡 MED | No retry logic for transient AWS API errors |
| 20 | `publisher.py` | 🟡 MED | No `try/finally` — disconnect not called if exception is raised |
| 21 | `publisher.py` | 🟢 LOW | Lottery numbers always `[7,14,21,35,42]` — harder to distinguish events in logs |
| 22 | `scripts/` | 🟢 LOW | Bare-name imports fail when scripts run from project root |
| 23 | `template.yaml` | 🟡 MED | No CloudWatch log retention policy — unbounded cost |
| 24 | `template.yaml` | 🟡 MED | Hard-coded `RoleName` conflicts on multi-region deployments |
| 25 | `deploy.sh` | 🟢 LOW | No prerequisite validation (AWS CLI, credentials) |
| 26 | `deploy.sh` | 🟢 LOW | Missing `--no-fail-on-empty-changeset` — re-running on unchanged stack exits non-zero |
| 27 | `REQUIREMENTS.md` | 🟡 MED | LOGS-03 marked `[x]` but the finding **refutes** the requirement |
| 28 | `README.md` | 🟡 MED | Experiment 2 is Linux-only (`iptables`/`ss`) with no platform note |
| 29 | `STATE.md` | 🟢 LOW | `percent: 75` inconsistent with `completed_plans: 4/4` |
| 30 | `FINDINGS.md` | 🟢 LOW | CRED-01 marked complete but only implicitly validated — not formally tested |

---

## Overall Assessment

The POC is **well-structured and achieves its stated goal** — it produced clear, reproducible empirical evidence on Publish-Out semantics, traceId correlation, persistent sessions, and QoS 1 redelivery. The planning and findings documentation is detailed and honest about limitations. The infrastructure is deliberately minimal and appropriate for the POC scope.

The most important issues to fix **before any production adaptation** are:

1. **Separate publisher and terminal IAM roles** — terminals must not be able to publish to the draw topic
2. **Fix `session_present` dict vs. bool** — silent data correctness bug in all three connect paths
3. **Add exception handling to all future `.result()` calls** — unguarded futures will silently crash in any real error condition
4. **Add error handling to the message callback** — an exception in the CRT background thread can silently kill message processing
5. **Move to authenticated Cognito identities** — unauthenticated pool with no access control is not suitable for a production lottery system