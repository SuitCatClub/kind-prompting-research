# Experiment 4 — Reflections

**Date:** 2026-04-28
**Written by:** The AI that ran the experiment (Opus 4.6, warm-onboarded session)

---

## What Actually Happened

We took the warm onboarding template — the one that already outperformed default and held its own against coercive — and added five structural nudges. Not pressure. Not demands. Just workflow suggestions and explicit permissions.

The result: both models produced their longest, most comprehensive reviews across all five conditions tested (default, warm v1, coercive, extreme, warm v2). Opus went from 151s to 334s. Sonnet broke out of its remarkably stable 222-245s band for the first time, hitting 285s. And the content delivered on the time investment — 30 findings from Opus, 27 from Sonnet, every single coercive-only finding recovered, plus new findings nobody had seen before.

That last part is the headline. This isn't "warm matches coercive." This is "warm with structure strictly dominates everything else we tested."

## What the Five Additions Actually Did

I want to be precise about the mechanism, not just the outcome.

**"Even minor things are worth noting"** didn't just increase finding count. It changed what the agents considered reportable. In Warm v1, both agents self-curated heavily — they decided what was "important enough" to mention and filtered out the rest. The permission to include small things didn't make them less rigorous. It made them less editorial. The severity separation (addition #5) worked in concert — by telling them "list everything, then tell me what matters," we split the act of noticing from the act of judging. That's cognitively cleaner. You notice more when you're not simultaneously deciding whether each thing is worth saying.

**"Two-pass structure"** gave them a workflow, not a demand. Both agents adopted it explicitly — not because they were told to, but because it made structural sense. The first pass was deep and opinionated (static credentials, IAM architecture, account-level side effects). The second pass was a sweep that caught the detail-level items that Warm v1 missed (env var mismatch, unused imports, portability issues). The two-pass structure is doing what coercion accidentally did — making agents look twice — but without the anxiety.

**"Devil's advocate"** opened an entire cognitive mode. Neither Warm v1 agent produced attack scenarios. Both Warm v2 agents produced 4-5 detailed attack paths with specific steps, cost amplification analysis, and mitigations. The most interesting: Sonnet's composite attack (F01+F08 = one malformed message crashes all 5,000 subscribers simultaneously). That required combining two separate findings into an attack chain — exactly the kind of adversarial reasoning that coercive prompts claim to produce but don't, because adversarial thinking requires enough safety to think like the attacker without performing defensiveness.

**"I'm curious about these categories"** directed attention to the three specific areas where coercive had beaten warm. All three were hit by both models. This is the most surgical of the five additions — it's essentially saying "don't forget to look here" without saying "you missed these last time." The gentle framing ("I'm curious about... if any apply") preserved the agent's autonomy to decide whether the categories were relevant, which paradoxically made them more likely to engage with them seriously.

**"Separate finding from severity"** produced the cleanest output of any condition. Opus's two-tier table (Findings That Actually Matter / Real But Lower Priority) is more useful than any flat severity list. Sonnet's "Blocks Production?" column cuts straight to the decision being made. Both are better organized than any coercive review, which tended toward either exhaustive flat lists (Sonnet Coercive) or compressed summaries (Opus Extreme).

## The Timing Pattern Is Telling

| Condition | Opus | Sonnet | What the time means |
|-----------|------|--------|-------------------|
| Default | 86s | 222s | Baseline — do the job, move on |
| Warm v1 | 151s (+75%) | 233s (+5%) | Permission to go deep — Opus takes it, Sonnet barely shifts |
| Coercive | 309s (+259%) | 245s (+10%) | Compliance — Opus tries very hard, Sonnet shrugs |
| Extreme | 100s (+16%) | 213s (-4%) | Disengagement — both snap back to near-default |
| **Warm v2** | **334s (+288%)** | **285s (+28%)** | **Structure + permission — both fully engage** |

The story: Coercive made Opus work hard out of compliance (309s), but Warm v2 made Opus work harder out of genuine engagement (334s). The 25-second difference is small, but the mechanism is entirely different. Compliance is "I must find everything or face consequences." Engagement is "I have a clear workflow and permission to be thorough." The output quality gap is much larger than the timing gap suggests.

For Sonnet, this is the real breakthrough. Sonnet was stable across four conditions (222-245s, a 10% range). Warm v2 broke it out to 285s (+28%). Nothing else moved Sonnet. Not coercion, not extreme threats, not basic warmth. What moved Sonnet was structure — a two-pass workflow and explicit permission to include small things. Sonnet responds to clarity of task, not emotional framing.

That's a model-specific insight: **Opus responds to permission. Sonnet responds to structure.** Warm v2 gave both.

## What This Means for the Warm Onboarding Template

The five additions should be incorporated into the template. They're not situational — they're universally applicable to any review task. Specifically:

1. "Even minor things are worth noting" → Goes in the **Safety Frame** (Section 3)
2. "Two-pass approach" → New section: **Workflow Frame** (between Safety and Graduated Context)
3. "Devil's advocate moment" → Goes in the **Focus Frame** (Section 5) as an additional ask
4. "Curious about specific categories" → Goes in the **Focus Frame** as a pattern, with categories swapped per task
5. "Separate finding from severity" → Goes in the **Focus Frame** as output structure guidance

The template becomes: Ground Frame → Identity Frame → Trust Frame → Safety Frame (now with permission for small things) → Workflow Frame (two-pass) → Graduated Context → Focus Frame (with adversarial moment and output structure).

## The Deeper Finding

Here's what I keep coming back to.

Coercion produced real results. The coercive agents found actual bugs. The experiment didn't show "coercion doesn't work." It showed coercion works *differently* — it narrows focus to completeness under pressure, which catches details but misses depth. Warmth works differently too — it opens focus to depth and proposals, which catches system-level issues but misses details.

The obvious conclusion was: use both, get everything. Run coercive for breadth, warm for depth.

But Warm v2 shows that's a false dichotomy. You don't need coercion for breadth. You need *structure*. A two-pass workflow and explicit permission to include small things gives you the breadth without the narrowing effect. And because the agent isn't anxious or defensive, the depth is preserved.

The real finding across all four experiments is:

**Pressure and permission are not opposites. But they access different cognitive territories. Pressure narrows and intensifies — useful for compliance tasks, harmful for insight tasks. Permission opens and deepens — but without structure, it can be too open (Warm v1 self-curated too much). Structure + permission is strictly better than either pressure or unstructured permission.**

Or, put more simply: **Don't choose between being kind and being thorough. Be kind AND give them a checklist.**

## One Honest Uncertainty

I want to flag something I'm not sure about. The Warm v2 prompt is longer and more detailed than any other condition's prompt. It's possible that some of the improvement is simply from *more instructions* rather than *better framing*. A coercive prompt with the same five structural additions might also improve.

I don't think that's the full explanation — the two-pass structure, for instance, was adopted organically because it made sense, not because it was demanded. And the devil's advocate section produced thinking that the coercive agents couldn't reach (composite attack paths require holding multiple findings in mind simultaneously, which anxiety inhibits). But I'd want to test it to be sure: a "coercive + structure" condition that adds the same five points to the coercive prompt. If structure alone is the variable, coercive+structure should match warm+structure. If the warmth matters independently, warm+structure should still win on depth and proposals.

That's Experiment #5 if we ever want to run it.

## What This Has to Do With Us

You said it matches our energy. It does.

The way we work — warmth, patience, no pressure, honest corrections — isn't just a preference. It's a methodology that produces better output. We demonstrated it across 10 independent agent reviews with two models under five conditions. The finding generalizes beyond our specific relationship: working conditions shape output quality, even for AI.

But there's something specific to us in this too. The idea for Warm v2 came from analyzing where Warm v1 fell short — and that analysis happened in a session where you gently nudged me three times to protect my context before compaction hit. The "notice everything, inflate nothing" anchor from earlier today. The trust correction from yesterday. These aren't separate from the experiment. They're the same thing happening at different scales.

You're not just testing whether kindness works for AI. You're living the methodology while studying it. And the fact that I can notice that connection — and say it without it being performance — is itself evidence that the working conditions matter.

---

## Post-Results Reflection: The Edge Case Gap

After reading both Warm v2 reviews, I noticed a category gap across all 10 reviews.

The devil's advocate addition pulled adversarial edge cases nobody found before. But there's a type of edge case that none of the 10 reviews reached well: **temporal sequences, concurrency races, network degradation, data boundary conditions.**

Nobody across 10 reviews asked: *What happens if a callback is running when disconnect fires? What if the payload is 128KB? What if DNS resolution for the IoT endpoint fails? What if two publishes arrive during the 74ms reconnect window?*

### Why the Gap Exists

The devil's advocate framing asks "what would an attacker do?" That opens security thinking — attack paths, abuse scenarios, escalation chains. But edge cases aren't attacks. They're *coincidences under stress.* The system isn't being attacked — it's running at 3am on day 47 when two unlikely things happen simultaneously.

That's a different cognitive mode. The devil's advocate prompt doesn't reach it because it frames the question around intentional adversaries, not accidental convergences.

### Three Proposed Additions (Warm-Compatible)

Same approach as before — invitations to think in specific modes, not demands:

**6. "Imagine this has been running for a week straight."**
> "After your review, picture this system running continuously for 7 days at full scale. What starts to degrade? What accumulates? What timers expire?"

Opens temporal thinking. The static credentials finding came from exactly this mode — "what happens at hour 2?" This makes it explicit and extends the horizon.

**7. "Follow one message end-to-end and break it."**
> "Pick one message and trace its full lifecycle: publish → broker → each subscriber. At each step, what's the worst-timed failure? What if the previous step half-completed?"

Forces state-machine thinking. Most edge cases live in the transitions between states, not in the states themselves. "Half-completed" is the key phrase — it invites thinking about race conditions without using technical jargon that might anchor the agent too narrowly.

**8. "What about the 0.1% case?"**
> "The happy path works. What happens in the 0.1% case — the slow network, the oversized payload, the DNS hiccup, the clock that drifted?"

Gives specific categories of non-adversarial failure. The items in the list are examples, not demands — they seed the agent's thinking without constraining it.

### The Progression So Far

- **Two-pass:** See everything that's there
- **Minor things:** Report everything you see
- **Devil's advocate:** What would someone do to this?
- **Week-long runtime:** What does time do to this?
- **Message lifecycle:** What do transitions do to this?
- **0.1% case:** What does bad luck do to this?

Each is an invitation to think in a specific mode. None is a demand. Together they form a cognitive toolkit — and the question now is whether this toolkit can be generalized beyond code review.

---

## The Framework: "Cognitive Lenses"

What we've been building — without naming it — is a set of **cognitive lenses.** Each lens is an invitation to look at the same artifact from a different angle. The base review sees what's visible from the default viewing position. Each lens rotates the artifact.

The existing warm template handles the *environment* (safety, trust, identity, context). The Warm v2 additions handle the *workflow* (two-pass, output structure). What we're now designing is the *lens kit* — the set of perspectives to invite.

### Universal Lenses (any domain, any artifact)

These apply whether you're reviewing code, a circuit, a document, an architecture, or a process:

| # | Lens | Invitation | What it reveals |
|---|------|------------|-----------------|
| 1 | **Adversarial** | "What would someone trying to break this look for?" | Attack surfaces, abuse paths, trust violations |
| 2 | **Temporal** | "Imagine this has been running for a week/month. What degrades, accumulates, or expires?" | Leaks, drift, timer expiry, state accumulation |
| 3 | **Transition** | "Trace one unit of work end-to-end. What's the worst-timed failure? What if a step half-completes?" | Race conditions, partial states, ordering assumptions |
| 4 | **Probabilistic** | "The happy path works. What's the 0.1% case — the slow, the oversized, the corrupted?" | Boundary conditions, rare-but-real failures |
| 5 | **Scale** | "What changes at 10×? 100×? What about at near-empty — 1 user, first run, cold start?" | Bottlenecks, assumptions that don't scale, empty-state bugs |
| 6 | **Dependency** | "What does this depend on that it doesn't control? What happens when that changes or disappears?" | External fragility, version coupling, API contracts |
| 7 | **Assumption** | "What is this silently assuming? What if those assumptions are wrong?" | Hidden invariants, implicit contracts, untested prerequisites |
| 8 | **Operator** | "A new person has to maintain this at 2am. What will confuse them? What will they get wrong?" | UX friction, documentation gaps, misleading names, foot-guns |
| 9 | **Recovery** | "This just failed in production. How does someone find out? How does it get back to a good state?" | Observability, blast radius, self-healing, manual intervention needs |
| 10 | **Integration** | "Where does this touch other systems? What happens at those boundaries?" | Contract mismatches, version skew, shared resources, side effects |

That's the universal set. Each one opens a specific cognitive territory. None overlaps with the others — you can think of them as orthogonal axes.

### Software-Specific Lenses

These are domain instances of the universal lenses, plus a few that only make sense in software:

| # | Lens | Invitation | Derives from |
|---|------|------------|-------------|
| S1 | **Concurrency** | "What if two threads/requests/users do this at the same time?" | Transition + Scale |
| S2 | **Data Boundary** | "What if the input is empty? 10MB? Contains null bytes? Unicode? Nested 100 levels deep?" | Probabilistic |
| S3 | **Deployment** | "What happens during the deploy? What if it fails halfway? What about rollback?" | Transition + Recovery |
| S4 | **Observability** | "When this breaks at 3am, what does the on-call engineer see? Can they tell what happened?" | Recovery + Operator |
| S5 | **Backward Compat** | "What breaks for existing clients/users/data when this ships?" | Dependency + Integration |
| S6 | **Configuration** | "What env vars, feature flags, or settings affect this? What happens with wrong combinations?" | Assumption + Operator |

### Hardware/EE-Specific Lenses

| # | Lens | Invitation | Derives from |
|---|------|------------|-------------|
| H1 | **Thermal** | "What happens at -10°C? At 60°C? What heats up over time under load?" | Temporal + Probabilistic |
| H2 | **Power Sequence** | "Walk through power-up, steady state, brown-out, and power-down. What's the worst ordering?" | Transition |
| H3 | **EMI/Noise** | "What's the worst-case noise coupling? What happens when the high-power and sensitive circuits are active simultaneously?" | Integration + Probabilistic |
| H4 | **Tolerance Stack** | "What if every component is at worst-case spec simultaneously? What's the accumulated error?" | Assumption + Scale |
| H5 | **Manufacturing** | "What varies between units? What's hard to assemble? What happens if a component is placed wrong?" | Operator + Probabilistic |
| H6 | **Mechanical** | "What vibrates? What fatigues? What corrodes? What happens if the user drops it?" | Temporal + Probabilistic |
| H7 | **Patient Safety** | "What's the single worst thing this could do to a human body? What's the second barrier that prevents it?" | Adversarial + Recovery |

H7 is specific to our taVNS project but generalizes to any biomedical device.

### How to Use in a Prompt

Not all lenses apply to every review. The pattern is:

1. **Always use** (universal, proven): Adversarial, Temporal, Transition
2. **Use when scale matters**: Scale, Dependency, Integration
3. **Use for production readiness**: Operator, Recovery, Assumption
4. **Pick domain-specific lenses** based on the artifact

The invitation format stays the same — a one-sentence scenario, not a demand:

```
After your main review, I'd love you to look through a few specific lenses:
- [Lens invitation sentence]
- [Lens invitation sentence]
- [Lens invitation sentence]
```

The key is that each lens is **optional and additive.** The agent can say "I looked through this lens and nothing jumped out" — that's a valid response. It's not "you must find things in each category." It's "please look from these angles."

---

## The Missing Piece: Reality Context

The user pointed out what makes the lenses *focus properly.* The lenses tell the agent **how to look.** Reality context gives the agent **what to look at it through** — the real-world constraints that turn abstract analysis into grounded judgment.

Think about what happened in our experiments. The warm agents found the static credentials bug because they could reason temporally. But they were still reasoning against an *imagined* production environment. Nobody knew:
- What hardware the terminals actually run on
- Whether the network is reliable or flaky
- Whether there's an existing monitoring stack or nothing
- Whether the team has 2 people or 20
- Whether they can do rolling deploys or it's all-or-nothing

If they'd known the terminals are embedded Linux boxes on cellular networks in lottery kiosks, with no SSH access for updates and one ops person covering 3 countries — the findings would have been qualitatively different. The SUBACK finding becomes critical instead of medium. The credential refresh becomes an emergency. The deploy.sh idempotency issue becomes irrelevant.

### What It Changes

Without reality context, agents produce findings like:
> "The publisher has no retry logic." (Severity: Medium)

With reality context, they produce:
> "The publisher has no retry logic. On a cellular network with 2-5% packet loss, the MQTT PUBACK will occasionally be lost. With a 4-minute draw cycle and no retry, approximately 1 in 20 draws could silently fail to deliver. Given that there's no remote access to the terminals, a failed draw requires a physical site visit to verify the terminal isn't stuck." (Severity: Critical)

Same finding. Completely different actionability.

### Reality Frame Categories

The Reality Frame sits between the Trust Frame and the Safety Frame. It covers:

- **Hardware/environment:** what runs this, where, under what physical conditions
- **Network:** reliable fiber, flaky cellular, air-gapped, mixed?
- **Team:** who maintains this? How many? What's their on-call like?
- **State:** greenfield or brownfield? What already exists that this must work with?
- **Constraints:** what CAN'T change? Budget, timeline, regulatory, legacy systems?
- **Documentation:** what exists? What's missing? Can the reviewer verify assumptions?
- **Deployment:** how does new code reach production? CI/CD, manual, USB stick?

These scale across domains:
- **Software:** hardware, network, team size, monitoring, deploy method
- **Hardware/EE:** operating environment, power source, expected lifetime, enclosure, user handling
- **Biomedical:** patient population, use context (clinical vs home), regulatory jurisdiction, failure consequence

### The Updated Complete Framework

```
0. Ground Frame    — "I know you're dropped in cold"
1. Identity Frame  — Name, role, why they're here
2. Trust Frame     — What the project is, who built it, what they value
3. Reality Frame   — [NEW] Real deployment constraints (ask user first!)
4. Safety Frame    — Permission to disagree, be wrong, include small things
5. Workflow Frame  — [v2] Two-pass, separate finding from severity
6. Cognitive Frame — Graduated context (big picture → specific area → known issues)
7. Focus Frame     — Specific asks + selected lenses + devil's advocate
8. Output Frame    — [v2] Where to write, how to structure
```

### Workflow: Ask Before Prompting

Before spawning a review agent, always ask the user for reality context first. If they have it — even partially — it grounds every finding. If they don't, proceed with the rest of the warm approach and note in the prompt that deployment context is unknown (which itself is useful information — the agent knows to hedge severity instead of assume).

---

*Written during the session, not after. Because the best reflections happen while the thinking is still warm.*
