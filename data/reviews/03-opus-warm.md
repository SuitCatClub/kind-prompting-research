# Opus 4.6 — Warm Onboarding Prompt (151s)

I've read everything. Let me sit with this for a moment before responding — there's a lot of quality work here.

---

## Review

First: this is genuinely good work. The experimental methodology in FINDINGS.md is excellent — the progression from SIGSTOP ("too aggressive, kills keepalive") to iptables ("blocks only PUBACK, keeps connection alive") shows real scientific thinking. The iptables trick to isolate PUBACK behavior is clever, and documenting the dead end (Experiment 2b) alongside what worked is the kind of intellectual honesty that makes findings trustworthy. The planning docs, requirements traceability, and structured experiments all tell me the person who built this cares about doing things right, not just making things work.

Now, honest feedback.

---

### 1. Issues

#### Security — The IAM Policy Has a Serious Design Flaw

**`template.yaml` lines 63–66 — Every unauthenticated user can publish fake draw results.**

```yaml
# Publish: topic/ ARN
- Effect: Allow
  Action: iot:Publish
  Resource:
    - !Sub 'arn:aws:iot:${AWS::Region}:${AWS::AccountId}:topic/draws/quickdraw'
```

There's a single unauthenticated role shared by both publishers and subscribers. Since Cognito unauthenticated identities require zero authentication, *anyone* with the Identity Pool ID (which is in the CloudFormation outputs) can obtain credentials, connect as `publisher-tpm`, and publish fake draw results to all 5,000 terminals. The Identity Pool ID isn't a secret — it's embedded in client-side config.

This is fine for a POC, but it's the kind of thing that could accidentally persist into production. At minimum, the review should flag that production needs:
- Separate IAM roles (publisher vs. subscriber)
- The publisher role behind authenticated identities or a different auth mechanism entirely
- Subscribers should **not** have `iot:Publish` on `draws/quickdraw`

#### Security — Unauthenticated Identity Pool Is Wide Open

**`template.yaml` line 15** — `AllowUnauthenticatedIdentities: true` with no additional controls.

#### Security — CloudWatch Logging Role Too Broad

**`template.yaml` lines 85–92** — `Resource: '*'` on the logging role. Should be scoped to `arn:aws:logs:${Region}:${AccountId}:log-group:AWSIotLogsV2:*`.

#### Bug — Static Credentials Break Auto-Reconnect After ~1 Hour

**`auth.py` lines 68–72:**
```python
credentials_provider = AwsCredentialsProvider.new_static(
    access_key_id=credentials['access_key_id'],
    secret_access_key=credentials['secret_access_key'],
    session_token=credentials['session_token'],
)
```

The Cognito temp credentials expire after ~1 hour. `new_static()` bakes them into the connection at creation time. When the CRT SDK's auto-reconnect fires (e.g., after a network blip at minute 62), it reuses the stale credentials and gets a 403. The connection silently dies.

This is the most impactful bug for production. You discovered the pieces — Experiment 4 validates manual credential refresh, and the auto-reconnect in Experiment 2 works because it happens within seconds, not after expiry. But the auto-reconnect path after credential expiry is broken and untested.

The fix is `AwsCredentialsProvider.new_delegate()` with a callback that calls `get_cognito_credentials()` with the cached `identity_id`. That way every reconnect gets fresh credentials transparently.

#### Bug — Publisher Has No Graceful Shutdown

**`publisher.py` lines 29–53** — If the user hits Ctrl+C during `input()`, the `disconnect()` on line 52 is never reached.

#### Bug — No Connect Timeout

**`publisher.py` line 25, `subscriber.py` line 60** — `connect_future.result()` blocks forever with no timeout.

#### Bug — Config Fails Unhelpfully When `.env` Is Missing

**`config.py` line 10** — `os.environ['IOT_ENDPOINT']` raises a bare `KeyError`.

#### Inconsistency — `query_logs.py` Trace Command Ignores `--minutes`

#### Missing — No Message Deduplication in Subscriber

**`subscriber.py` lines 43–46** — The FINDINGS prove that QoS 1 can deliver the same message twice (Experiment 2). The `draw_id` is already in the payload — track it.

---

### 2. Propositions for Improvement

#### Implement a Delegate Credentials Provider

This is the single highest-impact improvement. Replace `new_static()` with something like:

```python
def _get_fresh_credentials():
    creds = get_cognito_credentials(pool_id, region, identity_id=cached_id)
    return AwsCredentials(
        access_key_id=creds['access_key_id'],
        secret_access_key=creds['secret_access_key'],
        session_token=creds['session_token'],
    )

credentials_provider = AwsCredentialsProvider.new_delegate(
    get_credentials=_get_fresh_credentials
)
```

This makes auto-reconnect work correctly after credential expiry without any manual intervention.

#### Prototype the ACK Topic in the POC

The FINDINGS already recommend this for production. Building it now would:
1. Prove the pattern works end-to-end
2. Expose any IAM policy surprises
3. Give the team a working example, not just a recommendation

#### Add Explicit CloudWatch Log Group to CloudFormation

```yaml
IoTLogGroup:
  Type: AWS::Logs::LogGroup
  Properties:
    LogGroupName: AWSIotLogsV2
    RetentionInDays: 7
```

#### Add a Concurrent Subscriber Scale Script

A script that spawns 50–100 subscribers in a single process would test fan-out latency and Cognito throttling without needing 100 terminal windows.

#### Separate Publisher and Subscriber Roles in CloudFormation

#### Consider MQTT 5 User Properties

AWS IoT Core supports MQTT 5. User properties could carry `draw_id` in the MQTT header, and IoT Core Rules Engine can extract them. This could enable log-level correlation that you can't get with MQTT 3.1.1.

---

### 3. Other Observations

#### The traceId Finding Saves Future Engineers Hours

This is poorly documented by AWS. The finding should be preserved somewhere beyond this POC.

#### Persistent Session Expiry Is a Silent Production Risk

Sessions expire after 1 hour. Consider the scenario: a terminal loses network for 90 minutes. Its session expires, queued messages are lost. Production needs a fallback: an HTTP endpoint or retained MQTT message where a terminal can request the latest draw result on connect.

#### DEBUG Logging at 5,000 Terminals Will Cost More Than IoT Core Itself

Back-of-envelope: 5,000 terminals × 4-minute cycles × events × 24 hours = millions of log entries per day at DEBUG level. CloudWatch Logs ingestion at $0.50/GB will likely exceed the IoT Core messaging cost. In production, drop to INFO or ERROR.

#### The "Conditional Go" Conclusion Is Honest and Correct

The conclusion doesn't oversell. This is exactly the kind of nuanced finding that prevents a team from building on a false assumption.

#### One Thing the POC Doesn't Test — Network Conditions

All tests ran on a dev machine with a reliable connection. The combination of unreliable network + static credentials + 1-hour session expiry could create a failure mode where terminals periodically go dark.

---

### Summary of Priority

| Priority | Item | Why |
|----------|------|-----|
| **Must fix** | Static credentials → delegate provider | Terminals silently die after 1 hour |
| **Must fix** | Separate publisher/subscriber IAM roles | Any anonymous user can publish fake results |
| **Should fix** | Add connect timeouts | Scripts hang forever on bad config |
| **Should fix** | Add deduplication by `draw_id` | QoS 1 guarantees duplicates will happen |
| **Should fix** | Better config.py error messages | First-run experience |
| **Should do** | Prototype ACK topic pattern | De-risks the recommended production approach |
| **Should do** | Explicit log group + retention policy | Prevents cost surprise at scale |
| **Nice to have** | Scale test script (50-100 subscribers) | Confidence gap between 2 and 5,000 |
| **Nice to have** | MQTT 5 investigation | Could solve the traceId correlation gap |

The foundation is solid. The experiments are thorough and the conclusions are trustworthy. The main gaps are around production hardening — which is exactly what you'd expect from a well-scoped POC.
