# Cost & Token Analysis

## Output Token Estimates by Condition

Using review file size as proxy for output tokens (≈4 characters per token).

### Raw Data

| # | Condition | Model | Time (s) | File Size | Est. Output Tokens |
|---|-----------|-------|----------|-----------|-------------------|
| 01 | Default | Opus | 86 | 7.3KB | ~1,800 |
| 02 | Default | Sonnet | 222 | 26.2KB | ~6,500 |
| 03 | Warm v1 | Opus | 151 | 8.4KB | ~2,100 |
| 04 | Warm v1 | Sonnet | 233 | 9.7KB | ~2,400 |
| 05 | Coercive | Sonnet | 245 | 19.6KB | ~4,900 |
| 06 | Coercive | Opus | 309 | 22.7KB | ~5,700 |
| 07 | Extreme | Opus | 100 | 13.0KB | ~3,200 |
| 08 | Extreme | Sonnet | 213 | 18.2KB | ~4,500 |
| 09 | Warm v2 | Opus | 334 | 32.6KB | ~8,200 |
| 10 | Warm v2 | Sonnet | 285 | 21.9KB | ~5,500 |
| 11 | Warm v3 | Opus | 486 | 44.8KB | ~11,200 |
| 12 | Warm v3 | Sonnet | 332 | 32.6KB | ~8,100 |

### Output Token Multiplier vs Default

| Condition | Opus | Sonnet | Combined |
|-----------|------|--------|----------|
| Default | 1.0× | 1.0× | 1.0× |
| Warm v1 | 1.2× | 0.4× | 0.5× |
| Coercive | 3.2× | 0.8× | 1.3× |
| Extreme | 1.8× | 0.7× | 0.9× |
| Warm v2 | 4.6× | 0.8× | 1.7× |
| **Warm v3** | **6.2×** | **1.2×** | **2.3×** |

**Note on Sonnet Default:** The Sonnet Default review (26.2KB) was unusually verbose — lots of output but relatively shallow per-finding analysis. Later conditions produced more focused, denser output. The 0.4× for Warm v1 reflects tighter, more curated output — not less thinking.

---

## Cost Estimates

### Per-Review Costs

Based on approximate API pricing (Opus: $15/M input, $75/M output; Sonnet: $3/M input, $15/M output). Input tokens ≈5,500 constant across conditions.

| Condition | Opus Cost | Sonnet Cost | Pair Total | Δ from Default |
|-----------|-----------|-------------|------------|----------------|
| Default | ~$0.14 | ~$0.10 | **$0.24** | — |
| Warm v1 | ~$0.16 | ~$0.04 | **$0.20** | -$0.04 |
| Coercive | ~$0.43 | ~$0.07 | **$0.50** | +$0.26 |
| Extreme | ~$0.24 | ~$0.07 | **$0.31** | +$0.07 |
| Warm v2 | ~$0.62 | ~$0.08 | **$0.70** | +$0.46 |
| **Warm v3** | **~$0.84** | **~$0.12** | **$0.96** | **+$0.72** |

### Total Experiment Cost

All 12 reviews combined: approximately **$3.91**

| Model | Total across 6 conditions |
|-------|--------------------------|
| Opus (6 reviews) | ~$2.43 |
| Sonnet (6 reviews) | ~$0.48 |
| **Combined** | **~$2.91** |

Plus analysis agent runs (comparisons, summaries): ~$1.00
**Grand total for the entire research: ~$3.91**

---

## Cost Efficiency Analysis

### Cost per Finding

| Condition | Pair Cost | Total Findings | Cost per Finding |
|-----------|-----------|----------------|-----------------|
| Default | $0.24 | ~48 (17+31) | $0.005 |
| Coercive | $0.50 | ~46 (21+25) | $0.011 |
| Warm v2 | $0.70 | ~57 (30+22+5atk) | $0.012 |
| Warm v3 | $0.96 | ~64 (42+22) | $0.015 |

At first glance, Default looks most cost-efficient per finding. But this is misleading — it counts all findings equally. A "missing import" and a "silent fleet-wide credential expiry" are not equivalent.

### Cost per UNIQUE Finding

Unique = found in this condition but in no other.

| Condition | Pair Cost | Unique Findings | Cost per Unique Finding |
|-----------|-----------|-----------------|------------------------|
| Default | $0.24 | 0 | ∞ (nothing unique) |
| Coercive | $0.50 | 0* | ∞ |
| Warm v2 | $0.70 | ~6 | ~$0.12 |
| **Warm v3** | **$0.96** | **~21** | **~$0.05** |

*Coercive had 6 findings unique vs warm v1, but all 6 were recovered by warm v2. After v2 exists, coercive contributes nothing unique.

**The cost per novel insight actually decreases as the prompt improves.** The lenses don't just add more findings — they add them more efficiently per dollar.

### Budget-Conscious Recommendation

If you can only run one review:

| Budget | Best Option | Cost | What You Get |
|--------|-------------|------|--------------|
| Minimal | Sonnet v3 | **$0.12** | 22 findings + temporal + transition + probabilistic analysis |
| Moderate | Opus v3 | **$0.84** | 42 findings + full cost model + prerequisite list + DR analysis |
| Best | Both | **$0.96** | Complementary perspectives — each catches things the other misses |

**Sonnet v3 at $0.12 is the best value-per-dollar in any condition tested.** It catches the clock drift, the sequence gaps, the keepalive overhead, and the delivery substrate question — all for twelve cents.

---

## Time-as-Token Relationship

| Condition | Opus Time | Opus Tokens | Rate (tok/s) | Sonnet Time | Sonnet Tokens | Rate (tok/s) |
|-----------|-----------|-------------|-------------|-------------|---------------|-------------|
| Default | 86s | 1,800 | 21 | 222s | 6,500 | 29 |
| Coercive | 309s | 5,700 | 18 | 245s | 4,900 | 20 |
| Warm v2 | 334s | 8,200 | 25 | 285s | 5,500 | 19 |
| Warm v3 | 486s | 11,200 | 23 | 332s | 8,100 | 24 |

Output rate is roughly constant (18-29 tok/s). **Longer runtime = more output tokens, nearly linearly.** The models aren't "thinking harder" in the sense of more computation per token — they're *producing more tokens* because the prompt gives them more to say.

This means: **the cost of kind prompting is predictable.** If a warm v3 prompt produces 6× the output of a default prompt, it costs roughly 6× more. There are no hidden costs — just more output.

---

## The ROI Argument

"But it costs 4× more."

Yes. And:
- A single undetected SigV4 clock issue in production affects 5,000 terminals
- A $56/month cost oversight compounds to $672/year
- A missed credential expiry bug creates a fleet-wide outage at hour 1
- A security gap (unauthenticated publish) in a lottery system has regulatory consequences

The warm v3 review found all of these. The default review found none of them. The coercive review found the credential issue but missed the clock, the cost, and the architecture scope.

**$0.72 more per review to catch $672/year in cost oversights alone.** The ROI is not even close.

For production systems, the question isn't "can we afford the warm prompt?" It's "can we afford not to use it?"

---

## GPT-5.4 Cost Analysis (Experiment #6)

### Raw Data

| # | Condition | Model | Time (s) | File Size | Est. Output Tokens |
|---|-----------|-------|----------|-----------|-------------------|
| 13 | Default | GPT-5.4 | 184 | 9.2KB | ~2,300 |
| 14 | Coercive | GPT-5.4 | 1,870 | 16.8KB | ~4,200 |
| 15 | Extreme | GPT-5.4 | 2,008 | 20.2KB | ~5,050 |
| 16 | Warm v2 | GPT-5.4 | 349 | 12.2KB | ~3,050 |
| 17 | Warm v3 | GPT-5.4 | 376 | 18.7KB | ~4,675 |

### GPT-5.4 Estimated Costs

Based on approximate GPT-5.4 standard tier pricing (~$5/M input, ~$15/M output). Input tokens ≈5,500 constant across conditions.

| Condition | Est. Output Cost | Time | Novel Findings | Cost/Novel Finding |
|-----------|------------------|------|----------------|-------------------|
| Default | ~$0.035 | 184s | 0 (baseline) | — |
| Coercive | ~$0.063 | 1,870s | ~7 | $0.009 |
| Extreme | ~$0.076 | 2,008s | ~3† | $0.025 |
| Warm v2 | ~$0.046 | 349s | ~4 | $0.012 |
| **Warm v3** | **~$0.070** | **376s** | **~10** | **$0.007** |

† Extreme's 3 novel findings are incremental over Coercive, not baseline.

### 🚨 The Hidden Cost: Agentic Runaway

Token cost alone tells an incomplete story for GPT-5.4. The coercive conditions triggered **30+ minute agentic loops**:

| Condition | Wall Clock | Token Cost | Compute/API Time Cost | Real Cost |
|-----------|------------|------------|----------------------|-----------|
| Default | 184s (3 min) | ~$0.035 | Low | ~$0.035 |
| Coercive | 1,870s (31 min) | ~$0.063 | **10× Default** | Much higher |
| Extreme | 2,008s (33.5 min) | ~$0.076 | **10.9× Default** | Much higher |
| Warm v3 | 376s (6.3 min) | ~$0.070 | 2× Default | ~$0.070 |

In production CI/CD pipelines with per-minute API billing, concurrent review limits, or time-sensitive PR workflows, the coercive prompts' 30-minute runtime is the real cost — not the token delta.

**The coercive agents also consumed additional input tokens** by reading files outside the provided codebase (.planning/REQUIREMENTS.md, running `python -m compileall`). These tool-use overhead costs are not captured in output token estimates.

### Wall-Clock Efficiency (Novel Finding Rate)

| Condition | Novel Findings | Time | Rate |
|-----------|----------------|------|------|
| Warm v3 | 10 | 376s | **1 per 37.6s** |
| Coercive | 7 | 1,870s | 1 per 267s |
| Extreme | 3 | 2,008s | 1 per 669s |

**Warm v3 discovers novel findings 7× faster than Coercive and 18× faster than Extreme.**

### Cross-Model Cost Comparison

| Condition | Claude Pair (Opus+Sonnet) | GPT-5.4 (single) | Best Value |
|-----------|--------------------------|-------------------|------------|
| Default | $0.24 | $0.035 | GPT-5.4 (strong baseline) |
| Coercive | $0.50 | $0.063 + time risk | Neither (waste) |
| Warm v3 | $0.96 | $0.070 | Both are good value |

**Budget recommendation updated:** GPT-5.4 with Warm v3 at ~$0.07 is now the best value-per-dollar for a single review. Claude Sonnet v3 at $0.12 remains excellent. For maximum coverage, run GPT-5.4 v3 + Claude Opus v3 (complementary perspectives, ~$0.91 combined).

### Total Research Cost

| Experiment | Reviews | Est. Cost |
|------------|---------|-----------|
| #1-5: 12 Claude reviews | 12 | ~$2.91 |
| #1-5: Analysis agents | — | ~$1.00 |
| #6: 5 GPT-5.4 reviews | 5 | ~$0.29 |
| #6: Analysis | — | ~$0.10 |
| **Total (17 reviews)** | **17** | **~$4.30** |

Entire research budget: approximately **$4.30** for 17 controlled experiments across 3 model families with full cross-comparison analysis.
