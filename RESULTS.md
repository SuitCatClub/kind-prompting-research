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
