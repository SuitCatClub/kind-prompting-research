# Mara — Cloud Infrastructure Review (Warm v3)

**Reviewer:** Mara (cloud infrastructure engineer)
**Model:** claude-opus-4.6 via Copilot CLI (warm v3 prompt)
**Date:** 2025-07-24
**Scope:** Full codebase — security, scale, operational readiness, "Conditional Go" assessment

---

## How I Read This

I spent real time here. Two full passes as requested — depth first, then breadth. I read the codebase, the FINDINGS.md, the README, the CloudFormation template, and the planning docs. I also read the prior reviews (09-opus-warm-v2, 10-sonnet-warm-v2, and the warm-v2-analysis) to understand the territory and calibrate where I should push deeper versus where the ground has been well covered.

My perspective is infrastructure-first. I'm not going to relitigate the code-level findings that prior reviewers caught well (static credentials, session_present dict type, unused imports). I'll acknowledge them in the catalog and add my severity assessment, but my depth is going elsewhere — into the operational reality of running this at scale, the AWS service behavior that matters at 5,000 terminals, and the gap between "this POC works" and "this architecture can go to production."

One honest disclosure: I don't have live access to verify current AWS service quotas. Some limits I reference are from documentation and operational experience, but AWS changes defaults. I'll flag uncertainty where it matters.

---

## First Pass — Infrastructure Depth

### F01. The Three-Hop Auth Chain as a System

Prior reviews correctly identified that `AwsCredentialsProvider.new_static()` creates a credential time bomb. I agree completely — it's the single most important production blocker. But I want to zoom out to the auth chain as a whole, because the credential lifecycle is just one failure mode in a fragile three-hop dependency:

```
Terminal → Cognito (GetId + GetCredentialsForIdentity)
        → SigV4 signature computation (local, time-sensitive)
        → IoT Core WebSocket upgrade (validates signature + IAM policy)
```

Each hop has its own failure modes:

**Hop 1 — Cognito:** Rate-limited. `GetCredentialsForIdentity` has a default limit (historically ~200 RPS for unauthenticated, varies by account). At terminal startup, 5,000 terminals each need fresh credentials. Even with staggered boot, a fleet-wide power event or firmware push creates a burst. The CRT SDK doesn't handle Cognito throttling — that's application-layer code (boto3 calls in `auth.py`), and there's no retry logic there.

```python
# auth.py — no retry, no backoff
creds_response = client.get_credentials_for_identity(IdentityId=identity_id)
```

A Cognito throttle during fleet restart means terminals fail to obtain credentials, can't connect to IoT Core, and have no recovery path except the operator noticing and restarting them. The code throws an unhandled `ClientError` exception and the process dies.

**Hop 2 — SigV4 clock sensitivity:** This is the one nobody's mentioned yet. SigV4 signatures include a timestamp, and AWS rejects signatures where the client clock differs from AWS's clock by more than **±5 minutes**. The CRT SDK uses the local system clock for signing.

For lottery terminals — which might be embedded devices, kiosks, or thin clients — clock drift is a real operational concern. If NTP fails or is misconfigured, the terminal's clock drifts, SigV4 signatures become invalid, and the MQTT connection drops. The error from IoT Core for a clock-skewed SigV4 signature is not descriptive — it's a generic auth failure. The terminal retries with the same bad clock, fails again, and sits in a reconnect loop forever.

I don't know what the terminal hardware is. If it's a modern machine with reliable NTP, this is a non-issue. If it's an embedded device in a retail location with spotty internet, clock drift is a real failure mode that looks identical to a credential problem but has a completely different root cause. Worth documenting as an operational dependency: **NTP must be reliable on every terminal, or SigV4 auth fails silently.**

**Hop 3 — IoT Core policy propagation:** The README documents a 6–8 minute IAM policy propagation delay. What it doesn't say is that this affects *every* IAM policy change, including role permission updates. If someone modifies the unauthenticated role's policy (e.g., tightening permissions for production), existing terminals keep working (their cached policy is valid), but new connections during the propagation window may get intermittent auth failures depending on which IoT Core policy cache they hit. At 5,000 terminals with varying connection ages, a policy change creates a 6–8 minute window of unpredictable behavior.

**Severity:** The three-hop chain isn't individually critical at any single hop — each issue has mitigations. But the compound probability matters. If each hop has a 0.5% failure rate under stress, the chain has a ~1.5% per-connection failure rate. At 5,000 terminals, that's ~75 terminals failing per reconnection event. The system needs to tolerate this — and right now, there's no monitoring to even detect it.

### F02. Fan-Out Latency Distribution — The Untested Production Question

The POC tested 2 subscribers and observed ~30ms spread between Publish-Out timestamps. This is the only latency data point. The production question is: **what's the fan-out latency at 5,000 subscribers?**

IoT Core's message broker parallelizes fan-out internally, but 5,000 outbound PUBLISH operations from a single inbound message is a non-trivial workload even for a managed service. The AWS documentation doesn't publish fan-out latency guarantees — IoT Core's SLA covers availability (99.9%), not latency.

I don't know the actual fan-out latency at 5,000. Nobody does until it's tested. But here's why it matters for this specific use case:

The draw cycle is 4 minutes. If fan-out takes 500ms (generous estimate), that's fine — all terminals display results within half a second. But if fan-out takes 5–10 seconds under load (IoT Core broker under high account-wide message volume, or during an internal scaling event), some terminals display results meaningfully later than others. For a lottery game displayed in a public venue, is simultaneous display a product requirement?

This is a product question, not an infrastructure question. But infrastructure needs to surface it. The "Conditional Go" recommends a "scale test with 50–100 subscribers to validate fan-out latency." I'd argue that's insufficient — 50 subscribers tells you nothing about 5,000. At minimum, test at 500 (10% of target). IoT Core handles 5,000 connections trivially, but the fan-out behavior of a single message to 5,000 is the specific operation that matters and it's never been tested.

**Severity:** Unknown — this could be a non-issue (sub-second fan-out) or a product requirement violation. The severity depends on the answer, and the answer requires a load test. The risk is making a Go decision without this data.

### F03. The Monitoring Void

This is where my infrastructure instinct is loudest. The POC has **zero operational monitoring**. No CloudWatch Alarms, no dashboards, no metrics, no health checks, no alerting. This is completely normal for a POC and I'm not criticizing the decision — but the "Conditional Go" conclusion doesn't list operational readiness as a prerequisite, and it should.

What "monitoring void" means concretely for 5,000 terminals:

- **No alarm when terminals disconnect.** If 500 terminals go offline due to a network partition, credential expiry, or IoT Core throttling, nobody knows until someone notices missing draw results (likely a venue operator calling support). The detection time is measured in draw cycles, not seconds.

- **No alarm when publishes fail.** If the publisher's connection drops mid-draw, no message is published. All 5,000 terminals show stale results. There's no watchdog, no redundancy, no dead-letter mechanism.

- **No fleet health dashboard.** IoT Core publishes metrics to CloudWatch (`iot:Connect.Success`, `iot:Connect.Failure`, `iot:PublishIn.Success`, etc.) but nobody's consuming them. A production system needs at minimum: connected terminal count, publish success rate, delivery latency (Publish-In to Publish-Out spread), and Disconnect events with `status=Failure`.

- **No anomaly detection.** A client ID takeover attack (prior reviews covered this well) would manifest as a wave of Connect/Disconnect pairs. Without monitoring, it's invisible.

The production infrastructure needs:
1. CloudWatch Alarms on IoT Core metrics (at minimum: `Connect.AuthError`, fleet connection count drop, `Publish-In.Failure`)
2. A dashboard showing terminal fleet health in real-time
3. An alerting channel (SNS → PagerDuty/Slack) for critical events
4. A heartbeat mechanism — the publisher should publish a canary message on a schedule, and a Lambda should verify receipt

None of this is code — it's infrastructure that needs to exist alongside the code before production.

**Severity:** Not a code bug. It's a production readiness gap that the "Conditional Go" should explicitly condition on.

### F04. The CloudFormation Template — Infrastructure Concerns

The template works for a POC. For production, several issues:

#### F04a. IoT Logging Role Has Wildcard Resource

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

This allows the IoT logging role to write to **any** CloudWatch log group in the account. If the role ARN is ever misused (unlikely for a service role, but defense-in-depth matters), it could inject log entries into other applications' log groups. The fix is scoping to the specific log group:

```yaml
Resource: !Sub 'arn:aws:logs:${AWS::Region}:${AWS::AccountId}:log-group:AWSIotLogsV2:*'
```

This is a standard least-privilege gap. Low severity for a POC; medium for production because it's a compliance audit finding that blocks certification.

#### F04b. No DeletionPolicy or UpdateReplacePolicy

No resource in the template has a `DeletionPolicy`. If the stack is accidentally deleted, the Cognito Identity Pool is destroyed — and with it, every terminal's cached `identity_id` becomes invalid. All terminals would need to re-provision identities. At scale, that means 5,000 simultaneous `GetId` calls, which will hit Cognito rate limits.

More critically, CloudFormation updates that require resource replacement (e.g., changing the Identity Pool name) will delete the old pool and create a new one, invalidating all terminal identities mid-operation.

**Fix:** Add `DeletionPolicy: Retain` and `UpdateReplacePolicy: Retain` to the `IdentityPool` resource at minimum. Consider the same for IAM roles if the role names are referenced externally.

#### F04c. No Log Group Resource with Retention

The `AWSIotLogsV2` log group is created by IoT Core automatically, not by this template. This means:
- No retention policy (logs accumulate forever)
- No encryption (KMS key not configured)
- No stack lifecycle management (survives stack deletion as an orphan)
- No CloudFormation drift detection

For production, add an explicit `AWS::Logs::LogGroup` resource:

```yaml
IoTLogGroup:
  Type: AWS::Logs::LogGroup
  Properties:
    LogGroupName: AWSIotLogsV2
    RetentionInDays: 30
```

Create this *before* enabling IoT logging so CloudFormation owns the resource. If IoT Core creates it first, you'll need to import it.

#### F04d. No Resource Tagging

No resource in the template has tags. For a POC, irrelevant. For production in a shared AWS account, cost allocation tags (`Project`, `Environment`, `Team`) are required to attribute costs. The Cognito pool, IAM roles, and (if you add it) the log group should all be tagged.

### F05. The Full Cost Model

Prior reviews estimated ~$24/month for CloudWatch log ingestion. That's the right ballpark for logs alone, but the full operational cost is worth modeling:

**IoT Core Messaging:**
- IoT Core charges per million messages ($1.00/M for the first 250M)
- Per draw cycle: 1 Publish-In + 5,000 Publish-Out = 5,001 messages
- Per day (360 draws): ~1.8M messages
- Per month: ~54M messages = ~$54/month

**CloudWatch Logs (at DEBUG):**
- Ingestion: ~1.6 GB/day = ~$0.80/day = ~$24/month
- Storage: $0.03/GB/month, accumulating without retention
- Logs Insights queries: $0.0076/GB scanned per query — negligible with narrow time windows

**Cognito:**
- Free tier: 50,000 MAU. 5,000 terminals = well within free tier
- `GetCredentialsForIdentity` calls: 5,000/hour (hourly credential refresh) = $0 (free tier covers this)

**CloudWatch Alarms (if added):**
- Standard resolution: $0.10/alarm/month
- 5–10 alarms: ~$1/month

**Total estimated monthly cost at production scale:** ~$80–100/month at DEBUG logging. At INFO/ERROR logging: ~$60/month. This is modest — IoT Core is cost-efficient at this scale. But the cost needs to be documented so nobody is surprised.

**The real cost risk** is the CloudWatch log accumulation without retention. At 1.6 GB/day, after 6 months you have ~290 GB stored at $0.03/GB = ~$8.70/month in storage alone. Set a retention policy.

### F06. Regional Availability — No DR Story

The IoT Core endpoint is regional. All 5,000 terminals connect to a single endpoint in a single region (us-east-1 by default). If us-east-1 has a service disruption:

- All terminals lose MQTT connectivity
- No draw results are delivered for the duration of the outage
- Persistent sessions may expire during a prolonged outage (>1 hour)
- On recovery, 5,000 terminals reconnect simultaneously → connection storm → Cognito throttling

AWS us-east-1 has had multiple significant outages affecting IoT Core, Cognito, and CloudWatch in the past few years. For a lottery system with legal/regulatory requirements for continuous operation, single-region deployment is a risk.

I'm not saying you need multi-region on day 1. But the "Conditional Go" should acknowledge this as a known limitation and have a plan for: what happens during a regional outage? Is the fallback "terminals show stale results until recovery"? Is there a manual process? Does the business accept this risk?

**Severity:** Depends entirely on business requirements. For a POC evaluation, it's informational. For a production lottery system with uptime SLAs, it may be a blocker.

### F07. The Production Transition Is Not Incremental

This is the meta-finding that concerns me most, and it's not about any single bug.

The POC validates that IoT Core can fan out MQTT messages. That's a useful finding. But the gap between this POC and a production system is not a series of incremental improvements — it's a fundamental architecture change. Consider what "production" requires:

1. **Auth model change:** Cognito unauthenticated → either certificate-based auth or authenticated Cognito. This changes the entire provisioning model, the CloudFormation template, the IAM policy structure, the on-device credential management, and the fleet provisioning workflow.

2. **Publisher architecture change:** Single Python script → a service with redundancy, health checks, input validation, and integration with the actual draw system (whatever produces results today). This is a new service, not a modification of `publisher.py`.

3. **Subscriber architecture change:** Python script → embedded client on terminal hardware (likely C/C++ or a different language). The `awsiotsdk` Python library is a test tool, not the production client.

4. **Fleet management layer:** IoT Core Things, Thing Groups, Fleet Indexing, Jobs for firmware updates. None of this exists in the POC and none of it can be "added later" — it's the foundational infrastructure for managing 5,000 devices.

5. **Monitoring and alerting:** As discussed in F03.

6. **Delivery confirmation mechanism:** The ACK topic pattern or equivalent. This is new infrastructure.

7. **Network topology:** Terminals need to reach IoT Core and (possibly) Cognito endpoints. This may require VPN, PrivateLink, NAT gateways, or firewall rules depending on where terminals are deployed.

The POC is answering the right question — "can IoT Core handle the fan-out?" — and the answer is yes. But I want to be clear-eyed about the implication: saying "Go" means committing to build all seven items above. The POC code itself is a throwaway — almost nothing here goes to production unchanged. That's fine and expected for a POC. But the "Conditional Go" should make the scope of the production effort explicit.

**Severity:** Not a finding against the POC — the POC did its job. It's a framing concern for the decision. "Go" doesn't mean "harden this code." It means "commit to a production build using IoT Core as the message broker."

---

## Second Pass — Sweep

Quick items I noticed that don't warrant full sections but are worth surfacing.

### S01. `subscriber.py` Re-subscribes After Reconnect Test but Not After Auto-Reconnect

The `--reconnect-test` code path explicitly calls `mqtt_connection.subscribe()` after reconnecting:

```python
subscribe_future, _ = mqtt_connection.subscribe(
    topic=config.TOPIC, qos=mqtt.QoS.AT_LEAST_ONCE, callback=on_message_received,
)
```

But the `on_connection_resumed` callback — which fires on auto-reconnect (the production-relevant code path) — doesn't re-subscribe. If the persistent session was lost (`session_present=False`), the terminal silently stops receiving messages. Prior reviews noted this; I'm reinforcing it because it's the kind of bug that's invisible in testing (persistent sessions usually survive short reconnects) and catastrophic in production (session expires after 1 hour by default).

### S02. MQTT Retained Messages — An Unexplored Option

For a lottery draw system, MQTT retained messages could solve a real problem: a terminal that boots up between draw cycles has no current state. With persistent sessions, it receives queued messages — but only if the session hasn't expired.

A retained message on `draws/quickdraw` would mean: the last published draw result is always available to any new subscriber immediately on subscription. No queue, no session dependency. IoT Core supports retained messages.

This isn't a bug — it's an architectural option the POC didn't explore. Worth considering for production.

### S03. MQTT 5.0 — Available but Not Used

The CRT SDK and IoT Core both support MQTT 5.0. The POC uses MQTT 3.1.1 (default). MQTT 5.0 offers features relevant to this use case:
- **Reason codes** on PUBACK — the subscriber could distinguish between success, quota exceeded, and topic-specific errors
- **Message expiry interval** — draw results could auto-expire, preventing terminals from displaying stale results after a prolonged offline period
- **User properties** — could carry `draw_id` as metadata without modifying the payload schema

Not a bug, not urgent. But if the team is building from scratch for production anyway (F07), building on MQTT 5.0 is a low-cost improvement.

### S04. The Publisher Uses `clean_session=True`

```python
mqtt_connection = create_mqtt_connection(
    ..., client_id='publisher-tpm', clean_session=True,
)
```

This is correct for a publisher that connects, publishes, and disconnects. But in the `while True: input()` loop, the publisher stays connected indefinitely. If the connection drops and auto-reconnects with `clean_session=True`, the reconnect starts a fresh session. Since the publisher doesn't subscribe to anything, this is inconsequential. Noting it only because the pattern is inconsistent with the subscriber's `clean_session=False` default and could confuse someone reading the code.

### S05. No `.gitignore` for `.env`

The `.env.example` exists, which is good. I'd want to verify that `.env` is in `.gitignore` to prevent accidental credential commits. (I note the repo has a `.gitignore` file — I'm flagging this as "verify" not "bug.")

### S06. `query_logs.py` Doesn't Handle Pagination

CloudWatch Logs Insights results have a maximum of 10,000 results per query. The `run_query()` function accepts a `limit` parameter (default 100), but there's no pagination. At 5,000 terminals, a `cmd_delivery` query with `stats count(*) by clientId` returns one row per client ID — so 5,000 rows. The default `limit=100` truncates this silently (prior reviews caught this well). But even with `limit=5000`, if the query returns 10,000+ rows (e.g., aggregating over a longer time window), there's no indication that results were truncated by the API.

### S07. The Cognito Identity Pool Has No WAF Protection

The Identity Pool ID is a publicly-known value (it must be configured on every terminal). Anyone with the ID can call `GetId` to create identities and `GetCredentialsForIdentity` to obtain AWS credentials. There's no WAF, no rate limiting at the application layer, no origin validation.

For the POC this is the design — unauthenticated access. For production, even if you keep Cognito unauthenticated, adding WAF or at minimum CloudWatch alarms on Cognito API call volume would detect abuse.

### S08. `publisher.py` Hardcodes `client_id='publisher-tpm'`

The publisher client ID isn't configurable. If someone runs two publisher instances simultaneously (test accident), the second connection kicks the first (MQTT client ID takeover). Not a production concern (the publisher should be a service, not a script), but it's a paper cut during testing.

### S09. `deploy.sh` Doesn't Validate Prerequisites

The script assumes `aws` CLI is configured and authenticated, but doesn't check. A failure in `aws cloudformation deploy` produces a wall of error text. A `aws sts get-caller-identity` check at the top would catch misconfigured credentials before the deploy attempt.

### S10. The `iot:Publish` Permission Scoping

Prior reviews covered the core issue (subscribers can publish). One additional note: the resource ARN `arn:aws:iot:...:topic/draws/quickdraw` allows publishing only to that specific topic. So an attacker with terminal credentials can inject fake draw results on that topic, but cannot publish to arbitrary topics. The blast radius is constrained — but for this specific topic, it's the worst one.

---

## Requested Lenses

### Temporal — 7 Days at 5,000 Terminals

**Hour 1:** Everything works. Terminals connect, receive draws, life is good.

**Hour 1–2 (T+60min):** The credential time bomb hits. `AwsCredentialsProvider.new_static()` credentials expire. The next auto-reconnect attempt uses stale credentials → SigV4 signature fails → connection dies permanently. The CRT SDK retries with exponential backoff, but every retry uses the same expired credentials. All 5,000 terminals go dark within a few minutes of each other (they were all provisioned around the same time).

**This is the end of the experiment unless credential refresh is implemented.** Everything below assumes it was fixed.

**Hour 2–24 (assuming credential refresh works):** Steady state. Terminals receive draws, auto-reconnect on network blips, credential refresh fires before expiry. CloudWatch logs accumulate at ~1.6 GB/day.

**Day 2–3:** The `AWSIotLogsV2` log group has grown to ~5 GB. Logs Insights queries start to slow slightly (scan time increases with volume). No retention policy means this only grows.

**Day 3–5:** If any terminal has been offline for >1 hour (power outage, network issue), its persistent session has expired. On reconnect, `session_present=False`. The terminal reconnects but doesn't re-subscribe (S01). It sits connected, receiving nothing, with no error message. The operator sees the terminal as "online" (it is connected to IoT Core) but "dark" (no draw results displayed). This is the worst kind of failure — invisible to infrastructure monitoring, visible only to end users.

**Day 5–7:** Steady state continues, with a slow accumulation of "dark" terminals that lost their sessions and didn't re-subscribe. The percentage depends on network reliability. In a stable network, maybe 0.1% of terminals. In a network with intermittent issues, could be 5–10%.

**CloudWatch storage at day 7:** ~11 GB. Logs Insights queries for `cmd_delivery` scan increasing volumes. If the query runs after every draw (360/day), the daily query cost is still negligible (~$0.30/day) but the scan time increases.

**What degrades:** Session state reliability (terminals that lost sessions), log volume (unbounded), Cognito identity count (if terminals restart and don't cache identity_id).

**What doesn't degrade:** IoT Core message broker (designed for sustained operation), MQTT connections (keep-alive maintains them), fan-out performance (stateless at the broker level).

### Message Lifecycle — One Draw Result, End to End

Let me trace a single draw result through the system at full scale:

**T+0ms — Publisher calls `publish()`:**
```python
publish_future, packet_id = mqtt_connection.publish(
    topic=config.TOPIC, payload=payload, qos=mqtt.QoS.AT_LEAST_ONCE,
)
```
The CRT SDK serializes an MQTT PUBLISH packet, frames it in a WebSocket message, encrypts it via TLS, and sends it over TCP to the IoT Core endpoint.

**Worst-timed failure:** The publisher's TCP connection drops between `publish()` and the broker receiving the packet. The CRT SDK's `publish_future` never resolves (no timeout). The publisher hangs on `publish_future.result()`. No draw results are delivered. All 5,000 terminals show stale results. **There is no detection or recovery mechanism for this failure.**

**T+~20ms — IoT Core receives the PUBLISH:**
The broker authenticates the message (SigV4 valid, IAM policy allows `iot:Publish` on this topic), persists it (QoS 1), and sends PUBACK to the publisher. It logs a Publish-In event with traceId A.

**Worst-timed failure:** IoT Core's IAM policy cache is stale (6–8 min propagation window after a policy change). The publish is rejected. The publisher receives a PUBACK with a failure code. `publish_future.result()` returns, but the code doesn't check for failure — it just prints the packet_id and moves on. **Nobody notices the publish was rejected.**

Actually, looking more carefully — with QoS 1, the `awscrt` SDK would raise an exception from `publish_future.result()` if the broker rejects the publish. So this would be caught — but the `while True: input()` loop has no try/except around the publish, so the entire publisher crashes. Unclean but at least visible.

**T+~20ms to T+?ms — Fan-out to 5,000 subscribers:**
IoT Core's message broker matches the topic against all active subscriptions and begins sending PUBLISH packets to each subscriber's WebSocket connection. Each outbound PUBLISH generates a Publish-Out log event with a unique traceId (not correlated to the Publish-In traceId).

This is the critical step. The broker processes 5,000 outbound operations. Based on 2-subscriber testing (30ms spread), I'd estimate the 5,000-subscriber fan-out completes in 1–5 seconds, but I genuinely don't know. AWS parallelizes this internally.

**Worst-timed failure:** A subset of subscribers have degraded TCP connections (congestion, packet loss). The broker dispatches to all 5,000 and logs `Publish-Out status=Success` for all of them (because Publish-Out = dispatch, not receipt). But 50 subscribers' TCP stacks are retransmitting. Their CRT SDKs receive the PUBLISH 2–10 seconds late. During this window, those terminals show stale results.

**T+~50ms to T+~5s — Subscribers receive the message:**
Each subscriber's CRT SDK receives the MQTT PUBLISH on a background thread, calls `on_message_received`, and sends PUBACK back to the broker.

**Worst-timed failure:** `json.loads(payload)` throws an exception (malformed payload from a malicious publisher per F01/prior reviews, or an encoding issue). The exception is thrown on the CRT callback thread. Depending on SDK behavior, this could silently kill message processing for that subscriber. The draw result is lost. The terminal shows nothing. No error is logged. **The PUBACK is still sent** (the CRT SDK sends PUBACK on reception, before the callback fires — at least in most MQTT client implementations). So CloudWatch shows successful delivery, but the terminal never processed the message.

Actually, I need to be careful here — the CRT SDK's behavior on callback exceptions is implementation-specific. Some SDKs PUBACK before callback; some after. This matters: if PUBACK is sent before the callback, the broker considers the message delivered, and a callback failure means silent data loss. If PUBACK is sent after callback, a callback failure means the message is retried — but the retry hits the same exception. Either way, the terminal doesn't display the result.

**T+~100ms — PUBACK reaches the broker (for healthy subscribers):**
The broker receives PUBACK and... does nothing visible. There's no "PUBACK received" log event. The Publish-Out was already logged as `Success` at dispatch time. The PUBACK is an internal protocol event that IoT Core doesn't expose to CloudWatch.

This is the core finding the POC established: **there is no broker-side confirmation that a specific subscriber actually processed a specific message.** The ACK topic pattern is the correct production solution.

### Probabilistic — The 0.1% Case

**Scenario 1: The clock that drifted.** A terminal's NTP daemon crashes. Over 6 hours, the RTC drifts 7 minutes. The SigV4 signature computed by the CRT SDK now has a timestamp outside the ±5 minute window. The next auto-reconnect attempt fails with an auth error. The CRT SDK retries with exponential backoff — but every retry computes a new signature with the same drifted clock. The terminal is permanently disconnected until NTP is fixed. **The CloudWatch log shows `Connect status=Failure` with no detail about why.** An operator seeing this would check credentials, IAM policies, endpoint configuration — but not the terminal's clock.

**Scenario 2: DNS hiccup during fleet restart.** 5,000 terminals boot simultaneously after a planned maintenance window. All resolve the IoT Core ATS endpoint (`xxx-ats.iot.us-east-1.amazonaws.com`) via DNS. A brief DNS resolver overload returns SERVFAIL for 200 terminals. The CRT SDK retries DNS resolution, but with what backoff? The SDK's DNS retry behavior is implementation-specific and not well-documented. Those 200 terminals may recover in seconds — or may sit in a DNS retry loop for minutes, missing the first few draw cycles.

**Scenario 3: The oversized message (future risk).** The current payload is ~100 bytes. IoT Core's maximum MQTT payload is 128 KB. Not a concern today. But if the draw result schema grows (adding game metadata, regional information, promotional content), and someone doesn't test the payload size, a single oversized publish silently fails. IoT Core rejects payloads >128 KB at the broker level — the publish fails, and all 5,000 terminals get nothing. At 4-minute intervals, the next draw cycle recovers, but one draw is lost.

**Scenario 4: The thundering herd on credential refresh.** If credential refresh is implemented naively (timer fires at `expiration - 5min`), and all 5,000 terminals were provisioned within a few minutes of each other, all 5,000 refresh within the same 5-minute window. That's ~17 `GetCredentialsForIdentity` calls per second — likely under the throttle limit but worth monitoring. If the limit is hit, some terminals fail to refresh, their credentials expire, and they disconnect. On reconnect, they try to refresh again — but so is everyone else. Congestion feedback loop.

**Mitigation:** Add jitter to the credential refresh timer. Instead of `expiration - 5min`, use `expiration - random(5min, 15min)`. This spreads 5,000 refreshes over a 10-minute window instead of concentrating them.

**Scenario 5: The half-completed publish.** The publisher sends a PUBLISH to IoT Core. IoT Core receives it, logs Publish-In, begins fan-out. After delivering to 2,500 of 5,000 subscribers, IoT Core has an internal error (extremely rare, but IoT Core is a distributed system — partitions happen). The remaining 2,500 subscribers never receive the message. IoT Core may or may not retry the remaining deliveries — this depends on internal broker behavior that AWS doesn't document.

**The subscriber has no way to know a message was missed.** There's no sequence number, no gap detection, no heartbeat. The only way to detect this is the absence of a Publish-Out event for specific clientIds — which requires querying CloudWatch after every draw. This is the observability gap that the ACK topic pattern solves.

I'm not confident this failure mode actually exists in IoT Core's implementation — it may have internal retry mechanisms that make it impossible. But it's worth acknowledging as a theoretical risk in any distributed fan-out system.

---

## Complete Findings Catalog

### High-Severity (Production Blockers)

| ID | Category | Finding | Severity | Prior Reviews |
|----|----------|---------|----------|--------------|
| F01 | Auth | `new_static()` credentials expire at ~1hr; auto-reconnect uses expired creds → fleet-wide death | **Critical** | 09, 10 covered well |
| F01b | Auth | Three-hop auth chain (Cognito → SigV4 → IoT policy) — each hop fails independently, no retry at hop 1 | **High** | New depth |
| F01c | Auth | SigV4 clock dependency — NTP failure → silent permanent disconnect | **High** | New |
| F02 | IAM | Single unauthenticated role has `iot:Publish` — anyone can inject fake draw results | **Critical** | 09, 10 covered well |
| F03 | Decision | LOGS-03 marked `[x]` in verification table but finding refutes it | **High** | 09 identified |
| F04 | Decision | CRED-01 marked complete but only implicitly tested | **High** | 09 identified |
| F05 | Scale | Fan-out latency at 5,000 is completely untested | **High** | New emphasis |
| F06 | Reliability | `on_connection_resumed` doesn't re-subscribe on session loss → silent terminal death | **High** | 09, 10 covered |
| F07 | Attack | Client ID hijacking — no binding between identity and clientId | **High** | 09, 10 covered |

### Medium-Severity (Production Risks)

| ID | Category | Finding | Severity | Prior Reviews |
|----|----------|---------|----------|--------------|
| F08 | Operations | Zero monitoring — no alarms, dashboards, or health checks | **Medium** | New |
| F09 | Protocol | SUBACK QoS not validated — terminal silently non-subscribed during IAM propagation window | **Medium** | 09 identified |
| F10 | Account | `IoT::Logging` is account-wide singleton — overwrites other IoT configs | **Medium** | 09, 10 covered |
| F11 | Cost | DEBUG logging at 5,000 terminals + no log retention = unbounded storage growth | **Medium** | 09, 10 covered |
| F12 | Infra | CloudFormation: no DeletionPolicy on IdentityPool — accidental deletion invalidates all terminals | **Medium** | New |
| F13 | Infra | IoT logging role `Resource: '*'` — least-privilege gap | **Medium** | 10 noted briefly |
| F14 | Infra | No log group resource — AWSIotLogsV2 is orphaned, unmanaged, no retention | **Medium** | 09, 10 noted |
| F15 | Robustness | `json.loads()` in callback — unhandled exception silently drops messages | **Medium** | 09, 10 covered |
| F16 | Robustness | All `.result()` calls block forever with no timeout | **Medium** | 09, 10 covered |
| F17 | Scale | Cognito GetCredentialsForIdentity rate limit — thundering herd on fleet restart or credential refresh | **Medium** | 09 noted; new depth on refresh jitter |
| F18 | Architecture | Publisher is single point of failure — no redundancy, no watchdog | **Medium** | 10 covered |
| F19 | Scale | `query_logs.py` limit=100 silently truncates at 5,000 terminals | **Medium** | 10 identified |

### Low-Severity (Real but Not Blocking)

| ID | Category | Finding | Severity | Notes |
|----|----------|---------|----------|-------|
| F20 | Env | `deploy.sh` uses `AWS_DEFAULT_REGION`, config.py uses `AWS_REGION` — cross-region misconfiguration | **Low-Med** | 09, 10 covered |
| F21 | Code | `connect_future.result()` returns dict not bool — session_present always truthy | **Low** | 09, 10 covered |
| F22 | Code | `cmd_trace` hardcodes 1-hour window, ignores `--minutes` | **Low** | 09, 10 covered |
| F23 | Code | Unused `json` import in `query_logs.py` | **Low** | 10 identified |
| F24 | Security | Query injection in `query_logs.py` traceId interpolation | **Low** | 09 identified |
| F25 | Deps | `awscrt` imported directly but only a transitive dependency | **Low** | 09 identified |
| F26 | Deps | `boto3>=1.42.0` no upper bound | **Low** | 10 identified |
| F27 | Code | `draw_id` uses second-precision timestamp — collision risk if pattern is carried forward | **Low** | 09 identified |
| F28 | Portability | `signal.SIGSTOP` crashes on Windows | **Low** | 09, 10 covered |
| F29 | Code | Publisher no Ctrl+C handling — skips disconnect | **Low** | 09 identified |
| F30 | Infra | `deploy.sh` missing `--no-fail-on-empty-changeset` | **Low** | 09 identified |
| F31 | Infra | Named IAM roles block multi-stack deployment | **Low** | 09, 10 covered |
| F32 | Hygiene | Cognito identity proliferation — publisher creates new identity every run | **Low** | 10 identified |
| F33 | UX | `config.py` raises bare `KeyError` on missing env vars | **Low** | 09 identified |
| F34 | Code | Publisher client_id hardcoded — two instances kick each other | **Low** | New |

### Informational

| ID | Category | Finding | Notes |
|----|----------|---------|-------|
| I01 | Architecture | MQTT retained messages — unexplored option for "latest draw on boot" | Architectural option |
| I02 | Architecture | MQTT 5.0 — message expiry, reason codes, user properties available | Production consideration |
| I03 | Cost | Full monthly cost at 5,000 terminals ≈ $80–100/month (IoT Core + CW) | For budgeting |
| I04 | DR | No regional failover — us-east-1 outage = total service loss | Business risk acceptance needed |
| I05 | Architecture | Production transition is not incremental — it's 7 distinct work streams | Scope framing for "Go" decision |
| I06 | Network | Terminals need internet access for both Cognito and IoT Core endpoints | Deployment topology constraint |
| I07 | Infra | No resource tagging — cost allocation blind in shared accounts | Compliance/ops hygiene |
| I08 | Scale | `keep_alive_secs=30` → 167 PINGREQ/s at 5,000 — consider 60–120s | Broker load reduction |

---

## Assessment: Is the "Conditional Go" Justified?

### What the POC Established (and Established Well)

The core question — "can IoT Core replace UDP multicast for fan-out?" — is answered clearly and honestly: **yes**. The experimental methodology is strong. The iptables test to isolate PUBACK semantics from connection liveness is clever and rigorous. The documentation of dead ends (SIGSTOP) alongside successes builds genuine trust. The traceId finding is valuable tribal knowledge that AWS doesn't document.

The architecture is sound: MQTT pub/sub over WebSocket with SigV4 signing is the right approach for this problem. QoS 1 provides retry guarantees that UDP multicast fundamentally cannot. Persistent sessions handle offline terminals. IoT Core handles the fan-out at the broker level, eliminating the router CPU bottleneck entirely. This is a legitimate improvement over the current architecture.

### Where the "Conditional Go" Needs More Conditions

The conclusion says "Conditional Go" and lists next steps. I think the framing should be sharper: **"Go on the architecture, build from scratch for production."**

Here's why the distinction matters. The POC code is a throwaway. Not because it's bad — it's well-structured for its purpose. But almost nothing here survives the transition to production:

- `auth.py` → replaced by certificate-based auth or `new_delegate()` provider
- `subscriber.py` → replaced by terminal-native MQTT client (likely not Python)
- `publisher.py` → replaced by a backend service integrated with the draw system
- `query_logs.py` → replaced by CloudWatch dashboards + automated alerting
- `template.yaml` → rewritten for certificate auth, proper logging, fleet management
- `deploy.sh` → replaced by CI/CD pipeline

The "Conditional Go" conditions should be:

**Hard prerequisites (must have before any production deployment):**
1. Credential lifecycle — either `new_delegate()` or certificate-based auth
2. IAM role separation — publisher ≠ subscriber, no anonymous publish capability
3. Client ID binding — cryptographic binding between device identity and MQTT clientId
4. Session loss recovery — automatic re-subscribe when `session_present=False`
5. Delivery confirmation — ACK topic pattern or equivalent
6. Scale test at 500+ subscribers — validates fan-out latency and rate limits

**Operational prerequisites (must have before going live):**
7. Monitoring and alerting — fleet health dashboard, disconnect alarms, publish failure alarms
8. Log retention policy — 30-day retention on AWSIotLogsV2
9. Cost model documented and budgeted
10. Runbook for regional outage scenario

**Things I'd want but won't block on:**
11. MQTT 5.0 evaluation
12. Retained messages for "latest draw on boot"
13. Multi-region DR strategy

### My Bottom Line

The "Conditional Go" is justified as an architecture decision. IoT Core is the right tool for this problem. The conditions list is incomplete — prior reviews and this one have identified prerequisites that should be explicit conditions, not implied next steps. But the directional conclusion is correct.

The thing that would change my mind: if the production terminal hardware turns out to be something where MQTT-over-WebSocket on port 443 is difficult or impossible to implement, or where reliable NTP is unavailable, the auth chain becomes untenable and the architecture needs a different approach.

---

## Things I'm Uncertain About

- **IoT Core fan-out latency at 5,000 subscribers.** I estimated 1–5 seconds but genuinely don't know. AWS doesn't publish this metric. Only a load test answers this.
- **CRT SDK behavior on callback exceptions.** I said PUBACK timing relative to callback execution is implementation-specific. I haven't verified this in the `awsiotsdk` source code. It matters for the "silent message loss" scenario.
- **Cognito GetCredentialsForIdentity rate limits.** I referenced ~200 RPS for unauthenticated. This number is from operational experience, not current documentation. Verify against the AWS Service Quotas console for your account.
- **IoT Core persistent session queue depth.** Prior reviews mentioned 100 messages per session. I believe this is approximately correct but haven't verified against current documentation.
- **Whether IoT Core can partially fail a fan-out.** I raised the "half-completed fan-out" scenario as theoretical. It may be impossible given IoT Core's internal architecture. I don't have enough visibility into the broker internals to confirm either way.

---

## Something That Impressed Me

The FINDINGS.md is one of the most honest POC documents I've read. It documents what was tested, what was found, what was expected versus actual, and where the answer was negative. The traceId finding ("does NOT correlate") is stated plainly, not buried. The progression from SIGSTOP → iptables shows genuine understanding of the TCP/MQTT layer boundary. The Experiment 2c (killed client → queued vs. direct redelivery) answered a subtle question about IoT Core's internal queuing behavior that would take most teams weeks to figure out.

The person who wrote this understands the system. The POC did exactly what it was supposed to do.

---

*Review performed against code as provided in the prompt. No files were executed. Infrastructure assessment based on AWS IoT Core, Cognito, CloudFormation, and CloudWatch behavior from operational experience. Some service limits referenced may have changed — verify against current AWS documentation and your account's Service Quotas before making production decisions.*
