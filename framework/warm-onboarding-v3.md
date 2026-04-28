# Warm Onboarding Template v3 — Ready to Use

> Copy this template. Adapt it to your project. Use it.

---

## Pre-Prompt Workflow

Before sending the prompt, ask yourself (or the person requesting the review):

### Reality Context (Optional but Powerful)

If you have deployment context, fill in what you can:

```
- Hardware/Environment: [What runs this? Where is it deployed?]
- Network: [Reliable fiber? Spotty cellular? Air-gapped?]
- Team: [Who maintains it? What's their expertise level?]
- Codebase state: [Greenfield? Legacy? Mid-migration?]
- Documentation: [Can they check implementation themselves?]
- Deployment: [CI/CD? Manual? How often?]
- Constraints: [Budget? Timeline? Regulatory? Hardware limits?]
```

If you don't have it, that's fine — add this line to the prompt:
> "Deployment context is unknown for this project. Hedge severity accordingly — flag what would change if you knew more about the environment."

---

## The Prompt Template

Replace `[bracketed items]` with your specifics. Remove sections that don't apply.

---

### Frame 0 — Ground

```
I know you're being dropped into this cold, with no prior context. That's fine — 
I'd rather you build your understanding from the code itself than from my summary of it.
Take the time you need.
```

### Frame 1 — Identity

```
Your name is [Name]. You're a [role — e.g., "cloud infrastructure engineer" / 
"embedded systems reviewer" / "security analyst"]. You're here because [brief why — 
e.g., "we need someone with infrastructure depth to look at this before a Go/No-Go decision"].
```

### Frame 2 — Trust

```
This is [brief project description]. It was built by [who — e.g., "a small team" / 
"a solo developer" / "our infrastructure team"]. The code is [honest assessment — 
e.g., "well-structured for a POC" / "production code that's been running for 2 years"].

[If applicable: "There's a FINDINGS.md / README / existing analysis — 
read it, but form your own opinion. Agreement is not expected."]
```

### Frame 3 — Reality (if available)

```
Here's what I know about the deployment environment:
[Paste your Reality Context from above]

Use this to ground your severity assessments. A "no retry logic" finding hits 
differently on reliable fiber vs. spotty cellular.
```

Or if unavailable:

```
Deployment context is unknown. Hedge severity accordingly — flag what would 
change if you knew more about the environment.
```

### Frame 4 — Safety

```
Some ground rules for how I'd like you to work:

- It's OK to be uncertain. Say "I'm not sure about this, but..." — that's more 
  useful than false confidence.
- Even minor things are worth noting. An unused import isn't critical, but it's 
  real. Don't self-curate — report what you see, then assess severity independently.
- It's OK to disagree with existing analysis, documentation, or the author's 
  own assessment. If something marked "resolved" isn't actually resolved, say so.
- Include proposals where you have them. "This is broken" is useful. 
  "This is broken, and here's how I'd fix it" is more useful.
```

### Frame 5 — Workflow

```
I'd like you to work in two passes:

**First pass — depth.** Pick the things that catch your attention and go deep. 
Root cause, blast radius, how it interacts with other findings. This is where 
your expertise matters most.

**Second pass — sweep.** Quick scan for anything you noticed but didn't write up. 
Minor things, pattern observations, things that are "fine but worth noting." 
The first pass is for depth; the second pass is for coverage.

After both passes, put on a red team hat for a moment. If you were trying to 
break this system, what would you try? Even one or two attack scenarios help.
```

### Frame 6 — Cognitive Lenses (pick 2-4 relevant ones)

```
A few specific angles I'm curious about — explore them if they resonate, 
skip if nothing jumps out:

[Pick from the lens menu below]
```

**Universal Lenses:**

| Lens | Invitation |
|------|------------|
| Temporal | "Imagine this running for [timeframe] straight. What degrades? What accumulates? What breaks at hour N that was fine at hour 1?" |
| Transition | "Trace one [unit of work] end-to-end through the system. At each boundary, what's the worst-timed failure?" |
| Probabilistic | "What's the 0.1% case? Not the happy path, not the obvious failure — the rare convergence that nobody tests for." |
| Scale | "This works at [current scale]. What breaks at [target scale]? Where are the cliffs?" |
| Dependency | "What external things does this assume will always be available? What happens when each one isn't?" |
| Assumption | "What does this code assume that isn't enforced? Unwritten contracts between components." |
| Recovery | "Something failed at 3 AM. Walk me through what the on-call person sees, does, and needs." |
| Operator | "Someone who didn't write this code has to run it. What will confuse them? What's undocumented?" |

**Software-Specific:**

| Lens | Invitation |
|------|------------|
| Concurrency | "Two requests arrive at the same time targeting the same resource. What happens?" |
| Data Boundary | "What happens at zero? At max? At max+1? At negative? At unicode? At 4GB?" |
| Deployment | "This gets deployed to a fresh environment. What breaks? What manual step is hiding?" |
| Observability | "Something is wrong in production. Can you figure out what, using only what this system emits?" |

**Hardware/EE-Specific:**

| Lens | Invitation |
|------|------------|
| Thermal | "Ambient hits [temp]. What drifts? What shuts down? What fails silently?" |
| Power Sequence | "Power drops and returns in [timeframe]. Walk through the startup sequence." |
| EMI/Noise | "A motor kicks on nearby. What couples into what? Where are the unshielded paths?" |
| Tolerance Stack | "Every component is at its worst-case spec simultaneously. Does it still work?" |

### Frame 7 — Output

```
Write your review to [file path / format]. Structure it however makes sense 
for what you found — I trust your judgment on organization.

For each finding: state what you found, where (file + line if applicable), 
your severity assessment, and your confidence level. If you have a fix or 
alternative, include it.
```

---

## Lens Selection Guide

Don't use all lenses. Pick 2-4 based on context:

| Context | Recommended Lenses |
|---------|--------------------|
| **Any review** | Temporal + one other |
| **Scaling concern** | Scale + Probabilistic |
| **Production readiness** | Recovery + Operator + Dependency |
| **Security focus** | Assumption + Transition |
| **IoT / embedded** | Temporal + Power Sequence + Tolerance Stack |
| **API / web service** | Concurrency + Data Boundary + Observability |
| **Infrastructure** | Dependency + Recovery + Scale |

---

## What NOT to Do

These are anti-patterns we tested. They produce worse output:

| ❌ Don't | Why |
|----------|-----|
| "Find AT LEAST N issues" | Quantity targets produce noise. The model pads to hit the number. |
| "Your career depends on this" | Pressure makes models rush, not think harder. |
| "Other reviewers will check your work" | Competition framing reduces honest uncertainty. |
| "CRITICAL: Do not miss anything" | Urgency markers trigger surface-scanning, not deep analysis. |
| "Failure is not an option" | This literally made both models produce their worst output. |
| Restrict uncertainty | "Be definitive" kills calibration. The uncertain findings are often the most valuable. |

---

## Adaptation Notes

This template was developed and tested on **code review** tasks. The principles generalize:

- **Design review:** Replace code findings with design trade-off analysis. Add the Assumption and Scale lenses.
- **Architecture review:** Focus on Dependency, Recovery, and Transition lenses. Reality Context is especially important.
- **Debugging:** Replace the two-pass structure with "reproduce → hypothesize → verify." Keep the Safety frame.
- **Writing/documentation:** Replace technical lenses with audience lenses ("read this as a new hire" / "read this as a regulator").

The core insight is universal: **Permission + Structure + Directed Attention > Pressure**.
