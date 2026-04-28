## The Gap Pattern

Warm's 15 unique findings are all **depth** — proposals, production context, system thinking. Coercive/extreme's uniques are **detail** — env var mismatches, unused imports, SUBACK QoS, account-wide side effects.

**What's happening:** Warmth gives the agent permission to *prioritize*. That opens the proposal space (only warm produced operational thinking), but it also means the agent self-curates — small things get filtered as "not important enough to mention." Coercive agents don't have that filter. They report everything because they're afraid of missing anything.

We want the breadth without the fear.

## 5 Warm Prompt Improvements

**1. Normalize small findings explicitly**
> "Even minor things — unused imports, env var naming mismatches — are worth noting. Don't filter them out because they seem small."

Why: Warm agents skipped 8 real findings coercive caught. Not because they couldn't see them — because they judged them "not worth your time." Give explicit permission to include the small stuff.

**2. Add a two-pass structure**
> "First pass: go deep on whatever catches your attention. Second pass: quick sweep for anything you skipped."

Why: Warm Opus spent +75% more time than default and found the deepest bug (static creds / temporal reasoning). But it stopped once it felt "done." A second-pass nudge isn't pressure — it's workflow.

**3. Ask for a devil's advocate moment**
> "After your review, spend a moment thinking: what would someone *trying to break this* look for?"

Why: SUBACK QoS validation (extreme's best find) is an adversarial-thinking find. Warm agents think like collaborators. One sentence can open that lens without changing the tone.

**4. Explicitly invite edge-case categories**
> "I'm also curious about: environment variable handling, protocol-level edge cases, and cloud resource side effects — if any apply."

Why: The 3 categories where coercive beat warm. Not a demand — a "I'm curious about." Directs attention without threatening.

**5. Separate the severity judgment from the finding**
> "List everything you notice, then tell me which ones actually matter. I'd rather filter myself than miss something."

Why: Opus Coercive *deflated* 5 severity ratings from prior reviews — meaning it had better calibration than default. But warm agents pre-filter. This tells them: finding and ranking are separate acts.

## What NOT to Do

- Don't add urgency ("please be thorough" already implies pressure)
- Don't add quantity targets ("find at least 20 things" is coercive in disguise)
- Don't add competition framing ("previous reviewers missed..." — that's predecessor shaming)
- Don't restrict uncertainty ("no maybes" killed extreme's ability to reason about unknowns)

## Expected Impact

If we ran Experiment #4 with these additions:
- **Keep:** 15 unique proposals, temporal depth, production context
- **Gain:** ~6-8 of the detail findings currently only caught by coercive
- **Lose:** Nothing — none of these restrict the cognitive territory warmth opens
