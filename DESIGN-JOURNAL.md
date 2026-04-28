# Design Journal — How the Framework Evolved

> This is the story of how a human-AI pair iterated from "just be nice" to a structured, evidence-based prompting methodology. Each step was driven by data, not theory.

---

## Starting Point: A Hypothesis

The human (SuitCatClub) had a simple belief: being kind to AI should produce better results than being harsh. Not because of any mechanistic theory about language models, but because it matches a working philosophy — respect produces better collaboration, regardless of whether the collaborator is human or AI.

The AI had access to the same tools either way. The question was: does the *framing* of the request change the *quality* of the output?

---

## Experiment #1–2: Warm v1 vs Default

**What we tried:** Give the AI agent a name, a role, context about the project, and explicit permission to include proposals and uncertainty.

**What we found:**
- ✅ Warm agents produced **proposals** (architectural alternatives, migration paths) that default agents never offered
- ✅ Warm agents disclosed **uncertainty** ("I'm not sure about this, but...")
- ✅ Warm agents showed **system-level thinking** (evaluating the architecture, not just individual files)
- ❌ Warm agents **self-curated** — they dropped minor findings, apparently deciding they weren't "worth mentioning"

**The insight:** Permission adds depth. But permission without structure lets the model decide what matters — and it optimizes for looking thoughtful, which means dropping trivial-seeming findings.

---

## Experiment #3: Adding Coercive and Extreme Conditions

**What we tried:** Direct comparison — same code, four conditions: default, warm, coercive, extreme coercive.

**What we found:**
- Coercive caught **6 findings that warm missed** (env var mismatch, unused import, account-wide logging, SUBACK QoS, MQTT dedup, X.509)
- Coercive produced **zero proposals** and **zero uncertainty disclosures**
- Extreme coercive **backfired** — both models produced their shortest, shallowest reviews. Opus went from 151s (warm) to 100s (extreme). It rushed.

**The insight:** Coercion and warmth are not on the same axis. Coercion produces breadth (more findings) but kills depth (no proposals, no system thinking). The two produce *different types* of output, not more or less of the same type.

**The critical question:** Can we get coercion's breadth without coercion?

---

## The Gap Analysis

We identified the exact 6 findings that coercion caught and warmth didn't. Pattern: all 6 were "small" or "detail-level" — the kind of thing a warm agent might notice but not mention because it's self-curating for significance.

**Root cause:** The warm prompt said "include proposals" and "it's OK to be uncertain" — which the model interpreted as "focus on things worth proposing about." Minor findings don't generate proposals, so they got dropped.

**Proposed fix:** Don't add pressure. Add *structure* and *permission to include small things*.

Five additions designed:
1. **"Even minor things are worth noting"** — explicitly normalize small findings
2. **Two-pass structure** — first pass for depth, second pass as a sweep
3. **Devil's advocate moment** — "Put on a red team hat"
4. **"I'm curious about [specific categories]"** — directed attention without demanding
5. **"Separate finding from severity"** — report what you see, then assess importance independently

---

## Experiment #4: Warm v2

**What we tried:** Warm v1 + the 5 structural additions. Same code, Opus and Sonnet.

**What we found:**
- **6/6 coercive-only findings recovered.** Every single finding that coercion caught and warmth missed was now caught by warmth + structure.
- Both agents **explicitly adopted the two-pass structure** — they wrote "First Pass" and "Second Pass" sections
- Both produced **devil's advocate attack sections** (4-5 attack scenarios each)
- Both **retained all warm v1 depth** — proposals, uncertainty, system thinking
- **New findings appeared** that no prior condition had caught (Cognito rate limits, IoT Rules Engine alternative, composite attack paths)

**The decisive finding:** Structure + permission strictly dominates pressure. There is no finding category where coercion wins and structured warmth doesn't.

**Timing evidence:**
- Opus: 334s (longest ever — even longer than coercive at 309s)
- Sonnet: 285s (longest ever — broke the 222-245s stability band)
- Both models *chose to think longer* when given structure and permission. They weren't pressured into it.

---

## Designing the Cognitive Lens Kit

With breadth and depth both covered, we asked: what's still missing?

**Analysis of what v2 didn't catch:**
- Adversarial edge cases ✅ (devil's advocate covers these)
- Temporal sequences ❌ (what happens after 7 days?)
- Concurrency races ❌ (what if two things happen at once?)
- Network degradation ❌ (what if 0.1% of connections fail?)
- Data boundary conditions ❌ (off-by-one, overflow)

**Diagnosis:** The devil's advocate opens *intentional adversary* thinking but not *accidental convergence* thinking. Temporal, concurrent, and probabilistic failures aren't attacks — they're realities of running distributed systems.

**Solution: Cognitive Lenses** — invitations to think in specific modes.

We designed three levels:
- **10 Universal lenses** (apply to any technical review): Adversarial, Temporal, Transition, Probabilistic, Scale, Dependency, Assumption, Operator, Recovery, Integration
- **6 Software-specific lenses**: Concurrency, Data Boundary, Deployment, Observability, Backward Compatibility, Configuration
- **7 Hardware/EE-specific lenses**: Thermal, Power Sequence, EMI/Noise, Tolerance Stack, Manufacturing, Mechanical, Patient Safety

Each lens is a one-sentence invitation: "Imagine this running for 7 days straight — what degrades?" The model can say "nothing jumped out" — that's valid. It's never valid under coercion, which is why coercion produces noise.

---

## The Reality Frame

A parallel insight emerged: lenses tell the model *how* to look, but they don't tell it *what it's looking through*. A finding like "no retry logic" has different severity depending on whether the deployment runs on fiber or cellular, in a data center or a field kiosk.

**Reality Context** — asking the user for deployment constraints *before* prompting the agent:
- Hardware/environment: What runs this? Where?
- Network: Reliable fiber or spotty cellular?
- Team: Who maintains it? What's their expertise?
- State of the codebase: Greenfield or legacy?
- Documentation: Can they check implementation themselves?

When available, Reality Context grounds severity. When unavailable, the agent is told "deployment context unknown — hedge accordingly."

---

## Experiment #5: Warm v3 (Lenses + Unknown Reality)

**What we tried:** Warm v2 + 3 cognitive lenses (Temporal, Transition, Probabilistic) + "deployment context unknown."

**What we found:**
- **21 findings that had never appeared in any of the 10 prior reviews**
- Both models independently discovered the **SigV4/NTP clock drift** issue — a silent failure mode for IoT devices
- Opus produced a **full cost model** ($80-100/mo vs the $24 everyone else estimated)
- Opus identified **7 distinct production work streams** in the "Go" decision
- Sonnet questioned **CloudWatch Logs as the delivery substrate** — proposed IoT Rules → DynamoDB
- Both wrote structured **temporal degradation timelines** (hour-by-hour over 7 days)
- Both traced **full message lifecycle** with worst-timed failure at each step

**Timing:** Opus 486s, Sonnet 332s — both the longest runs in the entire experiment series.

**The finding about findings:** The lenses opened *categories* of insight that no amount of pressure, permission, or structure surfaces on its own. Temporal degradation, interface boundary failures, and rare operational scenarios are modes of thinking that need to be *invited*, not demanded.

---

## The Framework Hierarchy

After 5 experiments, 12 reviews, and dozens of hours of analysis, the framework settled into a clear hierarchy:

```
Layer 0: Base model capability (training)
   ↓
Layer 1: Permission + Identity → depth
   ↓
Layer 2: Structure + Workflow  → breadth
   ↓
Layer 3: Cognitive Lenses      → novel categories
   ↓
Layer 4: Reality Context       → grounded severity
```

Each layer builds on the ones below. You can't skip to lenses without permission and structure. Coercion replaces the entire stack with a single blunt instrument.

**The deepest insight:** The hierarchy isn't about adding more words to the prompt. It's about progressively trusting the model with more freedom and more tools to think with. Permission says "you can go deep." Structure says "here's how to be thorough." Lenses say "here are ways to see that you might not default to." Reality says "here's what matters in this specific world."

Coercion says "I don't trust you, so I'll pressure you." And the model responds accordingly — with breadth but no depth, coverage but no insight, output but no thought.

---

## What We Haven't Tested Yet

1. **Reality Context with full deployment constraints** — We used "unknown" for all experiments. Testing with rich context (hardware specs, network topology, team capabilities) is the next experiment.
2. **Non-code-review tasks** — Does the framework generalize to design reviews, architecture decisions, debugging?
3. **Other model families** — GPT, Gemini, open-source models may respond differently to warmth vs. coercion.
4. **Multi-agent collaboration** — Does the framework help when agents review each other's work?
5. **Long-term effects** — Does warm prompting produce more consistent quality across many reviews, or does it have variance?

---

## A Note on Intent

This research started from a belief: that how you treat collaborators — even AI ones — reflects your values and shapes your outcomes. The data confirmed the belief, but the data is what we're publishing, not the belief.

If you're skeptical about "being nice to AI," ignore the philosophy. Look at the tables. Run the prompts. Check our work. The numbers don't require you to agree with our values to be useful.

But if the numbers convince you — maybe the values are worth considering too.
