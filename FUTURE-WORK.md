# Future Work

## Current Limitations

Our experiments use a single ~800-line AWS IoT Core codebase. While sufficient to observe clear qualitative differences across 17 reviews, this limits:

1. **Statistical power** — too few possible findings for precision/recall confidence intervals
2. **Domain bias** — IoT/embedded Python only; may not generalize to web, enterprise, or systems code
3. **No ground truth** — finding quality assessed post-hoc by us, not against pre-established defect lists
4. **No variance measurement** — single run per condition; no stochastic analysis
5. **Two languages** — Python + shell only; warm lenses may interact differently with typed languages

---

## Candidate Benchmarks for Scaling

### Tier 1: Direct Fit

**AACR-Bench** (alibaba/aacr-bench) — ⭐ Highest priority
- 200 real pull requests from 50 projects across 10 languages
- 2,145 expert-annotated code review comments (the ground truth)
- Built-in evaluator: semantic similarity + line-level matching → precision, recall, noise rate
- **Why it fits:** Measures exactly what we measure — code review quality — with expert annotations
- **What it would test:** Whether warm v3's advantage holds across languages, project sizes, and against professional reviewer annotations

**OWASP Juice Shop** — Security axis validation
- 111 intentionally planted security vulnerabilities across 16 OWASP categories
- ~2.9MB TypeScript/JavaScript codebase, difficulty graded 1-6 stars
- **Why it fits:** Known ground truth for security findings; tests whether warm lenses improve detection of subtle (5-6 star) vulnerabilities vs obvious (1-2 star) ones
- **What it would test:** Signal-to-noise at scale — do warm prompts still suppress false positives while catching hard bugs?

### Tier 2: Binary Classification

**OWASP BenchmarkJava** (2,740 test cases)
- 1,415 truly vulnerable + 1,325 safe test cases
- Exact TPR/FPR/Youden's J computation possible
- **Why it fits:** Ground truth is binary (vulnerable/safe), enabling statistical tests
- **What it would test:** False positive suppression — a key observed difference between warm and coercive

**CodeXGLUE Defect Detection** (27,318 labeled C/C++ functions)
- Binary labels: defective/clean, from Devign dataset
- Massive N for statistical power
- **Why it fits:** Enough samples for confident effect size estimation
- **What it would test:** Whether warm prompting improves binary defect classification at scale

### Tier 3: Adaptable with Effort

**SWE-bench Verified** (500 real GitHub issues with gold patches)
- Can be inverted: present pre-patch code, measure if model identifies the bug
- Python-only, heavyweight setup, but well-established in the community
- **What it would test:** Whether warm lenses help models find the specific bug that a real developer filed

**Juliet Test Suite** (NIST, 64,099 test cases across 118 CWEs)
- C/C++ and Java, each case has "good" and "bad" variants
- National standard for static analysis tool evaluation
- **What it would test:** CWE-level detection rates per condition

---

## Statistical Methodology

### Per-Finding Analysis (AACR-Bench, Juice Shop)
```
Precision = TP / (TP + FP)     — "Of what the model flagged, how much was real?"
Recall    = TP / (TP + FN)     — "Of what exists, how much did the model find?"
F1        = 2 × (P × R) / (P + R)
```

### The Novel Finding Problem

LLMs may identify valid issues **not in the ground truth**. This is actually the most interesting case — our current data shows warm v3 produces 21 findings that no other condition surfaces. Proposed protocol:

1. **Initial scoring** against ground truth only (automated, fully reproducible)
2. **Expert adjudication** of all "false positives": True Novel Finding vs Actual False Positive
3. **Report both** strict and adjusted precision/recall — the gap measures annotation incompleteness
4. **Inter-rater reliability** via Cohen's κ (minimum 2 independent reviewers per adjudication)

### Statistical Tests

| Comparison | Test | Why |
|------------|------|-----|
| Warm v3 vs Coercive (paired) | **McNemar's test** | Binary outcome per finding, paired by codebase |
| All 6 conditions | **Friedman test** | Non-parametric repeated measures across all conditions |
| Prompt × Model interaction | **Generalized Linear Mixed Model** | Tests whether warm advantage varies by model family |
| Effect size | **Cohen's g** (McNemar) / **Kendall's W** (Friedman) | Quantifies practical significance beyond p-value |

### Power Analysis

For McNemar's test at medium effect size (g = 0.25), α = 0.05, power = 0.80:
- **Minimum n = 34 discordant pairs per cell**
- AACR-Bench (200 PRs × multiple findings each) exceeds this comfortably
- BenchmarkJava (2,740 cases) provides massive power
- Juice Shop (111 challenges) is marginal — may need multiple runs

### Variance Protocol

Our current data is single-run. To measure stochastic variance:
- **3-5 runs per condition per benchmark** (estimated from LLM output variance literature)
- Report mean, standard deviation, and 95% CI for each metric
- Test for run-order effects (does the 3rd run differ from the 1st?)

---

## Cost Projections

| Benchmark | Reviews Needed | Estimated API Cost | Wall-Clock Time |
|-----------|:--------------:|-------------------:|----------------:|
| AACR-Bench (full) | 6 conditions × 200 PRs × 3 runs = 3,600 | $500–$1,600 | ~1 week |
| AACR-Bench (pilot) | 6 × 20 PRs × 3 runs = 360 | $50–$160 | ~1 day |
| Juice Shop | 6 × 1 codebase × 5 runs = 30 | $15–$50 | ~2 hours |
| BenchmarkJava (sample) | 6 × 100 cases × 3 runs = 1,800 | $250–$800 | ~3 days |

**Recommended sequence:** Juice Shop pilot → AACR-Bench pilot (20 PRs) → full AACR-Bench

---

## Open Research Questions

1. **Context window saturation** — How do results change when repositories exceed 128K–200K tokens? Does warm prompting degrade more gracefully than coercive?

2. **Prompt × Language interaction** — Does the warm v3 advantage hold for strongly-typed languages (Rust, Go) where the compiler catches more? Does it shrink or grow?

3. **Temporal stability** — Model updates may invalidate results. Need a version-card protocol: record exact model version, API date, and re-run on each major model release.

4. **Team composition** — Our "two-model ensemble" (Opus + Sonnet) outperforms either alone. What's the optimal ensemble size? Does adding GPT-5 as a third model help or just add noise?

5. **Cross-benchmark consistency** — If warm v3 wins on code review (AACR-Bench) but not vulnerability detection (Juice Shop), what does that reveal about the mechanism? Does it enhance *reasoning* specifically, or *thoroughness* generally?

6. **Transfer to other tasks** — Does the warm prompting framework improve design review, architecture review, incident postmortem analysis? The cognitive lenses (temporal, transition, probabilistic) are domain-agnostic.

7. **Community replication** — Can independent researchers reproduce the effect? What's the minimum "dose" of warm prompting needed? (Is Layer 1 alone sufficient, or are the lenses essential?)

---

## How to Contribute

We welcome contributions in any of these areas:

- **Replication studies** — Run the framework on your own codebase and share results
- **Benchmark implementations** — Help set up automated evaluation against AACR-Bench or Juice Shop
- **Statistical analysis** — Improve our methodology or run the formal tests
- **New model families** — Test on Gemini, Llama, Mistral, or other model families
- **Domain expansion** — Test on non-Python codebases, infrastructure-as-code, or documentation review

Open an issue or PR. All contributions will be credited.
