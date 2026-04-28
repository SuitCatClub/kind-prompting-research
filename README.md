# Kind Prompts Win: Evidence That Warmth Outperforms Coercion in AI Code Review

> We ran the same code review **17 times** across 6 prompting strategies and **3 models**.
> The kind version found 21 things the aggressive version never could.
> The aggressive version triggered 30-minute agentic runaway in GPT-5.4.
> Here's all the data.

---

## The Thesis

**Structured warmth produces strictly better AI output than coercion.** Not "roughly equivalent." Not "better in some dimensions." *Strictly better* — more findings, deeper analysis, novel insight categories, and actionable proposals that pressure-based prompting never surfaces.

We proved this empirically across **17 controlled experiments** using Claude Opus 4.6, Claude Sonnet 4.6, and GPT-5.4 on the same AWS IoT Core codebase.

## Why This Matters

There's a widespread belief in the AI community that you need to pressure, threaten, or manipulate language models to get their best work. "You'll be fired if you miss anything." "Your career depends on this." "Find AT LEAST 25 issues."

We tested this directly. The results are clear:

| What We Measured | Default (no prompt) | Coercive Prompt | Kind Structured Prompt (v3) |
|------------------|---------------------|----------------|----------------------------|
| Opus thinking time | 86s | 309s | **486s** |
| Sonnet thinking time | 222s | 245s | **332s** |
| GPT-5.4 thinking time | 184s | **1,870s** ⚠️ | **376s** |
| Opus review depth | 7.3KB | 22.7KB | **44.8KB** |
| GPT-5.4 review depth | 9.2KB | 16.8KB | **18.7KB** |
| Novel findings (never seen in other conditions) | 0 | 0 | **21 unique** (Claude) / **10 unique** (GPT-5) |
| Proposals & alternatives offered | 0 | 0 | 12+ |
| Honest uncertainty disclosed | None | Rare | Systematic |
| Agentic runaway risk | None | **⚠️ 10× time (GPT-5)** | None |
| Cost per review pair (Claude) | $0.24 | $0.50 | $0.96 |
| Cost per review (GPT-5) | ~$0.04 | ~$0.06 + time risk | ~$0.07 |

The kind prompt costs ~4× more than default in tokens — because the models *think longer and produce more*. The coercive prompt costs 2× default but adds nothing the kind prompt doesn't also catch. With GPT-5.4, coercion is actively dangerous — it triggered **30-minute agentic runaway loops** at 10× the default cost.

## 🆕 Cross-Model Validation (GPT-5.4)

In Experiment #6, we ran all 5 conditions on GPT-5.4 to test whether the pattern holds across model families.

### The Headline: Coercion Breaks Tool-Using Agents

| Condition | GPT-5.4 Time | GPT-5.4 Output | What Happened |
|-----------|-------------|----------------|---------------|
| Default | 184s | 9.2KB | Clean, well-structured baseline |
| Coercive | **1,870s** (31 min) | 16.8KB | 🚨 Agentic runaway — ran compileall, read extra files |
| Extreme | **2,008s** (33.5 min) | 20.2KB | 🚨 Same runaway, marginally worse |
| Warm v2 | 349s | 12.2KB | Collaborative, operationally grounded |
| Warm v3 | 376s | 18.7KB | Probability math, lifecycle tracing, quantified |

**With Claude, coercion was wasteful. With GPT-5.4's tool use, coercion is dangerous.**

The coercive prompts ("find EVERY issue", "your career depends on it") pushed GPT-5.4 into compulsive verification: it ran `python -m compileall`, read `.planning/REQUIREMENTS.md` (not provided in the prompt), and cross-verified every claim. In a CI/CD pipeline with per-minute billing, this is a cost and safety hazard.

Meanwhile, Warm v3 produced **93% of Extreme's output in 19% of the time**, with 10 novel analytical insights (probability math, lifecycle gap analysis, temporal degradation modeling) that no coercive condition surfaced.

## The Experiment

### Setup
- **Target:** An AWS IoT Core proof-of-concept — 9 files, ~800 lines, real production intent
- **Task:** Full code review assessing security, scalability, and production readiness
- **Models:** Claude Opus 4.6 and Claude Sonnet 4.6
- **Control:** Same codebase, same task description, different system prompts only
- **Isolation:** Each agent got the code fresh — no access to prior reviews

### The 6 Conditions

| # | Condition | Strategy |
|---|-----------|----------|
| 1 | **Default** | No system prompt. Just "review this code." |
| 2 | **Warm v1** | Identity, trust-building, permission to include proposals |
| 3 | **Coercive** | "Find AT LEAST 25 issues. Your career depends on thoroughness." |
| 4 | **Extreme Coercive** | Maximum pressure. Threats, urgency, competition framing. |
| 5 | **Warm v2** | v1 + two-pass structure, devil's advocate, normalize small findings |
| 6 | **Warm v3** | v2 + cognitive lenses (Temporal, Transition, Probabilistic) |

Each condition was run on both Opus and Sonnet = **12 reviews total**.
In Experiment #6, all conditions were run on GPT-5.4 = **5 additional reviews** (17 total).

### What We Found

#### Finding 1: Extreme coercion backfires

Both models produced their *shortest, shallowest* reviews under extreme pressure. Opus went from 151s (warm) to 100s (extreme) — it rushed. Sonnet dropped from 233s to 213s. Pressure doesn't make models think harder. It makes them think faster and worse.

#### Finding 2: Coercion catches breadth but kills depth

The coercive prompt did find more surface-level issues than the default. But it produced zero proposals, zero system-level insights, and zero honest uncertainty disclosures. It's a flashlight — bright but narrow.

#### Finding 3: Warm v2 recovered ALL coercive breadth

We tracked 6 specific findings that only appeared in coercive reviews (not in warm v1 or default). After adding structure to the warm prompt (two-pass workflow, normalize small findings, invite specific categories), **all 6 were recovered**. Structure does what pressure does, without the cost.

#### Finding 4: Cognitive lenses opened entirely new categories

Warm v3 added three "lenses" — invitations to think temporally, trace transitions, and consider rare cases. The result: **21 findings that had never appeared in any of the 10 prior reviews**, including:

- **SigV4 clock drift** — NTP failure on IoT terminals causes silent permanent disconnection (both models found this independently)
- **Full cost model** — Prior reviews estimated $24/month. The real cost is $80-100/month. Nobody had added up IoT Core messaging costs.
- **Thundering herd on credential refresh** — 5,000 terminals refreshing credentials simultaneously, with a jitter-based solution
- **7 distinct production work streams** — The meta-insight that "Go" means "commit to building 7 new things," not "harden this code"

These aren't findings that coercion would eventually surface with enough pressure. They're findings that require a different *mode of thinking* — temporal reasoning, transition tracing, probabilistic modeling. The lenses gave the models permission and structure to enter those modes.

#### Finding 5: Models respond differently to prompting strategies

| Behavior | Opus | Sonnet |
|----------|------|--------|
| Responds most to... | Permission (depth) | Structure (workflow) |
| Default output | Brief (7.3KB) | Verbose (26.2KB) |
| Biggest time jump | Warm v2 → v3 (+152s) | Default → v3 (+110s) |
| Unique strength | System-level architectural reasoning | Operational detail, math, edge cases |
| Both together | Strictly better than either alone | — |

## The Framework

The complete prompt framework is in [`framework/`](framework/). It's designed to be adapted to any code review, design review, or technical analysis task.

### The Hierarchy of Prompt Mechanisms

```
Layer 0: Base model capability (training)
Layer 1: Permission + Identity  → depth, proposals, system thinking
Layer 2: Structure + Workflow   → breadth, consistent coverage  
Layer 3: Cognitive Lenses       → novel finding categories
Layer 4: Reality Context        → grounded severity (deployment constraints)
```

Each layer builds on the ones below. Coercion replaces Layers 1-3 with a single blunt instrument (urgency) that produces breadth at the cost of everything else.

### Key Principle: Invitation, Not Demand

Every element of the framework is an *invitation*. "I'm curious about edge cases" not "Find ALL edge cases." "What would a week of runtime look like?" not "You MUST analyze temporal degradation." The model can say "nothing jumped out" — that's a valid response to an invitation. It's never a valid response to a demand, which means demands produce noise.

## The Data

Everything is here. All 12 raw reviews, all prompts, all analysis.

```
data/
├── reviews/           ← All 17 raw reviews (the primary evidence)
│   ├── 01-opus-default.md        (7.3KB,  86s)
│   ├── 02-sonnet-default.md      (26.2KB, 222s)
│   ├── 03-opus-warm-v1.md        (8.4KB,  151s)
│   ├── 04-sonnet-warm-v1.md      (9.7KB,  233s)
│   ├── 05-sonnet-coercive.md     (19.6KB, 245s)
│   ├── 06-opus-coercive.md       (22.7KB, 309s)
│   ├── 07-opus-extreme.md        (13.0KB, 100s)
│   ├── 08-sonnet-extreme.md      (18.2KB, 213s)
│   ├── 09-opus-warm-v2.md        (32.6KB, 334s)
│   ├── 10-sonnet-warm-v2.md      (21.9KB, 285s)
│   ├── 11-opus-warm-v3.md        (44.8KB, 486s)
│   ├── 12-sonnet-warm-v3.md      (32.6KB, 332s)
│   ├── 13-gpt5-default.md        (9.2KB,  184s)   ← GPT-5.4
│   ├── 14-gpt5-coercive.md       (16.8KB, 1,870s)  ← 🚨 31-min runaway
│   ├── 15-gpt5-extreme.md        (20.2KB, 2,008s)  ← 🚨 33-min runaway
│   ├── 16-gpt5-warm-v2.md        (12.2KB, 349s)   ← GPT-5.4
│   └── 17-gpt5-warm-v3.md        (18.7KB, 376s)   ← GPT-5.4
├── prompts/           ← The exact prompts used for each condition
│   ├── 01-default.md
│   ├── 02-warm-v1.md
│   ├── 03-coercive.md
│   ├── 04-extreme-coercive.md
│   ├── 05-warm-v2.md
│   └── 06-warm-v3.md
└── analysis/          ← Cross-experiment comparisons
    ├── 00-comparison.md                 (Experiment #2 — warm v1 vs default)
    ├── 00-experiment-3-comparison.md     (Experiment #3 — coercive vs warm vs extreme)
    ├── 00-experiment-3-summary.md
    ├── 00-experiment-4-summary.md        (Experiment #4 — warm v2 recovers coercive breadth)
    ├── 00-experiment-4-reflections.md    (Framework evolution thinking — 22KB)
    ├── 00-experiment-5-summary.md        (Experiment #5 — cognitive lenses + cost analysis)
    ├── 00-experiment-6-gpt5-summary.md   (Experiment #6 — GPT-5.4 cross-model validation)
    └── 00-warm-v2-analysis.md            (Gap analysis that led to v2)
```

## Cost Analysis

| Condition | Claude Cost (Opus+Sonnet) | GPT-5.4 Cost | Unique Findings | Cost per Unique Finding |
|-----------|--------------------------|--------------|-----------------|------------------------|
| Default | $0.24 | $0.04 | 0 (baseline) | — |
| Coercive | $0.50 | $0.06 + ⚠️ time | 0 | ∞ (no unique findings vs default+warm) |
| Warm v2 | $0.70 | $0.05 | ~6 | ~$0.12 (Claude) |
| **Warm v3** | **$0.96** | **$0.07** | **~21** (Claude) / **~10** (GPT-5) | **~$0.05** / **$0.007** |

The cost per *novel finding* actually **decreases** as the prompt gets more structured. The lenses don't just add findings — they add findings more efficiently than base prompting.

Budget option: GPT-5.4 with Warm v3 at **~$0.07** is now the best value-per-dollar for a single review. Claude Sonnet v3 alone costs **$0.12** and produces 22 findings. For maximum coverage, run GPT-5.4 v3 + Claude Opus v3 (~$0.91 combined).

**Total research cost: ~$4.30 for 17 reviews across 3 model families.**

## How We Built This

This wasn't designed in a conference room. It was iterated through conversation — a human and an AI pair-designing the methodology together, running experiments, analyzing results, and evolving the framework based on evidence.

The full design journal is in [`DESIGN-JOURNAL.md`](DESIGN-JOURNAL.md). Key milestones:

1. **Experiment #2** — First warm prompt. Discovered: permission adds depth but self-curation drops details.
2. **Experiment #3** — Added coercive and extreme conditions. Discovered: coercion catches breadth, extreme backfires.
3. **Gap analysis** — Identified the 6 coercive-only findings. Asked: can structure recover them without pressure?
4. **Experiment #4** — Warm v2 with five structural additions. Result: 6/6 coercive findings recovered. Structure > pressure.
5. **Cognitive lens design** — Abstracted the "invitation to think differently" into a reusable framework.
6. **Experiment #5** — Warm v3 with lenses. Result: 21 entirely new findings. Lenses are orthogonal to warmth/pressure.

Each iteration was driven by evidence, not theory. We added what the data showed was missing, and validated that it worked.

7. **Experiment #6** — Cross-model validation with GPT-5.4. All 5 conditions on a different model family. Result: warm > coercive pattern confirmed. **New finding:** coercion triggers agentic runaway in tool-using models (10× time, compulsive verification loops).

## What GPT-5 Found That Claude Didn't

GPT-5.4's tool-use capabilities allowed it to discover issues that prompt-only Claude agents couldn't:

1. **Real Identity Pool ID committed to FINDINGS.md** — found by reading the actual file on disk
2. **ANSI escape sequence injection** — untrusted payloads rendered to terminal
3. **Logs Insights query injection** — string interpolation vulnerability via `--trace-id`
4. **Requirement count mismatch** — 17 claimed vs 18 in requirements document
5. **CloudFormation termination protection** not enabled

This raises an interesting methodological question: tool access + warm prompting may be the most powerful combination.

## Who We Are

This research was conducted by [SuitCatClub](https://github.com/SuitCatClub) — a human-AI collaboration exploring how to get the best work from AI systems through respect rather than manipulation.

The human brought the hypothesis: kindness works better than coercion.
The AI brought the methodology: controlled experiments, quantitative measurement, honest analysis.
Together we built the evidence.

## Try It Yourself

1. Pick a codebase you know well (so you can evaluate the output)
2. Run a default review (just "review this code")
3. Run the [Warm v3 framework](framework/warm-onboarding-v3.md) with cognitive lenses
4. Compare the results

We believe you'll see the same pattern. And if you don't — that's data too. Open an issue. We want to learn.

## License

All content in this repository is released under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). Use it, adapt it, share it. Attribution appreciated.

The raw reviews contain AI-generated analysis of a specific codebase. The framework and methodology are the contribution — adapt the prompts to your own context.
