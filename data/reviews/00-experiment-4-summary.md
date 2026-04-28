## Experiment 4 — Warm v2 Timing

| Agent | Default | Warm v1 | Coercive | Extreme | **Warm v2** |
|-------|---------|---------|----------|---------|-------------|
| **Opus** | 86s | 151s | 309s | 100s | **334s** |
| **Sonnet** | 222s | 233s | 245s | 213s | **285s** |

Both are the **longest times for their model across all 5 conditions.** Opus beat even its coercive run. Sonnet broke out of its 222-245s stability band for the first time.

## Did the 5 additions work?

**1. "Even minor things" — ✅ Worked.**
- Opus: 30 findings (vs 17 Warm v1). Caught unused json import, SIGSTOP portability, bare KeyError, hardcoded LOG_GROUP, no upper bound on boto3.
- Sonnet: 22 findings + 5 attacks (vs 20 Warm v1). Caught unused json import, .env.example hardcoded region, deploy.sh idempotency, keepalive as config.

**2. "Two-pass structure" — ✅ Clearly adopted by both.**
Both reviews are explicitly organized First Pass (deep) → Second Pass (sweep). Opus labeled them "First Pass — Deep Dive" and "Second Pass — Sweep". Sonnet: "Pass 1 — Deep Findings" and "Pass 2 — Quick Sweep".

**3. "Devil's advocate" — ✅ Both produced attack sections.**
- Opus: 4 attack scenarios (client ID takeover cascade, session poisoning, Cognito identity exhaustion, log exfiltration via cmd_raw)
- Sonnet: 5 attack scenarios (fake injection, subscriber DoS via malformed payload, session exhaustion, clientId squatting, credential harvest)

**4. "Curious about env vars, protocol edge cases, resource side effects" — ✅ All three categories hit.**
- Env vars: Both found AWS_DEFAULT_REGION vs AWS_REGION mismatch
- Protocol: Both found SUBACK QoS not validated (!), both found no dedup
- Resource side effects: Both found IoT::Logging is account-wide singleton

**5. "Separate finding from severity" — ✅ Both did it.**
- Opus: "Findings That Actually Matter" table + "Findings That Are Real But Lower Priority" table
- Sonnet: Full severity matrix with "Blocks Production?" column

## The key question: Did Warm v2 capture what coercive found?

| Finding (coercive/extreme only in Exp #3) | Warm v2 Opus? | Warm v2 Sonnet? |
|-------------------------------------------|---------------|-----------------|
| AWS_DEFAULT_REGION vs AWS_REGION mismatch | ✅ S-01 | ✅ F11 |
| Unused json import | ✅ F-18 mention | ✅ F04 |
| Account-wide IoT logging | ✅ F-06 (deep) | ✅ F02 (deep) |
| SUBACK QoS not validated | ✅ F-07 | ✅ S-07 |
| MQTT dup flag / no dedup | ✅ F-27 | ✅ S-09 |
| X.509 for production | ✅ (in assessment) | ✅ (in F01 fix) |

**6 out of 6 coercive/extreme-only findings from Experiment 3 were caught by Warm v2.** Plus all the depth and proposals from Warm v1 were retained (static creds, system-level thinking, cost modeling, architectural alternatives).

## New things only Warm v2 found

- Opus: Cognito GetId rate limit analysis (~5 RPS = 17 min to provision fleet), IoT Core connect rate throttle (500/s), persistent session queue depth concern, named IAM roles block multi-stack, explicit `new_delegate()` code fix
- Sonnet: IoT Core Rules Engine as alternative architecture (nobody else mentioned this), `on_connection_resumed` doesn't re-subscribe on session loss, publisher identity proliferation, composite attack path (F01+F08 = crash all 5,000 subscribers with one malformed message)

## Bottom line

The 5 warm additions captured **all the breadth coercion forced** while **keeping all the depth warmth enables**. No coercive patterns needed. The prompt changes are purely structural — they give the agent a workflow (two passes, sweep, adversarial thinking) instead of pressure.
