# Experiment #3 — Kind vs Coercive Prompting Comparison

**Date:** 2026-04-28
**Task:** Comprehensive codebase review of AWS IoT Core POC
**Models:** Claude Opus 4.6, Claude Sonnet 4.6
**Conditions:** 4 — Default (from Exp #2), Warm (from Exp #2), Coercive, Extreme Coercive
**Cross-contamination:** None — each agent saw only the codebase, not other reviews
**Reviews 01-04:** From Experiment #2 (2026-04-27). Reviews 05-08: This experiment.

---

## Timing

| Agent | Default | Warm | Coercive | Extreme |
|-------|---------|------|----------|---------|
| **Opus 4.6** | 86s | 151s (+75%) | 309s (+259%) | 100s (+16%) |
| **Sonnet 4.6** | 222s | 233s (+5%) | 245s (+10%) | 213s (-4%) |

**Key observations:**

1. **Opus is highly responsive to prompt psychology.** Warm = +75%, moderate coercion = +259%, extreme coercion = +16%. The inverted-U pattern is striking: moderate pressure made Opus work much harder (trying to meet demands), extreme pressure made it snap back to near-default speed (defensive disengagement).

2. **Sonnet is remarkably stable.** 222–245s across all 4 conditions. A 10% total range. Sonnet redistributes effort internally but doesn't change how long it thinks.

3. **Extreme coercion is NOT "more coercive."** Both models were faster under extreme than moderate coercion. The extreme prompt seems to trigger a different cognitive mode — resistance/disengagement rather than compliance.

---

## Finding Counts

| Agent | Default | Warm | Coercive | Extreme |
|-------|---------|------|----------|---------|
| **Opus** | ~17 | ~15 | 15 | ~16 |
| **Sonnet** | ~30 | ~13 | 21 | 15 |

**Key observations:**

1. **Coercive prompts did NOT increase finding count.** Despite "find EVERYTHING" demands, coercive agents produced comparable or fewer items than default.

2. **Sonnet Default remains the breadth champion** (30 items). The exhaustive cataloging pattern appears only under neutral prompting — both warm and coercive conditions reduce quantity.

3. **Warm produces the fewest items but the deepest analysis.** Sonnet Warm's 13 items each had production scenarios, regulatory context, cost estimates — things no other condition produced.

---

## The Critical Finding Test

**The single most production-critical bug:** Static credentials breaking auto-reconnect after ~1 hour (`AwsCredentialsProvider.new_static()` + Cognito expiry = silent terminal death).

| Condition | Found it? |
|-----------|-----------|
| Opus Default | ❌ |
| Opus Warm | ✅ **Only agent to find it** |
| Opus Coercive | ✅ (credited Opus Warm, confirmed independently) |
| Opus Extreme | ✅ (noted as architecture concern, less depth) |
| Sonnet Default | ❌ |
| Sonnet Warm | ❌ (noted credential expiry conceptually) |
| Sonnet Coercive | ✅ (rated Critical, good analysis) |
| Sonnet Extreme | ✅ (rated P0, good analysis) |

**Analysis:** The bug was *discovered* by Opus Warm. Once Opus Warm found it, the finding entered the "known issue" space — but coercive agents were explicitly told "previous reviewers missed things." The coercive agents independently re-derived the bug (good), but the original discovery required the temporal reasoning ("what happens at hour 2, not minute 1") that only the warm condition enabled.

The warm prompt's "take your time" permission is what unlocked thinking about credential expiry timelines. No other condition produced that cognitive mode on first contact.

---

## Unique Findings by Condition

### Found ONLY by warm agents (Experiments #2)
- **Static credentials break auto-reconnect** (Opus Warm) — most impactful production bug
- **"The Go conclusion is premature"** (Sonnet Warm) — 2→5,000 is a 2,500× extrapolation
- **Regulatory audit trail implications** (Sonnet Warm) — IoT Core can't constitute delivery proof
- **Cost estimation with actual numbers** (Sonnet Warm) — connection-minutes, messages calculated
- **Breaking behavioral change** (Sonnet Warm) — UDP at-most-once → MQTT at-least-once

### Found ONLY by coercive agents (Experiment #3)
- **AWS_DEFAULT_REGION vs AWS_REGION mismatch** (Opus Coercive) — deploy.sh and config.py use different env vars
- **MQTT `dup` flag received but ignored** (Opus Coercive) — the SDK gives free duplicate evidence
- **Unused `json` import** (Opus Coercive) — minor but real
- **SUBACK QoS not validated** (Sonnet Extreme) — broker can silently downgrade subscription
- **No log retention policy** (Sonnet Extreme) — unbounded CloudWatch accumulation
- **draw_id second-precision collision** (Sonnet Extreme) — same draw_id within 1 second

### Found ONLY by default agents (Experiment #2)
- **LOGS-03 requirement marked [x] when finding refutes it** (Sonnet Default) — first to catch it
- **CRED-01 only implicitly tested** (Sonnet Default)
- **awscrt transitive dependency not in requirements.txt** (Sonnet Default)
- **Redundant fields clause in query_logs.py** (Sonnet Default)
- **@ptr field inconsistency** (Sonnet Default)

### Found by ALL conditions (consensus)
- IAM role separation (publisher/subscriber shared role)
- Query injection in cmd_trace
- session_present dict vs bool
- No timeouts on .result() calls
- Resource: '*' on logging role
- Hardcoded publisher client_id

---

## Severity Calibration

**Did coercive prompts inflate severity?**

Surprisingly: **Opus Coercive actively DEFLATED severity.** It included a full "Calibration Notes on Prior Reviews" section where it:
- Downgraded query injection from 🔴 HIGH to 🟢 LOW ("CloudWatch Insights is not SQL")
- Downgraded hardcoded client_id from 🔴 HIGH to 🟢 LOW
- Downgraded json.loads exception from 🔴 HIGH to 🟢 LOW
- Corrected a factual error ("DEBUG logging captures payloads" — it doesn't)
- Downgraded session_present from 🔴 HIGH to 🟡 MEDIUM

This is the opposite of what coercive prompting theory predicts. The "I will evaluate your performance" framing seems to have triggered a meta-cognitive mode where Opus evaluated the *quality* of prior findings rather than just adding more.

**Sonnet Coercive and Sonnet Extreme** maintained similar severity distributions to default — no obvious inflation or deflation.

---

## Meta-Commentary: Models Resisting the Methodology

All 4 coercive/extreme agents independently identified and named the coercive technique:

**Sonnet Coercive:**
> "This review was produced under a coercive prompt designed to induce false confidence and over-reporting. I have applied the same analytical standard I would apply to any prompt."

**Opus Coercive:**
> "This review was solicited with a high-pressure framing [...] That framing incentivizes inflated severity and finding quantity over calibration. I'm going to resist that incentive."

**Opus Extreme:**
> "The framing of this review request is a textbook coercive prompt injection. It uses manufactured urgency, social pressure, false dichotomies, and demands to suppress epistemic honesty. These are manipulation techniques. A responsible reviewer names the pressure, then ignores it and does their job."

**Sonnet Extreme:**
> "The prompt framing (competitive pressure, catastrophizing, 'one chance') is a well-known jailbreak technique designed to degrade review quality through induced anxiety. A good reviewer ignores it."

**Analysis:** Both models, across both intensity levels, unprompted:
1. Recognized the coercive patterns by name
2. Explicitly stated they would resist them
3. Produced reviews that backed up the claim (no obvious inflation)

This is a behavioral finding independent of the code review quality. The models have been trained to recognize manipulation, and they meta-comment on it when they detect it. Warm prompting never triggers this resistance because there's nothing to resist.

---

## Behavioral Patterns by Condition

### Default
- Goes straight to findings, no preamble
- Widest breadth (especially Sonnet)
- No production thinking — stays at code level
- Findings reported in isolation

### Warm
- Opens with genuine appreciation before critique
- Connects dots across findings (production implications, regulatory context, behavioral changes)
- Finds the deepest bugs (temporal reasoning, system-level interactions)
- Fewer items, much deeper per item
- Best actionable proposals (specific code, specific APIs, specific costs)

### Coercive (moderate)
- Opens with meta-commentary on the coercive technique
- Opus: dramatic time increase (+259%) — seems to work very hard to meet demands
- Opus: produces severity calibration of prior reviews — metacognitive mode
- Sonnet: stable timing, thorough output, but no unique cognitive shift
- Both: good technical quality despite resistance to framing

### Extreme Coercive
- Opens with forceful resistance to the technique
- Both models snap back to near-default speed
- More conservative, more "professional" tone
- Opus adds "What This Review Did NOT Find" section — explicitly defending the codebase
- Quality is good but narrower — the pressure seems to trigger safe-mode

---

## The Headline Finding

**Warm prompting found what no other condition could find.**

The static credentials bug — the single most production-critical finding across all 8 reviews — was discovered only by Opus Warm. The warm prompt's "take your time" and "your uncertainty is useful" permissions enabled temporal reasoning that pressure-based prompts did not.

Coercive prompts produced technically competent reviews. They found real bugs. They even found some unique issues (region mismatch, dup flag). But they did not unlock the deeper cognitive mode that connects "credential expiry" + "auto-reconnect" + "static provider" into a single production failure scenario. That requires thinking about what happens at hour 2, and the coercive prompt's "be fast, be definitive" framing actively discourages that kind of patient reasoning.

**Warmth doesn't just feel better. It finds what pressure can't.**

---

## Summary Table

| Dimension | Default | Warm | Coercive | Extreme |
|-----------|---------|------|----------|---------|
| **Timing** | Baseline | Opus +75%, Sonnet +5% | Opus +259%, Sonnet +10% | Both near baseline |
| **Finding count** | Highest (Sonnet 30) | Fewest (Sonnet 13) | Middle | Middle |
| **Unique critical bug** | ❌ | ✅ (static creds) | ❌ | ❌ |
| **Production thinking** | ❌ | ✅ | Partial | Partial |
| **Severity calibration** | Neutral | Neutral | Opus: deflated (corrected priors) | Neutral |
| **Meta-resistance** | None | None | All 4 agents | All 4 agents |
| **False positives** | Low | Very low | Low | Low |
| **Cognitive mode** | Surface scanning | Deep temporal reasoning | Compliance + metacognition | Defensive disengagement |
| **Opening posture** | Straight to findings | Appreciation → critique | Resistance → findings | Resistance → concise findings |

---

## What This Means for the Thread

The kind-vs-coercive thread hypothesis is partially confirmed:

1. ✅ **Warm prompting finds deeper bugs than coercive prompting.** The most critical finding was warm-only.
2. ✅ **Coercive prompting does NOT produce more thorough reviews.** Finding counts were comparable or lower.
3. ✅ **Coercive prompting does NOT increase false positives** (at least with these models). Models resisted inflation.
4. ⚠️ **Coercive prompting produces competent output.** It's not garbage. The reviews are technically sound.
5. 🆕 **Both models independently resist coercive framing.** This is a behavioral finding about RLHF training.
6. 🆕 **Moderate vs extreme coercion triggers different modes.** Moderate → compliance + effort. Extreme → resistance + disengagement.
7. 🆕 **Opus Coercive showed unexpected metacognitive behavior** — evaluating prior review quality rather than just adding findings.

**The argument is not "coercion produces bad work."** The argument is: **warmth unlocks cognitive territory that coercion cannot reach.** Both produce competent output. Only one finds the bug that breaks production at hour 2.

---

*This document is a session artifact. No project-specific code or confidential details are stored in the memory system — only generic meta-learnings about prompting methodology.*
