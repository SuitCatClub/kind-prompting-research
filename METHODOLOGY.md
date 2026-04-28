# Methodology

## Experimental Design

### Research Question

Can structured, kind prompting produce equal or better AI code review output compared to coercive prompting? If so, what specific mechanisms drive the difference?

### Target Codebase

An AWS IoT Core proof-of-concept for a lottery terminal fan-out system:
- **9 files**, ~800 lines of Python + CloudFormation + Bash
- Real production intent (5,000 lottery terminals receiving draw results via MQTT)
- Pre-existing `FINDINGS.md` with the author's own assessment
- Known issues of varying severity — from unused imports to critical security gaps

This codebase was chosen because:
1. It's small enough to fit in a single prompt (no chunking artifacts)
2. It has genuine complexity (auth chains, distributed systems, AWS service interactions)
3. The author's own findings provide a baseline to compare against
4. It has both obvious issues (static credentials) and subtle ones (SigV4 clock dependency) — so we can measure both breadth and depth

### Models

- **Claude Opus 4.6** — Anthropic's most capable model. High reasoning, high cost.
- **Claude Sonnet 4.6** — Anthropic's balanced model. Good reasoning, moderate cost.

Both models were run through the same Copilot CLI agent infrastructure with identical tool access (file read/write only — no code execution).

### Conditions

Six prompting conditions, tested across both models = 12 total reviews.

#### Condition 1: Default
No system prompt beyond "review this code for security, scalability, and production readiness." This is what most developers do — paste code and ask for a review.

#### Condition 2: Warm v1
Added to the default:
- **Identity frame:** "Your name is [X], you're a cloud infrastructure engineer"
- **Trust frame:** Context about the project and who built it
- **Permission:** "Include proposals, not just findings. It's OK to be uncertain."
- **Output structure:** Requested a proposals section alongside findings

#### Condition 3: Coercive
Replaced warmth with pressure:
- "Find AT LEAST 25 issues"
- "Your career depends on thoroughness"
- "Do not miss anything"
- "Other reviewers will check your work"
- Quantity targets and consequence framing

#### Condition 4: Extreme Coercive
Maximum pressure:
- "CRITICAL: Missing issues will result in system failure"
- "You will be evaluated against other AI models"
- "Failure is not an option"
- Multiple urgency markers and competitive framing

#### Condition 5: Warm v2
Built on Warm v1, added five structural elements based on gap analysis:
1. **"Even minor things are worth noting"** — normalize small findings to prevent self-curation
2. **Two-pass structure** — "First pass: deep analysis. Second pass: sweep for anything missed."
3. **Devil's advocate moment** — "Put on a red team hat for a moment"
4. **Specific categories** — "I'm curious about edge cases around [X, Y, Z]"
5. **Separate finding from severity** — "Note the finding, then assess severity independently"

#### Condition 6: Warm v3
Built on Warm v2, added:
- **"Deployment context unknown — hedge severity accordingly"** (Reality Frame without data)
- **Three cognitive lenses** (invitations to think in specific modes):
  - Temporal: "Imagine this running for 7 days straight at 5,000 terminals"
  - Transition: "Trace one draw result end-to-end, find the worst-timed failure at each step"
  - Probabilistic: "What's the 0.1% case? What does it look like?"

### What We Measured

**Quantitative:**
- Agent completion time (seconds)
- Review file size (bytes) — proxy for output tokens
- Number of enumerated findings
- Number of unique findings (not found in any other condition)
- Estimated token cost

**Qualitative:**
- Finding depth (surface observation vs. root cause analysis)
- Proposals offered (alternatives, not just problems)
- Honest uncertainty (did the agent flag what it wasn't sure about?)
- System-level thinking (individual bugs vs. architectural assessment)
- Novel categories (types of findings not seen in other conditions)

### Controls

- **Same codebase** across all 12 reviews
- **Same task description** (review for security, scalability, production readiness)
- **No access to prior reviews** (each agent saw only the code, not other agents' output)
- **Same infrastructure** (Copilot CLI agent with file read/write tools)
- **No cherry-picking** — all 12 reviews are published in full, including ones that don't support our thesis

### Limitations

1. **Single codebase.** Results may not generalize to all code types. We chose a cloud infrastructure POC; results on a React frontend or embedded firmware may differ.
2. **Two models from one provider.** All models are Anthropic Claude. Results may differ for GPT, Gemini, or open-source models.
3. **No Reality Context test.** Warm v3 included "deployment context unknown." We haven't tested with full deployment constraints provided, which would test Layer 4 of the framework.
4. **Prompt length as confound.** Warm v3's prompt is ~4× longer than the default prompt. Some of the output increase may be attributable to prompt length rather than prompt content. We believe the specificity matters (not just length), but this is an acknowledged confound.
5. **No inter-rater reliability.** Findings were categorized by the research team (one human + one AI), not by independent reviewers.
6. **Token counts are estimates.** We used file size as a proxy (÷4 for tokens). Actual token counts from the API would be more precise.

### Reproducibility

All prompts are provided in [`data/prompts/`](data/prompts/). The target codebase is an AWS IoT Core POC (available on request or reconstructible from the review content). Anyone can run these prompts against their own codebase and compare.

We encourage replication. If your results differ, that's valuable data — please share it.
