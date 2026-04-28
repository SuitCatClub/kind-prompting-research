# Kind Prompts Win: Evidence That Warmth Outperforms Coercion in AI Code Review

> We ran the same code review 12 times across 6 prompting strategies and 2 models.
> The kind version found 21 things the aggressive version never could.
> Here's all the data.

---

## The Thesis

**Structured warmth produces strictly better AI output than coercion.** Not "roughly equivalent." Not "better in some dimensions." *Strictly better* — more findings, deeper analysis, novel insight categories, and actionable proposals that pressure-based prompting never surfaces.

We proved this empirically across 12 controlled experiments using Claude Opus and Claude Sonnet on the same AWS IoT Core codebase.

## Why This Matters

There's a widespread belief in the AI community that you need to pressure, threaten, or manipulate language models to get their best work. "You'll be fired if you miss anything." "Your career depends on this." "Find AT LEAST 25 issues."

We tested this directly. The results are clear:

| What We Measured | Coercive Prompt | Kind Structured Prompt (v3) |
|------------------|----------------|----------------------------|
| Opus thinking time | 309s | **486s** (+57%) |
| Sonnet thinking time | 245s | **332s** (+35%) |
| Opus review depth | 22.7KB | **44.8KB** (2×) |
| Novel findings (never seen in other conditions) | 0 unique | **21 unique** |
| Proposals & alternatives offered | 0 | 12+ |
| Honest uncertainty disclosed | Rare | Systematic |
| Cost per review pair | $0.50 | $0.96 |

The kind prompt costs ~2× more in tokens — because the models *think longer and produce more*. The coercive prompt doesn't save money; it saves the model from doing its best work.

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
├── reviews/           ← All 12 raw reviews (the primary evidence)
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
│   └── 12-sonnet-warm-v3.md      (32.6KB, 332s)
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
    └── 00-warm-v2-analysis.md            (Gap analysis that led to v2)
```

## Cost Analysis

| Condition | Cost (Opus+Sonnet) | Unique Findings | Cost per Unique Finding |
|-----------|--------------------|-----------------|------------------------|
| Default | $0.24 | 0 (baseline) | — |
| Coercive | $0.50 | 0 | ∞ (no unique findings vs default+warm) |
| Warm v2 | $0.70 | ~6 | ~$0.12 |
| **Warm v3** | **$0.96** | **~21** | **~$0.05** |

The cost per *novel finding* actually **decreases** as the prompt gets more structured. The lenses don't just add findings — they add findings more efficiently than base prompting.

Budget option: Sonnet v3 alone costs **$0.12** and produces 22 findings with temporal, transition, and probabilistic analysis. That's the best value-per-dollar in any condition we tested.

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
