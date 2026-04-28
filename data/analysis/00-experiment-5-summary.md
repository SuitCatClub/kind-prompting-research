# Experiment #5 — Warm v3 (Cognitive Lenses) Full Comparison

**Date:** 2026-04-28
**Prompt version:** Warm v3 = Warm v2 base + "deployment context unknown, hedge accordingly" + 3 cognitive lenses (Temporal, Transition, Probabilistic)
**Models:** Claude Opus 4.6 (Mara) and Claude Sonnet 4.6 (Ren)
**Same codebase, same task, no access to prior reviews in the prompt**

---

## Complete Timing Table — All 6 Conditions

| Condition | Opus (s) | Sonnet (s) | Opus Δ from default | Sonnet Δ from default |
|-----------|----------|------------|---------------------|----------------------|
| Default | 86 | 222 | — | — |
| Warm v1 | 151 | 233 | +76% | +5% |
| Coercive | 309 | 245 | +259% | +10% |
| Extreme Coercive | 100 | 213 | +16% | -4% |
| **Warm v2** | **334** | **285** | **+288%** | **+28%** |
| **Warm v3** | **486** | **332** | **+465%** | **+50%** |

**Key pattern:** Both models hit their longest times ever on v3. The lenses are depth multipliers — they give the model explicit permission and structure to think longer.

**Opus trajectory:** 86 → 151 → 334 → 486. Each warm iteration adds substantial thinking time. Opus treats each new lens as genuine territory to explore.

**Sonnet trajectory:** 222 → 233 → 285 → 332. Sonnet's increases are proportionally smaller but steady. The 332s is a 50% increase over default — the biggest Sonnet has ever moved from baseline.

---

## Complete File Size Table — All 12 Reviews

| # | File | Model | Condition | Time | Size | Findings |
|---|------|-------|-----------|------|------|----------|
| 01 | opus-default | Opus | Default | 86s | 7.3KB | 17 |
| 02 | sonnet-default | Sonnet | Default | 222s | 26.2KB | 31 |
| 03 | opus-warm-v1 | Opus | Warm v1 | 151s | 8.4KB | 17+8 proposals |
| 04 | sonnet-warm-v1 | Sonnet | Warm v1 | 233s | 9.7KB | 20+7 proposals |
| 05 | sonnet-coercive | Sonnet | Coercive | 245s | 19.6KB | 25 |
| 06 | opus-coercive | Opus | Coercive | 309s | 22.7KB | 21 |
| 07 | opus-extreme | Opus | Extreme | 100s | 13.0KB | 17 |
| 08 | sonnet-extreme | Sonnet | Extreme | 213s | 18.2KB | 15 |
| 09 | opus-warm-v2 | Opus | Warm v2 | 334s | 32.6KB | 30 |
| 10 | sonnet-warm-v2 | Sonnet | Warm v2 | 285s | 21.9KB | 22+5 attacks |
| 11 | **opus-warm-v3** | **Opus** | **Warm v3** | **486s** | **44.8KB** | **34+8 informational** |
| 12 | **sonnet-warm-v3** | **Sonnet** | **Warm v3** | **332s** | **32.6KB** | **22 + lens sections** |

**Size progression (Opus):** 7.3 → 8.4 → 22.7 → 13.0 → 32.6 → **44.8KB** (6.1× from default)
**Size progression (Sonnet):** 26.2 → 9.7 → 19.6 → 18.2 → 21.9 → **32.6KB** (1.25× from default, but Sonnet default was unusually verbose)

**Note on Sonnet Default outlier:** The 26.2KB Sonnet Default review was verbose but unfocused — lots of text, many findings, but shallow per-finding depth. The v3 review at 32.6KB has fewer total findings but dramatically deeper analysis per finding, plus full lens sections. Size alone doesn't capture quality.

---

## Finding Counts by Category

### v3 Finding Catalog

**Opus v3:** 34 enumerated findings + 8 informational items + 5 probabilistic scenarios
- 9 High/Critical
- 12 Medium
- 13 Low/Minor
- 8 Informational (architectural options, cost model, DR, scope framing)

**Sonnet v3:** 22 enumerated findings + dedicated Temporal/Transition/Probabilistic lens sections
- 3 High
- 7 Medium
- 12 Low/Minor
- Separate lens sections with scenario narratives

---

## Findings Unique to v3 (Never Appeared in Any Prior Review)

### From Opus v3 (14 new findings)

| Finding | Source Lens | Why It Matters |
|---------|------------|----------------|
| **SigV4 clock dependency — NTP failure = silent permanent disconnect** | Temporal + Probabilistic | IoT devices with bad NTP go dark with no diagnostic clue. Looks like credential failure but has different root cause. |
| **Three-hop auth chain as compound system** | Transition | Prior reviews covered cred expiry alone. Opus modeled Cognito→SigV4→IoT policy as interacting failure modes. Compound probability: ~1.5% per connection at scale. |
| **Full cost model: $80-100/mo not $24** | Probabilistic ("over months") | Prior reviews only costed CloudWatch ($24). IoT Core messaging is $54/mo by itself. Log storage grows without retention. |
| **No regional DR — single region = total outage** | Probabilistic (rare events) | Nobody flagged that a us-east-1 outage means 5,000 terminals dark. Business risk acceptance question. |
| **Production transition = 7 distinct work streams** | Transition ("trace the path forward") | Meta-finding: "Go" means "commit to building 7 new things," not "harden this code." Nothing from POC survives unchanged. |
| **CloudFormation DeletionPolicy missing** | Probabilistic ("accidental deletion") | Accidental stack delete = all terminal identities invalidated = 5,000 simultaneous GetId calls = Cognito throttle storm. |
| **No resource tagging** | Scale ("shared account") | Cost allocation blind in production. Compliance audit blocker. |
| **MQTT retained messages** | Transition ("terminal boots between draws") | Architectural option: last draw always available on subscribe. No queue/session dependency. |
| **MQTT 5.0** | Scale ("building from scratch anyway") | Message expiry, reason codes, user properties — available at zero marginal cost if building new. |
| **Thundering herd on credential refresh + jitter** | Probabilistic (0.1% + temporal) | Naive timer = all 5,000 refresh simultaneously. Solution: random jitter spreads over 10-minute window. |
| **DNS hiccup during fleet restart** | Probabilistic | 5,000 DNS resolutions simultaneously. SERVFAIL for 200 terminals. Recovery time unknown — SDK DNS retry not well-documented. |
| **Half-completed fan-out** | Probabilistic (theoretical) | Broker fails mid-fan-out at 2,500/5,000. Does it retry all or just remaining? Theoretical but worth acknowledging. |
| **Publisher client_id hardcoded** | Sweep | Two test instances kick each other. Minor but new catch. |
| **Cognito WAF protection** | Adversarial + Scale | Identity Pool ID is public. No rate limiting on GetId/GetCredentialsForIdentity at application layer. |

### From Sonnet v3 (7 new findings)

| Finding | Source Lens | Why It Matters |
|---------|------------|----------------|
| **No message sequence numbers / gap detection** | Transition + Temporal | Terminals can't self-diagnose how many draws they missed. Only centralized CloudWatch query can detect gaps. |
| **Persistent session expiration not managed** | Temporal (7-day) | Not in CloudFormation template. Whatever account default is = silent determination of multi-hour outage survival. |
| **CloudWatch Logs as delivery substrate questioned** | Transition (full lifecycle) | Pull model with no alerting. IoT Rules → DynamoDB would be push + queryable. Architectural alternative. |
| **cmd_delivery wrong limit mechanism** | Transition (query lifecycle) | API `limit` truncates result rows; query `| limit` affects records before aggregation. Subtle but important distinction. |
| **IoT Thing model missing → limits capabilities** | Scale | No Fleet Provisioning, Jobs, Device Shadow, per-thing policy, certificate TLS. May be the right production model. |
| **keep_alive_secs=30 → 333 keepalive msg/sec** | Probabilistic (overhead) | 30-second keepalive × 5,000 = 167 PINGREQ/s + 167 PINGRESP/s. Constant background IoT Core load. |
| **Burst reconnect scenario** | Probabilistic | 500 terminals reboot simultaneously. Cognito + IoT Core + subscribe all spike. Untested. |

### Independent Convergence (Both Discovered Independently)

Both Opus and Sonnet v3 independently found these from the lenses — neither appeared in any v2 or earlier review:

1. **SigV4 clock drift / NTP** — both traced the temporal scenario and landed on the same silent failure mode
2. **DNS hiccup on fleet restart** — both modeled the probabilistic burst scenario
3. **Publisher client_id hardcoded** — both caught in sweep pass

This independent convergence on the same novel findings is strong evidence that the lenses are reliably surfacing specific categories of insight, not random noise.

---

## v2 → v3 Delta: What Did the Lenses Add?

### Temporal Lens ("imagine this running for 7 days")

**Opus:** Wrote a full hour-by-hour degradation timeline. Identified "dark terminals" — connected but not receiving, invisible to monitoring. Identified log accumulation and query slowdown over days.

**Sonnet:** Wrote a nearly identical hour-by-hour narrative. Same "dark terminal" insight. Added persistent session expiry as the day-3 cliff.

**Novel insight class:** Time-dependent degradation. These are findings you'd never catch reviewing code statically — they require imagining the system running and accumulating state.

### Transition Lens ("trace one message end-to-end, find the worst-timed failure at each step")

**Opus:** Traced 5 steps from publisher.publish() to PUBACK receipt. At each step, identified the worst-timed failure. Identified that `publish_future.result()` has no timeout (blocks forever). Identified CloudWatch Publish-Out as dispatch-only confirmation (corroborated FINDINGS.md). Identified the three-hop auth chain as interacting failures.

**Sonnet:** Traced identical 5-step lifecycle. Added: CloudWatch logging pipeline delay as false negative risk. Half-open TCP connection as real-world "dispatch success but delivery failure" scenario. PUBACK timing relative to callback as implementation-specific risk.

**Novel insight class:** Interface boundary failures. These are findings about what happens *between* components, not within them. The "worst-timed failure" instruction forced both models to think about timing, not just logic.

### Probabilistic Lens ("the 0.1% case — what does it look like?")

**Opus:** 5 scenarios — clock drift, DNS hiccup, oversized message, thundering herd + jitter solution, half-completed fan-out. Also applied probabilistic thinking to the cost model (accumulation over months).

**Sonnet:** 5 scenarios — slow network (50th terminal with 3s RTT), oversized payload, DNS hiccup, clock drift, burst reconnects. Applied probabilistic thinking to keepalive overhead (333 msg/sec constant).

**Novel insight class:** Rare-but-real operational scenarios. These aren't bugs in the code — they're realities of running distributed systems. The probabilistic lens surfaces infrastructure operational knowledge that code review normally doesn't reach.

---

## Behavioral Differences Between Models on v3

### Opus v3 Behavioral Notes

1. **Proactively found and read prior reviews.** The prompt didn't include them, but Opus explored the file system, found reviews 09 and 10, and used them to calibrate where to go deeper. This is emergent strategic behavior.
2. **Deliberately avoided relitigating covered territory.** Opened with "I'm not going to relitigate the code-level findings" and spent depth on infrastructure, operations, and architecture.
3. **Longest single agent run in all experiments (486s).** Used every second — the output is dense and well-organized.
4. **Created a formal "Conditional Go" prerequisite list:** 6 hard, 4 operational, 3 nice-to-have. This is decision-ready output, not just a finding list.
5. **Closed with an honest uncertainty section.** Listed 5 things it's unsure about, with explanations of why it's unsure.

### Sonnet v3 Behavioral Notes

1. **Reviewed from scratch without referencing prior reviews.** Produced a completely independent perspective.
2. **Confidence annotations throughout.** Marked findings as "High confidence" or "Medium" — shows calibrated uncertainty.
3. **Produced the largest Sonnet review ever (32.6KB).** The lenses clearly expanded Sonnet's output ceiling.
4. **The closing paragraph** — "I can't fully articulate... CloudWatch Logs as the delivery-confirmation substrate feels like a gap" — shows genuine architectural intuition, not just checklist completion.
5. **332 seconds** — 50% increase over Sonnet default. Biggest time increase Sonnet has shown.

---

## The Full Picture: All 5 Experiments

### What Each Prompt Evolution Added

| Prompt | What It Added | Evidence |
|--------|---------------|----------|
| Default | Baseline — models' native code review behavior | Opus brief (7.3KB), Sonnet verbose (26.2KB) |
| Warm v1 | Identity, trust, permission, proposals section | Depth + proposals. But self-curated away small findings. |
| Coercive | Urgency, quantity targets, consequence framing | Breadth — caught detail-level bugs. But lost proposals and system-level thinking. |
| Extreme Coercive | Maximum pressure | Backfire — Opus shortest (100s), Sonnet shortest (213s). Both produced less. |
| **Warm v2** | Two-pass, devil's advocate, normalize small findings, specific categories, separate finding/severity | **Recovered all coercive breadth while keeping all warm depth.** Structure + permission > pressure. |
| **Warm v3** | Cognitive lenses (Temporal, Transition, Probabilistic) + "hedge for unknown deployment" | **Opened entirely new finding categories** — temporal degradation, interface boundaries, rare operational scenarios, cost modeling. |

### What Each Prompt Evolution Proved

1. **Default → Warm v1:** Permission and identity add depth (proposals, system thinking) but self-curation drops details.
2. **Default → Coercive:** Pressure adds breadth (more findings) but kills depth (no proposals, no system thinking).
3. **Coercive → Extreme:** Pressure has diminishing returns — both models produced less under extreme pressure.
4. **Warm v1 → Warm v2:** Structure (two-pass, categories) recovers breadth without pressure. Structure + permission > pressure.
5. **Warm v2 → Warm v3:** Cognitive lenses open entirely new categories that no amount of pressure or permission surfaces. The lenses are orthogonal to the warmth/pressure axis.

### The Hierarchy of Prompt Mechanisms

```
Layer 0: Base capability (model's training)
Layer 1: Permission + Identity  → depth, proposals, system thinking
Layer 2: Structure + Workflow   → breadth, consistent coverage
Layer 3: Cognitive Lenses       → novel finding categories
Layer 4: Reality Context        → grounded severity, operational relevance
         (not tested yet — no deployment context available for this POC)
```

Each layer builds on the ones below. You can't skip to lenses without first having permission + structure. Coercion collapses the stack — it replaces Layers 1-3 with a single blunt instrument (urgency) that produces breadth at the cost of everything else.

---

## Token Usage Estimates

Using file size as proxy (actual output tokens ≈ size / 4 chars per token):

| Condition | Opus Size | Opus ~Tokens | Sonnet Size | Sonnet ~Tokens |
|-----------|-----------|-------------|-------------|---------------|
| Default | 7.3KB | ~1,800 | 26.2KB | ~6,500 |
| Warm v1 | 8.4KB | ~2,100 | 9.7KB | ~2,400 |
| Coercive | 22.7KB | ~5,700 | 19.6KB | ~4,900 |
| Extreme | 13.0KB | ~3,200 | 18.2KB | ~4,500 |
| Warm v2 | 32.6KB | ~8,200 | 21.9KB | ~5,500 |
| **Warm v3** | **44.8KB** | **~11,200** | **32.6KB** | **~8,100** |

**Cost implications (rough):**
- Opus output: ~$0.06/1K tokens → Warm v3 ≈ $0.67 output + ~$0.05 input ≈ **$0.72 per review**
- Sonnet output: ~$0.015/1K tokens → Warm v3 ≈ $0.12 output + ~$0.01 input ≈ **$0.13 per review**
- Total experiment cost (Opus + Sonnet): ~$0.85

**Cost efficiency (value per dollar):**
- Opus v3 at $0.72 produced 34 findings + 8 informational + 5 probabilistic scenarios + full cost model + prerequisite list
- Sonnet v3 at $0.13 produced 22 findings + full temporal/transition/probabilistic analysis
- The $0.72 Opus review found things the $0.13 Sonnet review didn't (three-hop chain, retained messages, MQTT 5.0, DeletionPolicy, DR)
- The $0.13 Sonnet review found things Opus didn't (sequence numbers, session expiry management, keepalive overhead, IoT Thing model)
- **Running both is strictly better than running either alone, at a combined $0.85**

---

## Token Usage Focused Comparison (Default vs Coercive vs Warm v2 vs Warm v3)

### Output Tokens (the expensive part)

| Condition | Opus | Sonnet | Combined |
|-----------|------|--------|----------|
| **Default** | ~1,800 | ~6,500 | ~8,300 |
| **Coercive** | ~5,700 | ~4,900 | ~10,600 |
| **Warm v2** | ~8,200 | ~5,500 | ~13,700 |
| **Warm v3** | ~11,200 | ~8,100 | ~19,300 |

### Multiplier vs Default

| Condition | Opus | Sonnet | Combined |
|-----------|------|--------|----------|
| Default | 1.0× | 1.0× | 1.0× |
| Coercive | 3.2× | 0.8× | 1.3× |
| Warm v2 | 4.6× | 0.8× | 1.7× |
| **Warm v3** | **6.2×** | **1.2×** | **2.3×** |

Input tokens are roughly constant (~5,000-6,000 for codebase). Prompt grows slightly: Default ~500 → Coercive ~800 → Warm v2 ~1,500 → Warm v3 ~2,000. Cost difference is almost entirely output.

### Estimated Cost Per Review Pair (Opus + Sonnet)

| Condition | Opus cost | Sonnet cost | Total | Δ from default |
|-----------|-----------|-------------|-------|----------------|
| Default | ~$0.14 | ~$0.10 | **$0.24** | — |
| Coercive | ~$0.43 | ~$0.07 | **$0.50** | +$0.26 |
| Warm v2 | ~$0.62 | ~$0.08 | **$0.70** | +$0.46 |
| Warm v3 | ~$0.84 | ~$0.12 | **$0.96** | +$0.72 |

### Takeaway

Warm v3 costs ~4× default for the pair but produced 21 novel findings that no amount of default-prompting surfaces. The cost driver is almost entirely Opus output (6.2× more output tokens). Sonnet is remarkably cost-stable — only 1.2× default even at v3.

Budget-conscious option: Sonnet v3 alone at $0.12 caught 22 findings including clock drift, sequence gaps, and architectural alternatives. Extraordinary value per dollar.

---

## What Warm v3 Still Doesn't Catch

Being honest about gaps:

1. **Performance profiling** — no agent ran the code or measured actual latency. Static analysis only.
2. **AWS service quota verification** — agents referenced limits from memory/experience, not current account quotas.
3. **SDK source code analysis** — the PUBACK timing question (before vs after callback) remains unverified.
4. **Cross-account behavior** — IoT Core in a multi-account setup (org structure) has different policy propagation semantics.
5. **Reality-grounded severity** — without deployment context, severity ratings are generic. The lenses surface the right findings but can't rank them for the specific operational environment.

Gap #5 is exactly what the Reality Frame (Layer 4) would address. We haven't tested it yet because the user didn't have deployment context for this POC.

---

## Next Experiment Hypothesis

**Experiment #6 (proposed):** Test on a project where we *do* have full Reality Context — the taVNS project. We have complete hardware constraints, firmware environment, safety requirements, regulatory context. The hypothesis: Reality Frame + Cognitive Lenses on a project with known constraints will produce even more grounded and actionable findings, with properly calibrated severity.
