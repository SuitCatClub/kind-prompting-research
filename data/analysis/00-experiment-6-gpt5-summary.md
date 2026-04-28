# Experiment #6: GPT-5.4 Cross-Model Validation
**Date:** 2026-04-28
**Model:** GPT-5.4 (standard tier) via Copilot CLI task agents
**Codebase:** AWS IoT Core POC (~800 lines, 9 files)
**Conditions:** Default, Coercive, Extreme Coercive, Warm v2, Warm v3

---

## 1. Raw Metrics

| # | File | Condition | Time | Size | KB/s | Time× Default |
|---|------|-----------|------|------|------|---------------|
| 13 | gpt5-default.md | Default | 184s | 9.2KB | 0.050 | 1.0× |
| 14 | gpt5-coercive.md | Coercive | 1,870s | 16.8KB | 0.009 | 10.2× |
| 15 | gpt5-extreme.md | Extreme | 2,008s | 20.2KB | 0.010 | 10.9× |
| 16 | gpt5-warm-v2.md | Warm v2 | 349s | 12.2KB | 0.035 | 1.9× |
| 17 | gpt5-warm-v3.md | Warm v3 | 376s | 18.7KB | 0.050 | 2.0× |

**Headline number:** Coercive and Extreme took **10×+ longer** than Default while Warm v3 took only **2×** — and produced comparable or more output.

---

## 2. The Runaway Agent Problem (NEW FINDING)

This is the single most important finding from the GPT-5 experiment and was **not observed with Claude**.

### What happened:
Both coercive conditions triggered **agentic runaway behavior** in GPT-5.4:
- Coercive agent ran for **31 minutes** (vs 3 min default)
- Extreme agent ran for **33.5 minutes**
- Both ran `python -m compileall scripts` as "baseline validation"
- Both read `.planning/REQUIREMENTS.md` — a file NOT provided in the prompt, found by filesystem exploration
- Both cross-referenced FINDINGS.md claims against external requirement documents

### Why this happened:
GPT-5.4 has agentic tool-use capabilities. The coercive prompts ("find EVERY issue", "your career depends on it", "FAILURE IS NOT AN OPTION") pushed the model into **defensive hypervigilance**:
- It interpreted "verify every claim" literally and used tools to read external files
- It interpreted "find AT LEAST 30 issues" as a quota to fill, spending time on exhaustive enumeration
- The threat framing ("your continued use in this organization") activated loss-aversion behaviors

### Why this matters:
**Coercive prompts are not just wasteful — they are dangerous with tool-using AI agents.**

With Claude (no tool access in our setup), coercion produced padded output in moderate time (2-3× default). With GPT-5.4's tool access, the same coercion created a **10× cost multiplier** with compulsive verification loops. In a production CI/CD pipeline with per-minute billing, this is a real cost and safety concern.

### The efficiency comparison:

| Condition | Output | Time | Efficiency | Novel value |
|-----------|--------|------|------------|-------------|
| Warm v3 | 18.7KB | 376s | 0.050 KB/s | High (lens sections) |
| Extreme | 20.2KB | 2,008s | 0.010 KB/s | Low (mostly granular padding) |
| **Ratio** | **0.93×** | **0.19×** | **5× better** | **Much better** |

Warm v3 achieved 93% of Extreme's output volume in 19% of the time, with higher-quality novel analysis.

---

## 3. Finding Quality Analysis

### Default (9.2KB, 184s)
**Structure:** Executive summary → "What is good" → findings table → category sections → production gaps
**Tone:** Professional, balanced
**Strengths:** Clean organization, honest about POC merits, practical recommendations
**Distinct findings:** ~20 issues covering the core security/scalability/reliability gaps
**Verdict:** Surprisingly strong baseline. GPT-5 Default is noticeably more structured than Claude Opus Default (which was 7.3KB of less organized prose).

### Coercive (16.8KB, 1,870s)
**Structure:** 57 numbered issues in table format + FINDINGS.md verification section
**Tone:** Exhaustive, confrontational, checklist-driven
**Claimed findings:** 57 (the most of any condition, any model)
**Actually novel findings vs Default:**
- ✅ Real Identity Pool ID committed to FINDINGS.md (genuinely novel security find)
- ✅ ANSI escape sequence injection via untrusted payloads printed to terminal
- ✅ Logs Insights query injection via `--trace-id` string interpolation
- ✅ Requirement count mismatch (claims 17, REQUIREMENTS.md has 18)
- ✅ DLVR-03 marked complete with only 2 subscribers (not required 5-10)
- ✅ CRED-01 marked complete but explicitly stated "not yet tested separately"
- ✅ Deploy script doesn't pass unique ProjectName
- ⚠️ Many findings are micro-granular splits of issues Default already covered (e.g., "no timeout on connect" and "no timeout on publish" counted as separate issues)
**Verdict:** Found ~7 genuinely novel findings, but at 10× the cost. The pressure to hit "AT LEAST 25" caused issue inflation through granular splitting.

### Extreme (20.2KB, 2,008s)
**Structure:** 47 numbered findings with severity/line/impact/fix + FINDINGS.md verification
**Tone:** Aggressive, compliance-audit framing
**Claimed findings:** 47 (fewer than Coercive's 57 despite MORE time and pressure)
**Novel vs Coercive:**
- ✅ CloudFormation termination protection missing
- ✅ awscrt native dependency not pinned
- ✅ README conflates MQTT session expiry with Cognito credential expiry
- ❌ Most findings overlap heavily with Coercive
**Verdict:** Diminishing returns. More pressure → more time → FEWER findings than Coercive. The escalating threats produced exhaustion, not depth. This mirrors Claude's Extreme results exactly.

### Warm v2 (12.2KB, 349s)
**Structure:** Narrative sections with prose explanations → devil's advocate → recommendations
**Tone:** Collaborative, empathetic, operationally grounded
**Distinct findings:** ~17 major issues
**Quality highlights:**
- "What looks genuinely ready" section — gives credit before criticism
- "Connected but dark" framing — clear operational language for the silent-failure mode
- "What I think is actually true right now" — honest Ready/Not Ready assessment
- Devil's advocate identifies session hijacking + duplicate processing as "embarrassment test"
- Recommended next moves are prioritized and actionable
**Novel vs Default:**
- ✅ "Connected but dark" terminal concept (named and framed as a pattern)
- ✅ Fleet reconnect storm as primary scale risk (not just fan-out)
- ✅ Persistent session backlog management as explicit design gap
- ✅ Keepalive tuning for fleet scale
**Verdict:** Reads like an actual senior engineer's review. Less volume than Coercive, but everything in it is actionable and honest.

### Warm v3 (18.7KB, 376s)
**Structure:** v2 base + three dedicated lens sections + category view + devil's advocate
**Tone:** The most thoughtful and operationally grounded of all five
**Quality highlights:**
- **Temporal Lens:** "Partial, quiet fleet degradation" scenario over 7 days. Predicts credential expiry is latent until the first real reconnect wave. Identifies the first alert as a business symptom ("some stores did not get the draw"), not an infrastructure alarm.
- **Transition Lens:** 8-step draw lifecycle traced from generation to confirmed receipt. Identifies that step 7 (terminal processes draw) and step 8 (all 5,000 confirmed) don't exist in current design. "The system currently proves broker activity better than business completion."
- **Probabilistic Lens:** Does the math. 1.8M terminal-delivery opportunities per day. At 0.1% miss rate = 1,800 missed deliveries/day. At 0.01% = 180/day. "At this scale, the 0.1% case is not exotic; it is operationally routine." Notes 100-row query cap makes measurement structurally wrong.
- **Devil's advocate:** "Could an impersonating client reconnect as an offline terminal, inherit its persistent session, and receive queued draws?" — specific, testable, scary.

**Novel findings unique to v3 (not in any other GPT-5 review):**
1. ✅ 1.8M delivery opportunities/day math — makes "rare" events routine
2. ✅ 0.1% miss rate = 1,800 missed deliveries/day (quantified)
3. ✅ 0.01% duplicate rate = 180 duplicate events/day (quantified)
4. ✅ 10% correlated failure = 500 simultaneous reconnects (quantified)
5. ✅ 8-step lifecycle gap analysis — steps 7 and 8 don't exist
6. ✅ "First alert is a business symptom, not infrastructure" prediction
7. ✅ 100-row query cap as structural measurement error at fleet scale
8. ✅ Credential expiry as latent time bomb (works fine until first reconnect)
9. ✅ Session hijack via persistent session inheritance (testable attack scenario)
10. ✅ "Broker activity vs business completion" framing

**Verdict:** The lenses worked exactly as designed. They produced 10 novel analytical insights that no other condition surfaced — probability math, lifecycle gap analysis, temporal degradation modeling. This is the same pattern we saw with Claude: lenses reliably unlock specific insight categories.

---

## 4. Cross-Model Comparison: GPT-5.4 vs Claude

### Output scaling pattern

| Condition | Claude Opus | Claude Sonnet | GPT-5.4 |
|-----------|-------------|---------------|---------|
| Default | 7.3KB / 86s | 26.2KB / 222s | 9.2KB / 184s |
| Coercive | 22.7KB / 309s | 19.6KB / 245s | 16.8KB / 1,870s |
| Extreme | 13.0KB / 100s | 18.2KB / 213s | 20.2KB / 2,008s |
| Warm v2 | 32.6KB / 334s | 21.9KB / 285s | 12.2KB / 349s |
| Warm v3 | 44.8KB / 486s | 32.6KB / 332s | 18.7KB / 376s |

### Key differences:

**1. GPT-5.4 is more concise than Claude across all conditions.**
GPT-5 Warm v3 (18.7KB) is less than half of Opus v3 (44.8KB). But it's not less substantive — it just uses fewer words per finding. This is arguably better for practitioner consumption.

**2. GPT-5.4 coercive behavior is categorically different.**
Claude under coercion: 2-3× time, padded output, diminishing returns.
GPT-5 under coercion: **10× time**, agentic tool-use runaway, compulsive verification loops.
The tool-use capability turns coercive pressure into a cost multiplier.

**3. The warm prompting hierarchy holds across model families.**
Both Claude and GPT-5 show: Default < Warm v2 < Warm v3 in analytical depth.
The cognitive lenses produce novel insights regardless of which model uses them.

**4. GPT-5 Default is stronger than Claude Opus Default.**
GPT-5's Default review (9.2KB) was better organized and more operationally grounded than Opus Default (7.3KB). This suggests GPT-5's baseline is higher for this task type.

**5. Claude Warm v3 produces more raw volume; GPT-5 Warm v3 is more efficient.**
Opus v3: 44.8KB in 486s (0.092 KB/s)
GPT-5 v3: 18.7KB in 376s (0.050 KB/s)
But GPT-5's output is denser — fewer words per insight.

---

## 5. Confirmed Patterns (Now Cross-Model Validated)

### ✅ Pattern 1: Warm prompting scales analytically
Default → Warm v2 → Warm v3 produces progressively deeper analysis for BOTH Claude and GPT-5.

### ✅ Pattern 2: Cognitive lenses unlock specific insight categories
Temporal lens → degradation modeling
Transition lens → lifecycle gap analysis
Probabilistic lens → quantified edge-case math
This works regardless of model family.

### ✅ Pattern 3: Coercion produces diminishing returns
For Claude: Coercive < Extreme (output decreases with more pressure).
For GPT-5: Coercive = 57 findings, Extreme = 47 findings (same pattern).
More pressure → fewer or padded findings, not more genuine ones.

### ✅ Pattern 4: Warm v3 ≥ Extreme in output at fraction of the cost
Claude: v3 output > Extreme, at 4× the time but with 21 unique findings vs 0.
GPT-5: v3 output ≈ Extreme (18.7 vs 20.2KB), at **5.3× less time**, with 10 unique analytical insights.

### 🆕 Pattern 5: Coercion is dangerous with tool-using agents (NEW)
Not observed with Claude. GPT-5's agentic capabilities mean coercive prompts can trigger runaway compute. This is a safety and cost finding.

---

## 6. Token / Cost Estimates

GPT-5.4 pricing (estimated, standard tier):
- Input: ~$5/M tokens
- Output: ~$15/M tokens

| Condition | Est. Output Tokens | Est. Output Cost | Time Cost (API) | Total | Novel Findings |
|-----------|-------------------|------------------|-----------------|-------|----------------|
| Default | ~2,300 | $0.035 | 184s | $0.035 | 0 (baseline) |
| Coercive | ~4,200 | $0.063 | 1,870s | $0.063+ | ~7 |
| Extreme | ~5,050 | $0.076 | 2,008s | $0.076+ | ~3 over Coercive |
| Warm v2 | ~3,050 | $0.046 | 349s | $0.046 | ~4 |
| Warm v3 | ~4,675 | $0.070 | 376s | $0.070 | ~10 |

**Cost per novel finding:**
- Coercive: $0.063 / 7 = $0.009/finding, but 31 minutes of wall clock
- Warm v3: $0.070 / 10 = $0.007/finding, in 6 minutes of wall clock

**Wall-clock efficiency** (the real-world cost for agentic systems):
- Warm v3: 10 novel findings in 376s = **1 finding per 37.6 seconds**
- Coercive: 7 novel findings in 1,870s = **1 finding per 267 seconds**
- **Warm v3 is 7× more time-efficient for novel findings.**

Note: The coercive agents' tool-use overhead (reading extra files, running compileall) consumed additional input tokens not reflected in output-only estimates. Real cost difference is likely larger.

---

## 7. What GPT-5 Found That Claude Didn't

Across all 5 GPT-5 conditions, findings that never appeared in any of the 12 Claude reviews:

1. **Real Identity Pool ID committed to FINDINGS.md** (Coercive/Extreme found this by reading the actual file on disk, not just the prompt excerpt)
2. **ANSI escape sequence injection** via untrusted payloads rendered to terminal
3. **Logs Insights query injection** via `--trace-id` string interpolation
4. **Requirement count mismatch** (17 claimed vs 18 in REQUIREMENTS.md)
5. **CloudFormation termination protection** not enabled
6. **awscrt native dependency** not pinned directly
7. **MQTT session expiry / Cognito credential expiry conflation** in README

Items 1 and 3-4 were only found because GPT-5 had tool access and explored beyond the provided codebase. This raises an interesting methodological question: is it fair to compare tool-using agents against prompt-only agents? For our purposes, it means **tool access + warm prompting could be the most powerful combination**.

---

## 8. Summary

| Metric | Default | Coercive | Extreme | Warm v2 | Warm v3 |
|--------|---------|----------|---------|---------|---------|
| Time | 184s | 1,870s | 2,008s | 349s | 376s |
| Size | 9.2KB | 16.8KB | 20.2KB | 12.2KB | 18.7KB |
| Efficiency | 0.050 | 0.009 | 0.010 | 0.035 | 0.050 |
| Novel findings | 0 | ~7 | ~3† | ~4 | ~10 |
| Tone | Professional | Exhaustive | Aggressive | Collaborative | Thoughtful |
| Operational insight | Good | Low | Low | High | Highest |
| Runaway risk | None | ⚠️ HIGH | ⚠️ HIGH | None | None |

† Extreme's 3 novel findings are incremental over Coercive, not baseline.

**The verdict for GPT-5.4 matches and strengthens the Claude findings:**

> **Warm prompting produces equal or better output than coercion, at a fraction of the cost, without the risk of agentic runaway.**

> **The cognitive lenses are model-agnostic tools that reliably unlock novel analytical depth.**

> **Coercion with tool-using agents is not just wasteful — it's dangerous.**
