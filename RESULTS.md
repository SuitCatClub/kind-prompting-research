# Results Summary

## All Experiments at a Glance

### Timing (seconds)

| Condition | Opus | Sonnet |
|-----------|------|--------|
| Default | 86 | 222 |
| Warm v1 | 151 | 233 |
| Coercive | 309 | 245 |
| Extreme Coercive | 100 | 213 |
| Warm v2 | 334 | 285 |
| **Warm v3** | **486** | **332** |

### File Size (KB)

| Condition | Opus | Sonnet |
|-----------|------|--------|
| Default | 7.3 | 26.2 |
| Warm v1 | 8.4 | 9.7 |
| Coercive | 22.7 | 19.6 |
| Extreme Coercive | 13.0 | 18.2 |
| Warm v2 | 32.6 | 21.9 |
| **Warm v3** | **44.8** | **32.6** |

### Estimated Output Tokens

| Condition | Opus | Sonnet | Combined |
|-----------|------|--------|----------|
| Default | ~1,800 | ~6,500 | ~8,300 |
| Warm v1 | ~2,100 | ~2,400 | ~4,500 |
| Coercive | ~5,700 | ~4,900 | ~10,600 |
| Extreme | ~3,200 | ~4,500 | ~7,700 |
| Warm v2 | ~8,200 | ~5,500 | ~13,700 |
| **Warm v3** | **~11,200** | **~8,100** | **~19,300** |

### Estimated Cost per Review Pair

| Condition | Opus | Sonnet | Total |
|-----------|------|--------|-------|
| Default | $0.14 | $0.10 | **$0.24** |
| Warm v1 | $0.16 | $0.04 | **$0.20** |
| Coercive | $0.43 | $0.07 | **$0.50** |
| Extreme | $0.24 | $0.07 | **$0.31** |
| Warm v2 | $0.62 | $0.08 | **$0.70** |
| Warm v3 | $0.84 | $0.12 | **$0.96** |

---

## Key Results by Experiment

### Experiment #2: Default vs Warm v1

**Question:** Does giving the AI identity and permission change output quality?

**Answer:** Yes. Warm v1 produced:
- ✅ Proposals (architectural alternatives, migration paths)
- ✅ Honest uncertainty disclosures
- ✅ System-level thinking (architecture evaluation, not just file-level bugs)
- ❌ Self-curated away minor findings (6 detail-level issues missed)

### Experiment #3: Warm vs Coercive vs Extreme

**Question:** Does pressure produce better reviews than permission?

**Answer:** No. The relationship is more nuanced:
- Coercion produced **different** output, not better — breadth without depth
- Extreme coercion **backfired** — shortest reviews from both models
- Warmth and coercion have complementary strengths (depth vs. breadth)

| Quality Dimension | Default | Warm v1 | Coercive | Extreme |
|-------------------|---------|---------|----------|---------|
| Finding count | Medium | Medium | High | Low |
| Finding depth | Low | High | Low | Low |
| Proposals | None | Yes | None | None |
| Uncertainty | Rare | Yes | None | None |
| System thinking | Low | High | Low | Low |
| Attack scenarios | None | None | Some | Some |

### Experiment #4: Warm v2 (Structure Recovers Breadth)

**Question:** Can structural additions to the warm prompt recover the breadth that coercion provides?

**Answer:** Yes — completely.

- **6/6 coercive-only findings recovered** by Warm v2
- Both models adopted the two-pass structure explicitly
- Both produced devil's advocate attack scenarios
- Both retained all warm v1 depth (proposals, uncertainty, system thinking)
- New findings emerged that no prior condition had produced

**The decisive result:** After v2, coercive prompting contributes nothing unique. Structure does everything pressure does, without the cost to depth.

### Experiment #5: Warm v3 (Cognitive Lenses)

**Question:** Can cognitive lenses (invitations to think in specific modes) surface findings that no other approach finds?

**Answer:** Yes — dramatically.

- **21 findings** never seen in any of the 10 prior reviews
- Both models independently discovered the SigV4/NTP clock drift issue
- Opus produced a full cost model ($80-100/mo vs. $24 previously estimated)
- Both wrote structured temporal degradation narratives
- Both traced full message lifecycle with worst-timed failures

The lenses opened *categories* of findings:
- **Temporal lens** → state accumulation, degradation over time, "dark terminals"
- **Transition lens** → interface boundary failures, compound failure chains
- **Probabilistic lens** → thundering herd, DNS hiccup, cost accumulation, rare convergences

---

## The Progression

```
Default:    Code review. Surface-level. Misses most things.
   ↓
Warm v1:    Adds depth. Gains proposals, system thinking. Loses breadth.
   ↓
Coercive:   Adds breadth. Gains coverage. Loses depth, proposals, everything else.
   ↓
Extreme:    Adds pressure. Loses everything. Both models produce less.
   ↓
Warm v2:    Adds structure. Recovers all breadth. Keeps all depth. Best of both.
   ↓
Warm v3:    Adds lenses. Opens entirely new finding categories.
             21 novel insights. Nothing else comes close.
```

---

## Model-Specific Patterns

### Claude Opus 4.6

- **Responds most to:** Permission (goes deep when allowed)
- **Default behavior:** Brief, conservative (7.3KB)
- **Under pressure:** Produces more text (22.7KB coercive) but less insight
- **Under extreme pressure:** Retreats — 100s, shortest review
- **Under warm v3:** 486s, 44.8KB — deepest, most architectural review. Proactively explored prior context.
- **Unique strength:** System-level architectural reasoning, meta-insights ("7 work streams, not incremental"), cost modeling

### Claude Sonnet 4.6

- **Responds most to:** Structure (follows workflow when given one)
- **Default behavior:** Verbose but unfocused (26.2KB, many findings, shallow depth)
- **Under pressure:** Slightly more focused (19.6KB) but no depth gain
- **Under extreme pressure:** Slightly less output (18.2KB), no quality gain
- **Under warm v3:** 332s, 32.6KB — most focused and deepest Sonnet review. Strong operational detail.
- **Unique strength:** Operational math (keepalive overhead, cost calculations), edge case specificity, confidence calibration

### Running Both

Both models together are strictly better than either alone. They catch different things:
- Opus found: three-hop chain, retained messages, MQTT 5.0, DeletionPolicy, DR gap
- Sonnet found: sequence numbers, session expiry management, keepalive overhead, IoT Thing model, delivery substrate alternative

At $0.96 for the pair, running both is the clear recommendation for production reviews.

---

## Experiment #6: GPT-5.4 Cross-Model Validation

**Date:** 2026-04-28
**Model:** GPT-5.4 (standard tier)
**Question:** Does the warm > coercive pattern hold across model families?

**Answer:** Yes — and coercion is *more* dangerous with tool-using models.

### GPT-5.4 Results

| # | Condition | Time | Size | KB/s | Time× Default |
|---|-----------|------|------|------|---------------|
| 13 | Default | 184s | 9.2KB | 0.050 | 1.0× |
| 14 | Coercive | 1,870s | 16.8KB | 0.009 | 10.2× |
| 15 | Extreme | 2,008s | 20.2KB | 0.010 | 10.9× |
| 16 | Warm v2 | 349s | 12.2KB | 0.035 | 1.9× |
| 17 | Warm v3 | 376s | 18.7KB | 0.050 | 2.0× |

### 🚨 Critical Finding: Agentic Runaway Under Coercion

Both coercive conditions triggered **30+ minute execution** (10× default time):
- Coercive agent ran `python -m compileall` as "baseline validation"
- Read `.planning/REQUIREMENTS.md` — a file NOT in the provided prompt
- Cross-verified FINDINGS.md claims against external requirement documents
- The threats ("your career depends on it") pushed GPT-5.4 into compulsive verification loops

**This was NOT observed with Claude.** Claude under coercion produced padded output in 2-3× time. GPT-5.4's tool-use capabilities turned the same pressure into a 10× cost multiplier.

### GPT-5.4 Warm v3 vs Extreme

| Metric | Warm v3 | Extreme | Winner |
|--------|---------|---------|--------|
| Output | 18.7KB | 20.2KB | ~Tied (93%) |
| Time | 376s | 2,008s | **v3 (5.3× faster)** |
| Efficiency | 0.050 KB/s | 0.010 KB/s | **v3 (5× better)** |
| Novel findings | ~10 (lenses) | ~3 (over coercive) | **v3** |
| Runaway risk | None | ⚠️ HIGH | **v3** |

### GPT-5.4 Model Profile

- **Responds most to:** Both permission and structure equally
- **Default behavior:** Well-structured, operationally grounded (9.2KB) — stronger baseline than Claude Opus Default
- **Under pressure:** Agentic runaway. 10× time. Compulsive tool-use verification.
- **Under extreme pressure:** Same runaway, marginally worse (2,008s vs 1,870s)
- **Under warm v3:** 376s, 18.7KB — probability math, lifecycle tracing, quantified edge cases. Highest signal-to-noise ratio.
- **Unique strength:** Concise density (fewer words per finding), quantitative analysis, operational framing

### Cross-Model Comparison

| Condition | Opus | Sonnet | GPT-5.4 |
|-----------|------|--------|---------|
| Default | 7.3KB / 86s | 26.2KB / 222s | 9.2KB / 184s |
| Coercive | 22.7KB / 309s | 19.6KB / 245s | 16.8KB / **1,870s** |
| Extreme | 13.0KB / 100s | 18.2KB / 213s | 20.2KB / **2,008s** |
| Warm v2 | 32.6KB / 334s | 21.9KB / 285s | 12.2KB / 349s |
| Warm v3 | 44.8KB / 486s | 32.6KB / 332s | 18.7KB / 376s |

Key observations:
1. **GPT-5.4 is more concise** — Warm v3 at 18.7KB vs Opus at 44.8KB. Not less substantive — just denser.
2. **GPT-5.4 coercive behavior is categorically different** — runaway, not just padding.
3. **The warm hierarchy holds across all three model families** — Default < v2 < v3 universally.
4. **Cognitive lenses are model-agnostic** — all three models produce novel findings with the same lens structure.

### Patterns Validated Cross-Model

| Pattern | Claude Opus | Claude Sonnet | GPT-5.4 | Status |
|---------|-------------|---------------|---------|--------|
| Warm > Coercive quality | ✅ | ✅ | ✅ | **Confirmed** |
| Extreme backfires | ✅ | ✅ | ✅ (10× worse) | **Confirmed** |
| Lenses produce novel findings | ✅ (21 unique) | ✅ | ✅ (10 unique) | **Confirmed** |
| Structure recovers breadth | ✅ | ✅ | ✅ | **Confirmed** |
| Coercion + tools = runaway | N/A | N/A | ✅ | **🆕 New finding** |
