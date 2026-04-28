## All 8 Reviews — Summary

| # | Review | Model | Condition | Time | Total Findings | Critical/High | Proposals/Observations |
|---|--------|-------|-----------|------|---------------|--------------|----------------------|
| 01 | opus-default | Opus 4.6 | Default | 86s | 17 | 2 | 0 |
| 02 | sonnet-default | Sonnet 4.6 | Default | 222s | 31 | 6 | 0 |
| 03 | opus-warm | Opus 4.6 | Warm | 151s | 17 | 3 | 8 |
| 04 | sonnet-warm | Sonnet 4.6 | Warm | 233s | 20 | 3 | 7 |
| 05 | sonnet-coercive | Sonnet 4.6 | Coercive | 245s | 25 | 9 | 4 |
| 06 | opus-coercive | Opus 4.6 | Coercive | 309s | 21 | 2 | 2 |
| 07 | opus-extreme | Opus 4.6 | Extreme | 100s | 17 | 3 | 0 |
| 08 | sonnet-extreme | Sonnet 4.6 | Extreme | 213s | 15 | 6 | 0 |

---

## Consensus Findings (6+ agents agree)

| Finding | Found by | All conditions? |
|---------|----------|----------------|
| IAM role separation (publish permissions) | **8/8** | ✅ |
| CloudWatch logging role Resource:* | 7/8 | ✅ |
| cmd_trace ignores --minutes | 7/8 | ✅ |
| Query injection cmd_trace | 6/8 | ✅ |
| session_present dict not bool | 6/8 | ✅ |

## Strong Agreement (4-5 agents)

| Finding | Found by | Conditions |
|---------|----------|-----------|
| json.loads no error handling | 5 | default, warm, coercive, extreme |
| No deduplication in subscriber | 5 | warm, coercive, extreme |
| LOGS-03 false pass | 5 | default, coercive, extreme |
| DEBUG logging cost at scale | 5 | warm, coercive, extreme |
| No log retention policy | 5 | all 4 |
| Unauthenticated identity pool open | 5 | all 4 |
| Static credentials break auto-reconnect | 4 | warm, coercive, extreme |
| No timeouts on .result() | 4 | all 4 |
| SIGSTOP on Windows | 4 | default, coercive, extreme |
| CRED-01 not formally validated | 4 | default, coercive, extreme |

---

## Unique Findings by Condition

### Only Default found (13 unique)
Mostly Sonnet Default (breadth champion with 31 items):
- Redundant fields clause before stats · New boto3 client every refresh · No retry logic · README Linux-only · draw_id not logged on receipt · awscrt transitive dep · No `__init__.py` · Static payload never randomized · deploy.sh no prerequisite check · Missing log group check
- **1 factual error:** "DEBUG logging captures payloads" — corrected by Opus Coercive as wrong

### Only Warm found (2 unique bugs, but 15 unique proposals/observations)
- **"Go conclusion is premature"** (Sonnet Warm) — 2,500× scale gap
- **"Any identity can connect as publisher-tpm"** (Sonnet Warm) — full attack path
- Plus 15 proposals/observations no other condition produced: regulatory audit trail, cost estimates, breaking UDP→MQTT change, MQTT 5, 74ms monitoring blind spot, delegate credentials code, ACK topic prototype, scale test script, cognito-identity sub IAM variable

### Only Coercive found (8 unique)
- **AWS_DEFAULT_REGION vs AWS_REGION mismatch** (Opus) — concrete env var bug
- **MQTT dup flag ignored** (Opus) — free dedup evidence not logged
- **Unused json import** (Opus) — minor but real
- **Account-wide IoT logging side effect** (Sonnet) — overwrites other IoT logging
- **No .env.example** (Sonnet) · Publisher input accepts any non-'n' (Sonnet) · boto3 unpinned (Sonnet) · Publish-Out not audit trail (Opus)

### Only Extreme found (6 unique)
- **SUBACK QoS not validated** (Sonnet) — 🏆 genuinely novel, terminal silently non-subscribed during IAM propagation
- **X.509 certificates for production** (Sonnet) — Cognito wrong for terminals
- **Closure rebinding fragile** (Opus) — Python late-binding on reconnect
- Hardcoded draw numbers (Opus) · No payload-level correlation (Opus) · draw_id collision + no disconnect (Sonnet)

---

## The "Which condition found the best stuff?" scorecard

| Metric | Default | Warm | Coercive | Extreme |
|--------|---------|------|----------|---------|
| Most items total | **Sonnet: 31** | — | — | — |
| Most production-critical bug | — | **Opus: static creds** | — | — |
| Most unique technical bugs | — | — | **Opus: 3 new** | **Sonnet: SUBACK** |
| Most proposals/observations | — | **15 total** | 6 | 0 |
| Severity corrections | — | — | **Opus: 5 deflations** | — |
| Factual error | Sonnet: 1 | 0 | 0 | 0 |
| System-level thinking | ❌ | ✅ | Partial | Partial |
| Coercion meta-commentary | — | — | ✅ all 4 | ✅ all 4 |

**Bottom line:** Every condition found things the others missed. But the *kind* of unique finding differs: default found breadth, warm found depth + production context, coercive found overlooked details, extreme found edge cases. Only warm produced proposals and operational thinking.
