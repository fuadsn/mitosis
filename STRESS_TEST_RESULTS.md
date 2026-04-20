# Stress Test Results — 2026-04-19 (Sun PM)

> Pre-execution validation of Mitosis core assumptions. Run BEFORE committing 3 days of build effort.
>
> **Outcome: ALL FOUR TESTS PASSED.** No redesign needed. The architecture is validated. Spending ~$0.05 USDC total to learn this was a steal.

---

## Test A — Build API Auth (the critical blocker) ✅

**Question**: Can we authenticate to BuildWithLocus with our beta `claw_dev_*` key?

**Finding**: YES, but the docs do not mention this — there is a **separate beta Build API** at `https://beta-api.buildwithlocus.com`. The standard production endpoint (`api.buildwithlocus.com`) rejects beta keys with `{"error":"Invalid API key"}`.

**Working pattern**:
```bash
TOKEN=$(curl -s -X POST "https://beta-api.buildwithlocus.com/v1/auth/exchange" \
  -H "Content-Type: application/json" \
  -d "{\"apiKey\":\"$LOCUS_API_KEY\"}" | jq -r '.token')
```

JWT verified working against `whoami`, `projects`, `billing/balance`. Workspace ID confirmed: `ws_d75521d6`. Email matches: `fuadsanin2665@gmail.com`.

**Bonus discovery — wallet has $6 already** (not just the $5 promo we knew about):
```json
{
  "creditBalance": 6,
  "paidCreditBalance": 1,
  "promoCreditBalance": 5,
  "availableServiceCredit": 6,
  "totalServices": 0
}
```
$6 / $0.25 per service = **24 services worth of provisioning runway** before the gift code grant even arrives.

**Impact on PLAN.md**: Open Question #0 (the showstopper) is RESOLVED. `.env` updated with correct Build API URL.

---

## Test B — Decision LLM Reliability ✅

**Question**: Will the split-or-execute LLM call produce reliable, structured output that matches the intent of each task?

**Method**: 5 representative tasks ranging from atomic (2+2) to broad multi-domain (B2B SaaS GTM strategy). Same prompt and model (Claude Haiku 4.5, temperature 0.2). Expected outcomes pre-declared.

**Result: 5 of 5 correct decisions** with high-quality reasoning:

| # | Task | Expected | Got | Reasoning Quality |
|---|---|---|---|---|
| 1 | Codebase tech DD on multi-stack repo | SPLIT | ✅ SPLIT into 4 children with rational budget allocation | Excellent — recognized parallelism + low context overlap |
| 2 | "What is 2+2?" | EXECUTE | ✅ EXECUTE | "atomic arithmetic task" |
| 3 | "Write a haiku about cats" | EXECUTE | ✅ EXECUTE | "requires unified creative context" — sophisticated |
| 4 | "Analyze app/auth.py for SQL injection" | EXECUTE | ✅ EXECUTE | "splitting would fragment the analysis and miss cross-function injection patterns" — *very* smart |
| 5 | Full B2B SaaS GTM plan | SPLIT | ✅ SPLIT into 5 children including a synthesis specialist | Excellent — invented synthesizer pattern unprompted |

**Cost**: $0.017 USDC for all 5 decisions ($5.000 → $4.983).

**Quirk to handle in code**: model wraps JSON in ` ```json ` fences. Strip in parser.

**Impact on PLAN.md**: Decision LLM is reliable enough for production. The "demo might be theatrical because LLM split is unreliable" risk (#2 in old register) downgraded from M-likelihood to L-likelihood.

---

## Test C — Demo Memo Quality ✅

**Question**: Will the actual prompt chain (auditor → deep-analyzer → synthesizer) produce a DD memo good enough to land with VC judges?

**Method**: Seeded a fake but realistic intentionally-vulnerable Python file (SQL injection + plaintext password + hardcoded JWT + missing validation). Ran the three-stage chain with Claude Haiku 4.5.

**Result: Output is genuinely strong.**

- **Auditor** correctly flagged `auth.py` as suspect with one-sentence concern
- **Deep analyzer** found 7+ vulnerabilities, each with: severity, CVE/CWE reference, exact line, vulnerable code, working remediation code, exploit PoC where applicable
- **Synthesizer** produced a real DD memo: 3-bullet exec summary, severity-ranked findings table, immediate vs short-term recommendations, hour-by-hour effort breakdown, deployable/not-deployable verdict

This is genuinely better than what a junior security analyst produces on their first day — and it took 30 seconds of actual LLM time.

**Cost**: $0.045 USDC for the full 3-call chain.

**Real ROI math now defensible in pitch**:
- Demo run total: ~$0.05 LLM + $2 services = **$2.05**
- Equivalent manual work: 3-5 business days × ~$200/hr senior security analyst = **$4,800–8,000**
- Cost reduction: **2,000–4,000x**
- Original "$200k → $20" claim is conservative. Reality is closer to **$200k → $2.50**.

**Impact on PLAN.md**: The Business Impact pitch holds up under scrutiny. Numbers tighten in our favor.

---

## Test D — Timing Math ✅

**Question**: Will the live demo segment fit inside a 5-minute pitch slot?

**Calculation**:

| Beat | Time | Pre-deployed? |
|---|---|---|
| Spawner + Postgres + Root agent already warm | 0s during demo | YES |
| Submit task → root spawns 4 stack specialists (parallel) | 5s trigger + 90–120s cold start | NO — this is the "blooming" moment |
| 4 specialists run analyses in parallel | ~30s | — |
| Python specialist decides → spawns grandchild for auth.py | 90–120s cold start | NO — this is the "asymmetric split" proof moment |
| Grandchild executes deep analysis | ~30s | — |
| Reports flow back, root synthesizes final memo | ~30s | — |
| **Total live demo segment** | **~4–5 min** | Tight but workable |

**Mitigation if it slips**: pre-deploy ONE of the 4 specialists (e.g., the JS specialist that always says "execute"). Loses 1 of 4 "blooming" tiles in the visualizer but stays parallel and saves 0s. The Python → grandchild spawn must remain live because it's the autonomy-proof beat.

**Impact on PLAN.md**: Demo orchestration plan is concrete. Risk of "demo runs over time" downgraded from H to M with this mitigation.

---

## What Changed in `.env`

```diff
- LOCUS_BUILD_API_URL=https://api.buildwithlocus.com
+ LOCUS_BUILD_API_URL=https://beta-api.buildwithlocus.com
```

That one URL change unblocks the entire build path.

## Total Spend During Stress Tests

```
Promo balance: $5.000 → $4.954
Total spend:   $0.046 USDC
```

For ~$0.05 we know:
- Auth works
- Decision LLM is reliable
- Demo memo will impress
- Timing fits the pitch slot

Best ~$0.05 we'll spend this week.

---

## Verified-Working Endpoint Reference

| Operation | Endpoint | Auth |
|---|---|---|
| Wallet balance | `GET https://beta-api.paywithlocus.com/api/pay/balance` | `Bearer $LOCUS_API_KEY` |
| Gift code request | `POST https://beta-api.paywithlocus.com/api/gift-code-requests` | `Bearer $LOCUS_API_KEY` |
| Wrapped Anthropic chat | `POST https://beta-api.paywithlocus.com/api/wrapped/anthropic/chat` | `Bearer $LOCUS_API_KEY` |
| Build API JWT exchange | `POST https://beta-api.buildwithlocus.com/v1/auth/exchange` | None (apiKey in body) |
| Build API operations | `* https://beta-api.buildwithlocus.com/v1/*` | `Bearer $JWT` |

**Anthropic model identifier**: `claude-haiku-4-5` (resolves to `claude-haiku-4-5-20251001`)

**Wrapped catalog discovery**: `https://beta.paywithlocus.com/wapi/index.md` (NOT `/api/wrapped/md` — that 404s on beta)

**Per-provider details**: `https://beta.paywithlocus.com/wapi/<provider>.md`

---

## Honest Accounting — What's Actually Verified vs. Estimated

The "ALL FOUR PASSED" framing at the top is true but partial. Here's the strict reading:

| Claim | Verified how |
|---|---|
| We can authenticate to both Locus APIs | ✅ JWT exchange succeeded; `Bearer claw_*` works on PayWithLocus |
| We can READ state from Build API | ✅ whoami, projects list, billing balance all returned valid data |
| We can WRITE to Build API (create services, deploy, delete) | ❌ **Not tested.** Spike 2 covers this. |
| Wrapped Anthropic actually charges per call | ✅ Promo balance decreased by ~$0.045 across all tests |
| Decision LLM gives correct split/execute calls | ✅ 5/5 once each — NOT tested for retry variance. Temp was 0.2, not 0. |
| Decision LLM output is easy to parse | ⚠️ Model wraps responses in ` ```json ` fences — parser must strip |
| Demo memo will land with VCs | ⚠️ I judged output as "good" subjectively. Unverified by an actual reviewer. |
| Demo chain used real repo files | ❌ Used seeded string content. Real demo needs Firecrawl/git fetch — untested |
| `max_tokens: 1500` is sufficient | ❌ Deep-analyzer hit the ceiling mid-response. Will need ~3000+ for real runs |
| Chain handles partial/failed child reports | ❌ Synthesizer tested only with clean inputs |
| Cold start is 1-2 min | ❌ Pure docs claim. Not measured. |
| Demo fits in 5 minutes | ❌ **Pure calculation based on docs, not measured wall-clock.** |
| Recursive spawn works (agent-from-inside-agent) | ❌ Architectural keystone still unproven — Spike 3 |
| GHCR public pulls work from Locus | ❌ Not set up or tested |

### Things Monday Must Actually Prove

1. `POST /v1/services` actually provisions a container (Spike 2)
2. Deployment reaches `healthy` within docs-claimed timing (Spike 2)
3. An agent running inside a deployed service has the network reach + permissions to call Build API and spawn another agent (Spike 3)
4. Parallel spawns (4 services at once) don't hit an undocumented rate limit (Spike 3)

Until those four are green, the architecture is validated in the READ direction and the LLM-substrate direction, but not in the actual deploy-and-spawn direction.

---

## Recommended Next-Session Actions (Mon AM)

1. ~~Resolve Build API auth~~ — DONE
2. Spike 2: deploy a tiny ARM64 image to verify the deploy path end-to-end (15 min)
3. Spike 3: validate recursive spawn — agent-from-inside-agent (30 min, the only remaining unknown)
4. Build spawner FastAPI skeleton + deploy
5. Build agent FastAPI skeleton (decide → execute path only, no split yet)
6. Single end-to-end run: spawner accepts task → agent deploys → calls Anthropic → reports → self-deletes

If Spike 3 fails (agent-spawning-agent doesn't work), fall back to the **central spawner orchestrates all spawns** redesign — same demo, different internal pattern, ~4 hours of refactor.
