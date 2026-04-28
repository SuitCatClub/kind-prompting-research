# Security and Production Readiness Audit

Scope reviewed: `scripts/auth.py`, `scripts/config.py`, `scripts/publisher.py`, `scripts/subscriber.py`, `scripts/query_logs.py`, `infra/template.yaml`, `infra/deploy.sh`, `requirements.txt`, `README.md`, `.gitignore`, `.env.example`, `FINDINGS.md`.

Method: static review of every provided file and verification of claims in `FINDINGS.md`. No automated tests were present in this repo to run.

## Executive Summary

This POC is **not production-safe** for a lottery system. The dominant risk is an end-to-end anonymous publish chain:

1. `FINDINGS.md` exposes a real Identity Pool ID.
2. `infra/template.yaml` enables unauthenticated Cognito identities.
3. The unauthenticated IAM role can connect, publish, subscribe, and receive on the production draw topic.

That combination allows unauthorized parties to impersonate terminals, inject draw messages, and disrupt legitimate clients. In addition, multiple findings documents overstate delivery assurance that the experiments themselves disproved.

## Findings

### 1. CRITICAL — Real Cognito Identity Pool ID committed to the repository
- **File:** `FINDINGS.md`
- **Line:** 194
- **Description:** A real Identity Pool ID is embedded in the findings table.
- **Impact:** Anyone who can read the repo can target the live identity pool. Combined with findings 2 and 3, this becomes a direct unauthorized-access path to AWS IoT Core.
- **Fix:** Delete/rotate the exposed stack immediately and purge the ID from git history.

### 2. CRITICAL — Cognito pool allows unauthenticated identities
- **File:** `infra/template.yaml`
- **Line:** 15
- **Description:** `AllowUnauthenticatedIdentities: true` enables anonymous credential issuance.
- **Impact:** Unauthenticated users can obtain AWS credentials without proving identity.
- **Fix:** Set this to `false` for production and require authenticated identities or device certificates.

### 3. CRITICAL — Anonymous role can publish to the draw topic
- **File:** `infra/template.yaml`
- **Lines:** 63-66
- **Description:** The unauthenticated role is allowed `iot:Publish` on `topic/draws/quickdraw`.
- **Impact:** Any anonymous party can inject fake draw results to all subscribed terminals.
- **Fix:** Remove publish from the unauthenticated role; reserve publish for a dedicated authenticated publisher identity.

### 4. HIGH — Anonymous role can subscribe and receive draw traffic
- **File:** `infra/template.yaml`
- **Lines:** 53-61
- **Description:** The unauthenticated role can `iot:Subscribe` and `iot:Receive` on the draw topic.
- **Impact:** Unauthorized users can observe draw traffic and any future sensitive payloads on this topic.
- **Fix:** Restrict subscribe/receive to authenticated terminal identities only.

### 5. HIGH — Anonymous role can connect as the official publisher client ID
- **File:** `infra/template.yaml`
- **Lines:** 47-51
- **Description:** The connect policy explicitly permits `client/publisher-tpm` under the unauthenticated role.
- **Impact:** Attackers can impersonate the official publisher and make malicious publishes look legitimate in logs.
- **Fix:** Remove `publisher-tpm` from the anonymous role and use a separate publisher-only principal.

### 6. HIGH — Wildcard `terminal-*` client IDs permit terminal impersonation and session takeover
- **File:** `infra/template.yaml`
- **Line:** 50
- **Description:** Any anonymous client that knows the naming pattern can connect as `terminal-001`, `terminal-002`, etc.
- **Impact:** MQTT client-ID collisions let an attacker disconnect or replace a legitimate terminal session.
- **Fix:** Bind client IDs to authenticated principals or device certificates; do not rely on a naming convention alone.

### 7. HIGH — No authenticated role separation exists in the identity pool attachment
- **File:** `infra/template.yaml`
- **Lines:** 17-22
- **Description:** Only an `unauthenticated` role is attached; there is no authenticated path.
- **Impact:** The design has no principle-of-least-privilege separation between publisher, subscriber, and anonymous users.
- **Fix:** Add authenticated roles with distinct least-privilege policies, or move to IoT certificate-based auth.

### 8. HIGH — IoT logging is set to account-wide DEBUG
- **File:** `infra/template.yaml`
- **Lines:** 94-99
- **Description:** `AWS::IoT::Logging` with `DefaultLogLevel: DEBUG` changes the account-level default.
- **Impact:** High log volume, cost blow-up, and wider metadata exposure across all IoT workloads in the account.
- **Fix:** Use `ERROR` or `WARN` in production and enable narrower diagnostic logging only when needed.

### 9. HIGH — CloudWatch logs role uses `Resource: '*'`
- **File:** `infra/template.yaml`
- **Lines:** 85-92
- **Description:** The IoT logging role can write to any CloudWatch log group.
- **Impact:** This violates least privilege and expands blast radius if the role is abused or misconfigured.
- **Fix:** Scope the policy to the specific IoT log group ARN(s).

### 10. MEDIUM — No explicit log group or retention policy is defined
- **File:** `infra/template.yaml`
- **Lines:** 68-99
- **Description:** Logging is enabled, but no `AWS::Logs::LogGroup` with `RetentionInDays` is created.
- **Impact:** Logs may accumulate indefinitely, increasing cost and retention of operational metadata.
- **Fix:** Create the log group explicitly and set retention appropriate for the environment.

### 11. MEDIUM — Cognito API failures are unhandled
- **File:** `scripts/auth.py`
- **Lines:** 21-34
- **Description:** `get_id()` and `get_credentials_for_identity()` are called without retries or `ClientError` handling.
- **Impact:** Transient throttling, misconfiguration, or auth errors crash the process with raw stack traces.
- **Fix:** Catch `botocore.exceptions.ClientError`, retry transient failures, and emit actionable errors.

### 12. HIGH — Static credentials provider has no automatic refresh path
- **File:** `scripts/auth.py`
- **Lines:** 68-72
- **Description:** `AwsCredentialsProvider.new_static(...)` freezes the current short-lived STS credentials into the connection.
- **Impact:** Long-lived clients cannot transparently refresh expiring credentials.
- **Fix:** Use a refresh-capable credentials provider or implement proactive refresh/reconnect before expiration.

### 13. MEDIUM — Missing required environment variables crash at import time
- **File:** `scripts/config.py`
- **Lines:** 10-11
- **Description:** `os.environ[...]` raises `KeyError` immediately when variables are absent.
- **Impact:** Every script dies during import with poor operator guidance.
- **Fix:** Validate required variables explicitly and raise a clear configuration error.

### 14. MEDIUM — Silent fallback to `us-east-1`
- **File:** `scripts/config.py`
- **Line:** 12
- **Description:** `AWS_REGION` defaults silently instead of being required.
- **Impact:** Scripts can query or connect to the wrong region and fail in confusing ways.
- **Fix:** Require `AWS_REGION` explicitly or at minimum log a warning when defaulting.

### 15. LOW — Topic is hardcoded in source
- **File:** `scripts/config.py`
- **Line:** 15
- **Description:** The draw topic is not configurable via environment.
- **Impact:** Environment changes require code changes, increasing deployment error risk.
- **Fix:** Load the topic from configuration with a safe default.

### 16. HIGH — Publisher client ID is hardcoded to a globally known value
- **File:** `scripts/publisher.py`
- **Line:** 20
- **Description:** The publisher always connects as `publisher-tpm`.
- **Impact:** If another client connects with the same ID, the legitimate publisher can be disconnected.
- **Fix:** Bind the publisher to a dedicated authenticated identity and do not expose the client ID as a public constant.

### 17. HIGH — Published messages carry no authenticity signal or integrity protection
- **File:** `scripts/publisher.py`
- **Lines:** 35-40
- **Description:** The payload is plain JSON with no signature, MAC, or versioned schema.
- **Impact:** Subscribers cannot distinguish legitimate draw messages from forged ones if broker auth is bypassed or misconfigured.
- **Fix:** Add message signing or a server-side ACK/correlation design with a trusted publisher identity.

### 18. MEDIUM — `draw_id` is timestamp-based and collision-prone
- **File:** `scripts/publisher.py`
- **Line:** 36
- **Description:** `draw_id` uses `int(time.time())`, which has one-second resolution.
- **Impact:** Multiple publishes within a second can collide, undermining correlation, replay detection, and auditability.
- **Fix:** Use UUIDs or a monotonic draw sequence assigned by the source system.

### 19. LOW — Connect/publish waits have no timeout
- **File:** `scripts/publisher.py`
- **Lines:** 24-25, 43-48
- **Description:** `.result()` is called with no timeout.
- **Impact:** Network stalls can hang the publisher indefinitely.
- **Fix:** Use explicit timeouts and handle timeout exceptions.

### 20. MEDIUM — Publisher does not guarantee disconnect cleanup on failure
- **File:** `scripts/publisher.py`
- **Lines:** 24-52
- **Description:** Exceptions inside the loop bypass the final disconnect.
- **Impact:** Dirty shutdowns complicate diagnostics and can leave unexpected server-side session state.
- **Fix:** Wrap the main connection lifecycle in `try/finally`.

### 21. HIGH — Subscriber accepts arbitrary client IDs from the command line
- **File:** `scripts/subscriber.py`
- **Line:** 15
- **Description:** `--client-id` is required but unconstrained.
- **Impact:** In combination with the wildcard IoT policy, users can impersonate arbitrary terminals.
- **Fix:** Validate allowed client-ID format locally and enforce identity-bound client IDs in AWS.

### 22. MEDIUM — Subscriber prints Cognito identity IDs to stdout
- **File:** `scripts/subscriber.py`
- **Line:** 33
- **Description:** Internal identity IDs are emitted to the console.
- **Impact:** These values can leak into shell history, log collectors, or support screenshots.
- **Fix:** Remove or downgrade this to debug-only logging.

### 23. HIGH — Uncaught JSON parse errors in the message callback
- **File:** `scripts/subscriber.py`
- **Lines:** 43-46
- **Description:** `json.loads(payload)` is unguarded.
- **Impact:** A malformed payload can break message handling and create a remote denial-of-service against subscribers.
- **Fix:** Catch decode errors and reject malformed payloads safely.

### 24. HIGH — Subscriber performs no payload schema validation
- **File:** `scripts/subscriber.py`
- **Lines:** 43-46
- **Description:** The callback trusts that every message has the expected shape and semantics.
- **Impact:** Invalid or malicious fields can propagate to downstream logic, operator screens, or audits.
- **Fix:** Validate required keys, types, ranges, and schema version before accepting the message.

### 25. MEDIUM — Subscriber prints untrusted payloads directly to the terminal
- **File:** `scripts/subscriber.py`
- **Line:** 46
- **Description:** Received content is rendered without sanitization.
- **Impact:** Payloads containing ANSI escape sequences can manipulate terminals or poison operator logs.
- **Fix:** Escape control characters or emit structured logs instead of raw terminal output.

### 26. HIGH — Reconnect callback does not restore subscriptions when the session is lost
- **File:** `scripts/subscriber.py`
- **Lines:** 39-40
- **Description:** `on_connection_resumed` only logs `session_present`; it never re-subscribes when `session_present` is false.
- **Impact:** After a resumed connection with no preserved session, the subscriber can appear healthy while receiving nothing.
- **Fix:** In the resume callback, re-subscribe when `session_present` is false.

### 27. MEDIUM — Test harness flags are shipped in the production subscriber entry point
- **File:** `scripts/subscriber.py`
- **Lines:** 21-27
- **Description:** `--freeze-after-subscribe`, `--reconnect-test`, and `--refresh-test` are mixed into the runtime client.
- **Impact:** Operational misuse can deliberately stop message processing or alter connection behavior in production.
- **Fix:** Move destructive test modes to separate tooling.

### 28. MEDIUM — `SIGSTOP` test mode is not portable and crashes on Windows
- **File:** `scripts/subscriber.py`
- **Lines:** 72-78
- **Description:** The script uses POSIX-only `signal.SIGSTOP`.
- **Impact:** On Windows, this path fails instead of producing a controlled error.
- **Fix:** Guard by platform or remove this mode from the cross-platform subscriber.

### 29. HIGH — Normal subscriber mode never refreshes expiring credentials
- **File:** `scripts/subscriber.py`
- **Lines:** 169-177
- **Description:** The infinite loop only sleeps; refresh logic exists only in explicit test modes.
- **Impact:** After credential expiry, the next reconnect can fail and leave the terminal effectively dead.
- **Fix:** Track expiration and refresh/reconnect automatically before expiry.

### 30. HIGH — Logs Insights query injection via `--trace-id`
- **File:** `scripts/query_logs.py`
- **Lines:** 64-67
- **Description:** User input is interpolated directly into the query string.
- **Impact:** Attackers or misuse can alter the query shape, broaden output, and increase cost.
- **Fix:** Validate `trace_id` against a strict regex and avoid direct string interpolation.

### 31. HIGH — `raw` mode executes arbitrary Logs Insights queries
- **File:** `scripts/query_logs.py`
- **Lines:** 117-124, 157-160
- **Description:** The CLI intentionally runs any user-supplied query.
- **Impact:** If the runtime credentials are broader than intended, this becomes a log-exfiltration and cost-amplification tool.
- **Fix:** Remove `raw` from production tooling or tightly scope the IAM permissions available to this script.

### 32. MEDIUM — Query polling loop has no timeout or cancellation
- **File:** `scripts/query_logs.py`
- **Lines:** 36-40
- **Description:** The loop runs until a terminal status appears.
- **Impact:** Stuck queries can hang forever and consume query concurrency slots.
- **Fix:** Add a maximum poll duration, then cancel/abort the query.

### 33. MEDIUM — Boto3 errors in log querying are unhandled
- **File:** `scripts/query_logs.py`
- **Lines:** 25-45
- **Description:** `start_query()` and `get_query_results()` have no exception handling.
- **Impact:** Throttling, invalid queries, expired credentials, or missing log groups terminate the program abruptly.
- **Fix:** Catch `ClientError`, retry where appropriate, and emit operator-focused error messages.

### 34. MEDIUM — `--minutes` has no upper bound
- **File:** `scripts/query_logs.py`
- **Lines:** 131-134
- **Description:** Time window size is unconstrained.
- **Impact:** Very large windows can scan huge log volumes and generate unnecessary CloudWatch costs.
- **Fix:** Enforce a sane max window or require an explicit override for large scans.

### 35. LOW — Query result limit is hardcoded to 100
- **File:** `scripts/query_logs.py`
- **Line:** 12
- **Description:** Queries silently truncate after 100 results unless code changes.
- **Impact:** High-volume incidents can be partially hidden during audits.
- **Fix:** Expose `--limit` and make truncation explicit in the output.

### 36. MEDIUM — Native `awscrt` dependency is not pinned directly
- **File:** `requirements.txt`
- **Line:** 1
- **Description:** `awsiotsdk` is pinned, but the underlying native CRT dependency version is left to resolution.
- **Impact:** Different environments can end up with different TLS/runtime behavior and patch levels.
- **Fix:** Use a fully locked dependency set for deployment.

### 37. MEDIUM — `boto3` and `python-dotenv` have no upper bounds
- **File:** `requirements.txt`
- **Lines:** 2-3
- **Description:** Both dependencies float forward indefinitely.
- **Impact:** Future breaking changes can enter production unexpectedly.
- **Fix:** Add upper bounds and maintain a lock file.

### 38. MEDIUM — Dependency installs are not hash-pinned
- **File:** `requirements.txt`
- **Lines:** 1-3
- **Description:** No `--hash` verification is used.
- **Impact:** Supply-chain tampering or accidental resolver drift is harder to detect.
- **Fix:** Generate a fully hashed lock file and install with hash verification.

### 39. MEDIUM — Deploy script does not enable termination protection
- **File:** `infra/deploy.sh`
- **Lines:** 6-10
- **Description:** The stack is deployed without any follow-up protection.
- **Impact:** An accidental delete can remove the identity pool and related auth path for all clients.
- **Fix:** Enable CloudFormation termination protection after successful deploy.

### 40. MEDIUM — Fixed `ProjectName` causes IAM role-name collisions across stacks
- **File:** `infra/deploy.sh`
- **Lines:** 6-10
- **Related template lines:** `infra/template.yaml` 28, 72
- **Description:** The script never overrides `ProjectName`, while the template hardcodes IAM role names from it.
- **Impact:** Multiple deployments in the same account can fail or conflict on IAM role creation.
- **Fix:** Pass a unique `ProjectName` parameter per deployment or remove explicit IAM `RoleName` values.

### 41. LOW — Deploy script silently defaults to `us-east-1`
- **File:** `infra/deploy.sh`
- **Line:** 4
- **Description:** Region fallback happens with no warning.
- **Impact:** Engineers can deploy to the wrong region unintentionally.
- **Fix:** Require the region or print a loud warning before using the default.

### 42. LOW — README’s iptables experiment is unsafe for shared hosts and undocumented for non-Linux users
- **File:** `README.md`
- **Lines:** 120-152
- **Description:** The instructions modify outbound firewall rules directly without warning about host isolation or OS limitations.
- **Impact:** Operators can break unrelated traffic on shared machines or fail outright on Windows/macOS.
- **Fix:** State that this must run only on an isolated Linux test host and provide cleanup and platform caveats prominently.

### 43. LOW — README states a specific “6-8 minute” IAM propagation delay without authoritative basis
- **File:** `README.md`
- **Line:** 198
- **Description:** The document presents a precise timing claim as fact.
- **Impact:** This can mislead troubleshooting and operational expectations.
- **Fix:** Reference AWS’s documented behavior directly or soften the statement to an observed POC note.

### 44. MEDIUM — README conflates MQTT session expiry with Cognito credential expiry
- **File:** `README.md`
- **Lines:** 200-201
- **Description:** The adjacent notes can be read as a single behavior even though they are separate lifecycles.
- **Impact:** Operators may tune the wrong control and still lose connectivity or queued messages.
- **Fix:** Separate and clarify session-retention behavior versus STS credential TTL.

### 45. HIGH — `FINDINGS.md` claims “100% delivery confirmed” even though the experiments disproved that
- **File:** `FINDINGS.md`
- **Line:** 203
- **Description:** The requirement table marks delivery confirmation as passed based on Publish-Out count.
- **Impact:** This creates false assurance for stakeholders making production decisions.
- **Fix:** Mark this requirement failed or partial, and state explicitly that Publish-Out is dispatch-only.

### 46. MEDIUM — `FINDINGS.md` marks trace correlation as verified while stating it does not work
- **File:** `FINDINGS.md`
- **Line:** 206
- **Description:** `LOGS-03` is checked `[x]` even though the text says Publish-In and Publish-Out trace IDs differ.
- **Impact:** Readers or automated status parsing will treat a failed requirement as complete.
- **Fix:** Change the status to failed/partial and document the actual workaround.

### 47. MEDIUM — `FINDINGS.md` marks credential refresh verified even though it was “not yet tested separately”
- **File:** `FINDINGS.md`
- **Lines:** 168, 211
- **Description:** The narrative says the requirement was not directly tested, but the table marks it complete.
- **Impact:** This overstates production readiness of long-lived clients.
- **Fix:** Mark the requirement pending until a dedicated refresh test passes.

## Bottom Line

**Decision: NO-GO for production.**

Before this design can be considered safe for a financial/lottery environment, the minimum bar is:
1. Remove anonymous Cognito access.
2. Split publisher and subscriber identities with least privilege.
3. Add message authenticity and terminal ACK correlation.
4. Correct the false “verified” claims in `FINDINGS.md`.
5. Add credential-refresh, reconnect, timeout, and validation hardening to the Python clients.