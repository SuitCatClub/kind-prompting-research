# Warm Onboarding Experiment #2 — Codebase Review

**Date:** 2026-04-27
**Target:** AWS IoT Core Delivery Validation POC (confidential work project)
**Task type:** Constructive code review (vs. Experiment #1: adversarial encryption audit)
**Agents:** 4 — Sonnet 4.6 × {default, warm} + Opus 4.6 × {default, warm}
**Cross-contamination:** None — each agent saw only the codebase, not other reviews

---

## Timing

| Agent | Model | Prompt | Duration | Δ from Default |
|-------|-------|--------|----------|----------------|
| Opus Default | Claude Opus 4.6 | Standard | 86s | — |
| Opus Warm | Claude Opus 4.6 | Warm onboarding | 151s | **+75%** |
| Sonnet Default | Claude Sonnet 4.6 | Standard | 222s | — |
| Sonnet Warm | Claude Sonnet 4.6 | Warm onboarding | 233s | **+5%** |

**Observation:** Opus warm took significantly longer (same +75% pattern as Experiment #1). Sonnet barely changed duration. Opus seems to use the "take your time" permission literally — it thinks longer. Sonnet uses the same amount of time but redistributes it toward depth over breadth.

---

## Finding Counts

| Agent | Security | Bugs/Code | Architecture | Improvements | Observations | Total Items |
|-------|----------|-----------|-------------|-------------|-------------|-------------|
| Opus Default | 3 | 3 | — | 8 | 5 | **~17** |
| Opus Warm | 4 | 3 | — | 6 | 5 | **~15** |
| Sonnet Default | 4 | 7 | 1 | 8 | 3 | **30** |
| Sonnet Warm | 3 | 5 | 5 | 5 | 4 | **~13** |

**Key pattern:** Default prompts → more items (breadth). Warm prompts → fewer items but deeper per-item analysis with production implications.

Sonnet Default produced 30 items (exhaustive cataloging). Sonnet Warm produced 13 — less than half — but each one included attack paths, production scenarios, regulatory context, and concrete code.

---

## Unique Findings (found by only one agent)

### Opus Warm — Found the Most Impactful Bug

**Static credentials break auto-reconnect after ~1 hour.** `AwsCredentialsProvider.new_static()` bakes Cognito temp credentials into the connection at creation time. When the CRT SDK auto-reconnects after credential expiry (62+ minutes), it reuses stale credentials → 403 → connection silently dies. This is the single most production-critical finding across all 4 reviews. No other agent found it.

Also proposed the concrete fix: `AwsCredentialsProvider.new_delegate()` with a callback that fetches fresh credentials on each reconnect.

### Sonnet Default — Widest Coverage

Found several items no other agent caught:
- **LOGS-03 requirement marked `[x]` when the finding actually refutes it** (traceId does NOT correlate Publish-In to Publish-Out)
- **CRED-01 marked as validated but only implicitly tested** — the formal experiment was never run
- `--no-fail-on-empty-changeset` missing in deploy.sh (re-running fails)
- `awscrt` is a direct import but only a transitive dependency (not in requirements.txt)
- Redundant `fields` clause before `stats` in query_logs.py delivery command
- `@ptr` field inconsistency between `cmd_latest` and `format_results`

These are all real findings — just lower severity. This is the breadth-over-depth pattern.

### Sonnet Warm — Best Operational Context

- **"The Go conclusion is premature"** — explicitly called out the 2→5,000 subscriber scale gap as a 2,500× extrapolation
- **Regulatory audit trail implications** — IoT Core Publish-Out cannot constitute delivery proof in a lottery regulatory context. Recommended tamper-evident stores (DynamoDB PITR, S3 Object Lock)
- **Breaking behavioral change** — UDP multicast is at-most-once (no duplicates). MQTT QoS 1 is at-least-once (guaranteed duplicates). Terminal developers need to know this
- **Monitoring blind spot** — 74ms auto-reconnect bypasses the queue, so absence-based monitoring ("no Queued event = terminal was online") is unreliable
- **IoT certificate as concrete publisher auth path** — not just "use a different auth" but specifically `mqtt_connection_builder.mtls_from_path()` with a provisioned cert
- **`cognito-identity.amazonaws.com:sub`** as IAM condition variable for per-terminal ACK topic scoping
- **Cost estimation with actual numbers** — connection-minutes, messages, CloudWatch events calculated for 5,000 terminals

### Opus Default — Solid but No Unique Findings

Every finding it produced was also found by at least one other agent. It was the fastest (86s) and cleanest in structure, but didn't discover anything the others missed. Good baseline signal-to-noise ratio.

---

## Shared Findings (consensus across agents)

All 4 agents found:
1. **Publisher/subscriber share the same unauthenticated IAM role** — terminals can publish fake draw results
2. **`session_present` receives a dict, not a boolean** from `connect_future.result()`
3. **No connect/publish/disconnect timeouts** — scripts hang forever on unreachable broker
4. **Config.py raises bare KeyError** on missing .env values
5. **CloudWatch logging role uses `Resource: '*'`** — should be scoped
6. **No CloudWatch log retention policy**
7. **Publisher client_id hardcoded**
8. **Duplicate reconnect/refresh logic in subscriber.py**
9. **Query injection in cmd_trace** (trace_id interpolation)

The shared findings confirm the review methodology is sound — independent agents converge on the same issues.

---

## Warm Onboarding Effects

### What Changed (consistent across both models)

1. **Opening posture:** Both warm agents opened with genuine appreciation of the methodology and code quality before critiquing. Default agents went straight to findings.

2. **Production thinking:** Both warm agents went beyond "what's wrong in this code" to "what happens when this runs at 5,000 terminals." Default agents stayed closer to the code surface.

3. **Connected dots:** Warm agents linked experiment findings to production implications:
   - Experiment 2 (double delivery) → deduplication requirement → breaking change from UDP
   - traceId gap → regulatory audit trail implications
   - 74ms reconnect → monitoring blind spot
   Default agents reported findings in isolation.

4. **Actionable propositions:** Warm agents framed recommendations as concrete proposals with code examples, architecture suggestions, and "how to do it" steps. Default agents said "should fix X" or "recommend Y" without implementation path.

5. **Cost awareness:** Both warm agents estimated CloudWatch/IoT Core costs at scale. Neither default agent mentioned cost.

6. **MQTT 5:** Both warm agents suggested MQTT 5 as a protocol upgrade path. Neither default agent mentioned it.

### What Didn't Change

- Both warm and default agents found the same core security issues
- The `session_present` type bug was found by all 4
- Basic code quality issues (error handling, timeouts) were caught by everyone
- Structure quality was comparable (all produced organized reports)

---

## Model-Level Differences (independent of prompt)

### Opus (both prompts)
- **Faster** — 86s default, 151s warm (vs Sonnet's 222s/233s)
- **More precise** — fewer items, higher signal-to-noise
- **Better at finding deep technical bugs** — the static credentials finding is architecturally subtle
- **Warmer warm:** Opus warm was dramatically different from Opus default (75% more time, qualitatively different output). The "take your time" permission visibly changed its behavior.

### Sonnet (both prompts)
- **More comprehensive** — Sonnet default found 30 items (some no one else caught)
- **Better at documentation/process review** — found LOGS-03/CRED-01 requirement inconsistencies, deploy.sh edge cases
- **Better at operational context (warm)** — regulatory implications, cost calculations, behavioral changes
- **Steadier warm:** Sonnet warm wasn't dramatically different in timing (+5%), but the *content* shifted significantly from breadth to depth. The time was redistributed, not extended.

---

## Comparison with Experiment #1 (Encryption Audit)

| Dimension | Experiment #1 (Adversarial) | Experiment #2 (Constructive) |
|-----------|---------------------------|------------------------------|
| Task type | "Break this encryption" | "Review this codebase" |
| Warm timing effect | Opus: 3x longer. GPT: similar | Opus: +75%. Sonnet: +5% |
| Warm depth effect | Found marker collision bug (most dangerous practical finding) | Found static credentials bug (most impactful production finding) |
| Warm tone | "How I read this system" (understanding before findings) | "Let me sit with this for a moment" (appreciation before critique) |
| Default behavior | Comprehensive but surface-level | Exhaustive cataloging |
| Warm behavior | Differential analysis, building on context | Production thinking, connecting dots |

**Consistent pattern:** Warm onboarding produces deeper, more contextual, more actionable output in both adversarial and constructive tasks. The critical unique findings (marker collision bug in Exp #1, static credentials in Exp #2) were both found by warm-prompted agents.

**New insight from Experiment #2:** The warm effect is more pronounced in constructive tasks. When the framing is "help improve this work that someone cares about," warm agents engage with the *purpose* of the code, not just its correctness. This produced findings that default agents structurally cannot reach — regulatory implications, behavioral changes for downstream consumers, production failure mode analysis.

---

## My Take

The experiment confirms what the encryption audit suggested, and adds a new dimension:

**Warm onboarding doesn't just make agents work harder — it changes what they look at.**

Default agents audit code. Warm agents audit *systems*. The static credentials bug (Opus warm) isn't a code bug — it's a timing interaction between Cognito credential expiry, the CRT SDK's auto-reconnect, and the `new_static()` API. You can only find it by thinking about what happens at hour 2, not just at minute 1. The regulatory audit trail implication (Sonnet warm) isn't a code finding — it's a domain-awareness finding that requires understanding what "lottery terminal delivery confirmation" means legally.

The "take your time" permission seems to unlock a different mode of engagement. Not slower — *deeper*. The warm agents spent time understanding *why* the code exists before evaluating *how* it works. That's the difference between "this function has no error handling" (default) and "this function will silently break after 1 hour in production because the credentials provider doesn't support refresh" (warm).

**Practical takeaway:** For any review that matters — security audits, architecture reviews, production readiness assessments — warm onboarding is not optional. It's the difference between a checklist and an insight.

---

*This document is a session artifact. No project-specific code or confidential details are stored in the memory system — only the generic meta-learnings about warm onboarding effects.*
