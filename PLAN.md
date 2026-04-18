# Sentinel — Master Plan

> Per-tenant Trust & Safety workers with wallet-bounded autonomy, built on Locus.
> **Hackathon**: Paygentic Week 2 (BuildWithLocus). **Submission**: Wed 22 Apr. **Prize target**: 1st place → 30-min YC founder call.

---

## Table of Contents

1. [TL;DR](#tldr)
2. [The Pitch](#the-pitch)
3. [Vertical and Demo Personas](#vertical-and-demo-personas)
4. [Architecture](#architecture)
5. [Locus Primitives and Endpoints](#locus-primitives-and-endpoints)
6. [Tech Stack and Repo Structure](#tech-stack-and-repo-structure)
7. [Build Sequence](#build-sequence)
8. [Spike Validations — Run These FIRST](#spike-validations--run-these-first)
9. [Kill-Switch Staged Degradation](#kill-switch-staged-degradation)
10. [Pitch Outline (5 min)](#pitch-outline-5-min)
11. [Judging Rubric Alignment](#judging-rubric-alignment)
12. [Risk Register](#risk-register)
13. [Deferred / Out of Scope](#deferred--out-of-scope)
14. [Open Questions for Next Session](#open-questions-for-next-session)
15. [Next Session Quick Start](#next-session-quick-start)

---

## TL;DR

**Sentinel** is a deployable Trust & Safety service for marketplace platforms. Each merchant/brand gets an **isolated, wallet-capped worker deployed on BuildWithLocus** that moderates their listings through a staged pipeline of:

1. **Wrapped APIs** — OpenAI Moderation + GPT-4o-mini vision, Anthropic Claude Haiku for reasoning
2. **Locus Tasks** — human reviewer escalation for low-confidence items
3. **PayWithLocus wallet** — per-tenant budget that the worker self-honors

When a tenant's wallet drops through defined thresholds, the system **gracefully degrades** that tenant's pipeline only — other tenants keep running at full capacity. When the wallet tops up, the pipeline climbs back to full automatically.

The architecture is intentionally **two-layered**: a reusable orchestration engine (`engine/`) plus a Shopify-T&S adapter (`adapter/`). Same engine can back a Week 4 LocusFounder submission with a different adapter.

---

## The Pitch

**Hook**: Shopify-scale marketplace. 50 brands. Holiday season surge. 5,000 listings/hour. One T&S operator.

**Problem**: Every existing moderation pipeline has the same failure mode — it's either expensive (unbounded API spend when traffic surges) or lossy (queues pile up, bad content slips through). There's no graceful middle. Worse, a single bad-actor tenant can exhaust shared API budgets for every other tenant on the platform.

**Solution**: Sentinel gives each tenant its own **deployed worker** with its own **wallet** and its own **fiscal policy**. The platform operator sets the allowance. The agent enforces it. When a tenant's surge or abuse pattern burns through their budget, only that tenant degrades — the other 49 brands never notice.

**The Locus angle**: Three Locus primitives working as one programmable economy — BuildWithLocus for per-tenant deploys, Wrapped APIs for metered intelligence, Tasks for human escalation — all settled in one wallet, one ledger, one currency.

---

## Vertical and Demo Personas

**Platform**: Shopify-like marketplace (fully mocked — no real Shopify integration).

Three tenants for the live demo (reduced from 5 to fit realistic $8–9 credit budget). Each represents a different risk profile to show the architecture's range:

| Brand | Category | Risk Profile | Demo Role |
|---|---|---|---|
| **Wave & Stone** | Handmade jewelry | Low — mostly clean | Control group, stays at Stage 0 the whole demo |
| **PowerPro Gear** | Fitness supplements | High — prohibited medical claims | Secondary active (LLM-heavy), runs alongside |
| **GlowUp Beauty** | Beauty products | High — banned ingredients, medical claims | **Kill-switch target** — surge this during demo |

Mention in the close: "Same engine scales to N tenants; we ran 3 live for the demo. RetroVibe Apparel and Cozy Corner Home are in our fixtures and run identically."

Fixture brands kept (used in pre-recorded segments / scripts only):
- **RetroVibe Apparel** — vintage clothing, counterfeit logos, vision-heavy
- **Cozy Corner Home** — home decor, copyright issues, mixed pipeline

Synthetic listings live in `adapter/fixtures/` as JSON — title, description, image URL, claimed category. A mix of clearly-clean, clearly-bad, and borderline items per brand.

---

## Architecture

### Topology

```
┌──────────────────────────────────────────────────────────┐
│  CONTROL PLANE  (1 Locus service, always on)              │
│  - Tenant registry + wallet sync                          │
│  - Fiscal policy engine (the brain)                       │
│  - Provisioner (BuildWithLocus API client)               │
│  - Mock Shopify listing producer                          │
│  - Operator + demo UI (SSE stream)                        │
└───────┬──────────────────────────────────────────────────┘
        │ POST /v1/services  (per tenant at onboard time)
        ▼
┌──────────────────────────────────────────────────────────┐
│  TENANT WORKERS  (1 Locus service per brand)              │
│  Same pre-built ARM64 image, different env vars           │
│  - Pulls listings from tenant-scoped queue                │
│  - Calls wrapped APIs (circuit-broken by fiscal stage)    │
│  - Calls Tasks API for human escalation                   │
│  - Reports results + fiscal state to control plane        │
│  - SIGTERM handler → checkpoint + graceful shutdown       │
└───────┬──────────────────────────────────────────────────┘
        │
        ▼
┌──────────────────────────────────────────────────────────┐
│  SHARED ADDONS                                            │
│  - Postgres: tenants, listings, results, audit log        │
│  - Redis: fiscal counters, fleet_status flags per tenant  │
└──────────────────────────────────────────────────────────┘
```

**Service count for full demo**: 1 control plane + 3 tenant workers + 1 addon (Postgres) = **5 services × $0.25 = $1.25 fixed**.

> **Budget note**: hackathon credit grants are typically $5–10, not the schema-max $50. Plan was reduced from 5 tenants + Redis to 3 tenants + Postgres-only to fit a realistic $8–9 total burn. See §11 risk register and §12 cuts.

### Engine vs Adapter Split

```
sentinel/
├── engine/                       # reusable core (Week 4-ready)
│   ├── orchestrator/             # control plane service
│   │   ├── app.py                # FastAPI entry
│   │   ├── routes/
│   │   ├── tenant_registry.py
│   │   └── provisioner.py        # BuildWithLocus client
│   ├── fiscal_policy/
│   │   ├── stages.py             # the 4-stage state machine
│   │   ├── wallet_monitor.py     # polls + pre-flight checks
│   │   └── circuit_breaker.py    # wrapped API gating
│   ├── worker_runtime/
│   │   ├── worker.py             # FastAPI entry for tenant worker
│   │   ├── queue_client.py
│   │   ├── checkpoint.py         # SIGTERM handler + Redis state
│   │   └── reporter.py           # phones home to control plane
│   └── interfaces/
│       ├── task_decomposer.py    # abstract base class
│       ├── worker_contract.py
│       └── result_interpreter.py
├── adapter/                      # Shopify T&S specific
│   ├── listing_decomposer.py     # implements TaskDecomposer
│   ├── moderation_worker.py      # implements WorkerContract
│   ├── result_interpreter.py     # cleared / escalated / blocked
│   ├── prompts/
│   │   ├── vision.md
│   │   ├── claims_reasoning.md
│   │   └── escalation_brief.md
│   ├── tasks_mapping.py          # confidence band → Tasks category + tier
│   └── fixtures/
│       ├── brands.json
│       └── listings/*.json
├── ui/                           # Next.js 15 App Router
│   ├── app/
│   ├── components/
│   └── lib/sse.ts
├── infra/
│   ├── Dockerfile.control        # ARM64 slim python
│   ├── Dockerfile.worker         # ARM64 slim python
│   └── .locusbuild               # if we go multi-service-from-repo
├── scripts/
│   ├── spikes/                   # validation scripts (§8)
│   ├── bootstrap.sh              # one-shot env setup
│   └── provision_tenants.sh      # deploys all 5 demo tenants
├── .env.example
├── .gitignore
├── README.md
└── PLAN.md                       # this file
```

### Four-Layer Fiscal Policy Engine

One policy engine, four enforcement points:

| Layer | Substrate | Check Point | Action When Constrained |
|---|---|---|---|
| **A — Provisioning gate** | BuildWithLocus services | Before spawning a new tenant worker | Refuse spawn if workspace balance < N × $0.25 |
| **B — API circuit breaker** | Wrapped APIs (OpenAI, Anthropic) | Before every wrapped API call | Degrade per tenant stage (see §9) |
| **C — Escalation router** | Locus Tasks | Before submitting a task | Raise confidence threshold for escalation, fall back to auto-block |
| **D — Wallet monitor** | PayWithLocus balance | Every 20s + pre-flight before A/B/C | Updates tenant stage, propagates via Redis flag |

**Key invariant**: Layer D is the only writer of `fleet_status` per tenant. Layers A/B/C are readers. No races.

### Worker Lifecycle

1. **Cold start** — 1-2 min (pre-built ARM64 image). Accept the latency; pre-deploy all demo tenants before pitch.
2. **Steady state** — worker long-polls its tenant queue, processes listings, reports to control plane.
3. **Fiscal stage transition** — worker reads its `fleet_status:{tenant_id}` key from Redis at natural breakpoints (between listings, never mid-item). Reconfigures pipeline.
4. **Graceful shutdown** — SIGTERM handler: finish current item, checkpoint partial state to Postgres, requeue unstarted items with `status=INTERRUPTED`, call `DELETE /v1/services/{id}` on self.
5. **Dead man's switch** (engine-side safety net) — control plane reaper every 60s scans for workers past `expected_completion_at * 1.5` and force-kills + requeues.

---

## Locus Primitives and Endpoints

All env vars read from `.env` (gitignored). `.env.example` committed.

### Authentication

```bash
# Exchange API key for 30-day JWT
TOKEN=$(curl -s -X POST "$LOCUS_BUILD_API_URL/v1/auth/exchange" \
  -H "Content-Type: application/json" \
  -d "{\"apiKey\":\"$LOCUS_API_KEY\"}" | jq -r '.token')

# Verify
curl -s "$LOCUS_BUILD_API_URL/v1/auth/whoami" -H "Authorization: Bearer $TOKEN"
```

### BuildWithLocus — Provisioning

| Operation | Endpoint |
|---|---|
| Create project | `POST $LOCUS_BUILD_API_URL/v1/projects` |
| Create environment | `POST $LOCUS_BUILD_API_URL/v1/projects/{id}/environments` |
| Create service (pre-built image) | `POST $LOCUS_BUILD_API_URL/v1/services` with `source.type: "image"` |
| Trigger deployment | `POST $LOCUS_BUILD_API_URL/v1/deployments` |
| Poll deployment | `GET $LOCUS_BUILD_API_URL/v1/deployments/{id}` |
| Provision addon (Postgres/Redis) | per docs, exact endpoint in SKILL.md |
| Delete service | `DELETE $LOCUS_BUILD_API_URL/v1/services/{id}` |
| Billing balance | `GET $LOCUS_BUILD_API_URL/v1/billing/balance` |

**Constraints**:
- Pre-built images must be `linux/arm64`
- Containers listen on `PORT=8080` (auto-injected)
- Health check endpoint required (default `/health`, returns HTTP 200)
- New workspaces start with $1.00 free (covers 4 services)

### Wrapped APIs

Base: `https://api.paywithlocus.com/api/wrapped/<provider>/<endpoint>` (verify beta path in Spike 2).

```bash
# Example: OpenAI moderation
curl -s -X POST "https://api.paywithlocus.com/api/wrapped/openai/moderations" \
  -H "Authorization: Bearer $LOCUS_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"input": "some listing description"}'
```

USDC deducted from wallet per call. Catalog at `GET /api/wrapped/md`.

### Tasks API

Category: "Written Content" or "Graphic Design" per escalation type.
Tier: 1 (budget) / 2 (mid) / 3 (premium). Start at tier 1 for demo.
Exact endpoint shapes live in `beta-api.paywithlocus.com/api/skills/skill.md` — fetch in session 1.

### Wallet + Credits

```bash
# Gift code request — FILED on 2026-04-19. Request ID: 52606d86-658f-4442-a7b7-45b60f5ea0fd
# Approval ≤24h. Requested $50 USDC.
#
# NOTE: Docs claim "no auth required" for the agent-facing endpoint, but beta DOES require the API key.
# This works:
curl -X POST "$LOCUS_BASE_URL/api/gift-code-requests" \
  -H "Authorization: Bearer $LOCUS_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "email": "fuadsanin2665@gmail.com",
    "reason": "Building Sentinel — per-tenant Trust & Safety agent platform",
    "githubUrl": "https://github.com/fuadsn/sentinel",
    "requestedAmountUsdc": 50
  }'
```

Rate limit: 1 request per email per 24h. Don't re-file — check approval status instead.

---

## Tech Stack and Repo Structure

| Component | Choice | Rationale |
|---|---|---|
| Control plane + workers | **Python 3.11 + FastAPI** | Same language = fiscal policy/prompt/queue code reuses across roles |
| Container base | **`python:3.11-slim` on `linux/arm64`** | Locus requires ARM64 for pre-built images |
| Dependency mgmt | **`uv`** (or `pip` + `requirements.txt` if uv adds friction) | Fast, deterministic |
| Demo UI | **Next.js 15 (App Router) + SSE** | Real-time monologue stream — SSE is simpler than WebSockets, enough for demo |
| Persistence | **Postgres addon** (state) + **Redis addon** (fiscal counters + flags) | Native Locus addons, auto-injected URLs |
| Deploy method | **Pre-built ARM64 image** (`source.type: "image"`) | Skips 3-7 min source build; 1-2 min cold start instead |
| Image registry | Build and push to a public registry (GHCR) | Locus pulls by URI |

---

## Build Sequence

> **Lost day**: Saturday. Effective build days: Sun / Mon / Tue. Wed = polish + submit. Reality-check daily, cut aggressively if a day slips.

| Day | Goal | "Done" signal |
|---|---|---|
| **Sun (today)** | Spikes pass + scaffold up + control plane skeleton deployable | All 4 spikes green. Empty FastAPI control plane reaches `healthy` on Locus. |
| **Mon** | Engine core + one real worker | Engine four-layer fiscal policy runs locally. One tenant worker deploys, pulls from queue, calls one wrapped API, reports back. |
| **Tue AM** | Multi-tenant + kill switch + UI shell | All 5 tenants deployed. Staged degradation works end-to-end on a mocked wallet drain. UI shows per-tenant state. |
| **Tue PM** | Demo-polish + record backup video | 90s backup recording with voiceover in hand. UI acceptable. Fixtures finalized. |
| **Wed AM** | Final run-through, fix-only, no new features | Full demo executes live end-to-end without intervention. |
| **Wed PM** | Submit | Submission includes: repo link, demo video link (backup), README with 5-min pitch summary. |

**Already cut to fit $8–9 credit budget**:
1. ~~Redis addon~~ — Postgres only for fiscal counters
2. ~~5 tenants~~ — 3 live (Wave & Stone, PowerPro, GlowUp); other 2 in fixtures
3. ~~Live Tasks API in demo~~ — 1–2 real submissions captured for backup video, mocked live

**Cut list if we slip further** (in order):
1. Drop graceful recovery (wallet top-up auto-climb) — demonstrate down-stage only
2. Drop Stage 2 keyword fallback — go straight from Stage 1 to Stage 3 frozen
3. Drop secondary tenant (PowerPro) — show 2 tenants only (control + target)

**Never cut**: BuildWithLocus real deployments, wrapped API real calls, fiscal kill switch visibly firing. These are the three things the judges must see.

---

## Spike Validations — Run These FIRST

Goal: prove the architecture is buildable before sinking 3 days in. Budget: **~$1 spend, ~45 min**.

All scripts go in `scripts/spikes/`. Run in order; abort on failure and redesign.

### Spike 1 — Auth + Wallet Read (5 min)

```bash
# scripts/spikes/01_auth.sh
source .env
TOKEN=$(curl -s -X POST "$LOCUS_BUILD_API_URL/v1/auth/exchange" \
  -H "Content-Type: application/json" \
  -d "{\"apiKey\":\"$LOCUS_API_KEY\"}" | jq -r '.token')
echo "JWT: ${TOKEN:0:40}..."

# whoami
curl -s "$LOCUS_BUILD_API_URL/v1/auth/whoami" -H "Authorization: Bearer $TOKEN" | jq

# billing balance
curl -s "$LOCUS_BUILD_API_URL/v1/billing/balance" -H "Authorization: Bearer $TOKEN" | jq
```

**Pass criteria**: whoami returns identity, balance endpoint returns non-null credit balance.
**Fail action**: API key invalid or expired → regenerate via `/api/register`.

### Spike 2 — Wrapped API Call with USDC Deduction (5 min)

```bash
# scripts/spikes/02_wrapped.sh
# Note balance BEFORE
BEFORE=$(curl -s "$LOCUS_BUILD_API_URL/v1/billing/balance" -H "Authorization: Bearer $TOKEN" | jq -r '.creditBalance')

# Call cheapest wrapped API — OpenAI moderation
curl -s -X POST "https://api.paywithlocus.com/api/wrapped/openai/moderations" \
  -H "Authorization: Bearer $LOCUS_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"input": "This is a test listing with no violations"}' | jq

# Note balance AFTER
AFTER=$(curl -s "$LOCUS_BUILD_API_URL/v1/billing/balance" -H "Authorization: Bearer $TOKEN" | jq -r '.creditBalance')
echo "Before: $BEFORE → After: $AFTER"
```

**Pass criteria**: wrapped response is a valid OpenAI moderation payload; balance decreased.
**Fail action**: if base URL is wrong, try `beta-api.paywithlocus.com/api/wrapped/...` instead. Document the correct base in PLAN.md and `.env`.

### Spike 3 — Single Service Deploy from Pre-Built Image (15 min)

```bash
# scripts/spikes/03_deploy.sh
# Use a known-good arm64 public image as placeholder (nginx with arm64 will fail health check — use a hello-world app)
# Recommended: `ghcr.io/locusresearch/hello:latest` if Locus provides one, else build our own minimal
# For now, use a tiny FastAPI image we'll publish: ghcr.io/fuadsn/sentinel-spike:latest

PROJECT_ID=$(curl -s -X POST "$LOCUS_BUILD_API_URL/v1/projects" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"name":"sentinel-spike","description":"spike test"}' | jq -r '.id')

ENV_ID=$(curl -s -X POST "$LOCUS_BUILD_API_URL/v1/projects/$PROJECT_ID/environments" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"name":"production","type":"production"}' | jq -r '.id')

SERVICE_ID=$(curl -s -X POST "$LOCUS_BUILD_API_URL/v1/services" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{
    "projectId":"'"$PROJECT_ID"'",
    "environmentId":"'"$ENV_ID"'",
    "name":"hello",
    "source":{"type":"image","imageUri":"ghcr.io/fuadsn/sentinel-spike:latest"},
    "runtime":{"port":8080,"cpu":256,"memory":512,"minInstances":1,"maxInstances":1}
  }' | jq -r '.id')

DEPLOY_ID=$(curl -s -X POST "$LOCUS_BUILD_API_URL/v1/deployments" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"serviceId":"'"$SERVICE_ID"'"}' | jq -r '.id')

echo "Deploy: $DEPLOY_ID — poll every 60s"
```

**Prep required**: publish a tiny FastAPI hello-world image to GHCR for ARM64 before running.

```dockerfile
# infra/Dockerfile.spike
FROM python:3.11-slim
RUN pip install --no-cache-dir fastapi uvicorn
RUN echo 'from fastapi import FastAPI\napp=FastAPI()\n@app.get("/health")\ndef h(): return {"status":"ok"}\n@app.get("/")\ndef r(): return {"hello":"sentinel"}' > /app.py
EXPOSE 8080
CMD ["uvicorn","app:app","--host","0.0.0.0","--port","8080"]
```

Build and push:
```bash
docker buildx build --platform linux/arm64 -t ghcr.io/fuadsn/sentinel-spike:latest -f infra/Dockerfile.spike --push .
```

**Pass criteria**: deployment reaches `healthy` within 3 min; service URL returns `{"hello":"sentinel"}`.
**Fail action**: if health check fails, check the `/health` endpoint is wired. If port issue, verify `PORT=8080`.

### Spike 4 — Tasks API Submission (10 min + wait)

```bash
# scripts/spikes/04_tasks.sh
# First fetch skill.md for exact task endpoint
curl -s "$LOCUS_BASE_URL/api/skills/skill.md" > /tmp/locus-skill.md
grep -A 20 -i "tasks" /tmp/locus-skill.md | head -40

# Then submit a tier-1 task (exact body TBD from skill.md)
# Sketch:
# curl -X POST "$LOCUS_BASE_URL/api/tasks" -H "Authorization: Bearer $LOCUS_API_KEY" \
#   -d '{"category":"Written Content","timeline":1,"priceTier":1,"brief":"Write a 50-word description of a handmade necklace"}'
```

**Pass criteria**: task submits, receives an ID, status is observable.
**Fail action**: if Tasks API isn't live on beta, fall back to mocking it in UI and note architecturally.

---

## Kill-Switch Staged Degradation

Per-tenant. Transitions driven by wallet-monitor poll (Layer D) writing `fleet_status:{tenant_id}` in Redis. Workers read at natural breakpoints (between listings).

| Stage | Wallet | Behavior | API Cost per Listing |
|---|---|---|---|
| **0 — Healthy** | > $2.00 | Vision + LLM reasoning + Tasks escalation (if confidence < 0.7) | ~$0.015 |
| **1 — Conserving** | $0.50 – $2.00 | Vision off, LLM-only, escalation threshold raised to < 0.5 | ~$0.005 |
| **2 — Critical** | $0.10 – $0.50 | API calls paused, keyword-blocklist only, no escalations (auto-flag for operator) | ~$0.000 |
| **3 — Frozen** | < $0.10 | Processing paused, queue persists, operator alert, worker stays online but idle | $0 |

**Recovery**: Stage transitions work both directions. When wallet tops up, next poll cycle promotes the tenant back up the stages. UI shows this climb live.

**Testing**: engine exposes an admin endpoint `POST /debug/simulate_drain?tenant_id=X&target_balance=Y` to force-set a tenant balance for demos without actually burning USDC.

---

## Pitch Outline (5 min)

| Time | Beat | Visual |
|---|---|---|
| 0:00–0:30 | **Hook** — "Shopify-scale marketplace, holiday surge, 5,000 listings/hour, one T&S operator" | Dashboard: 5 brand tiles, live listing counter |
| 0:30–1:30 | **Problem** — current T&S either burns unbounded spend or falls over; shared pipelines let one bad tenant starve others | Slide with 2-column failure-mode comparison |
| 1:30–3:30 | **Live demo** — 5 brands running; surge GlowUp; watch its wallet drop through stages 0→1→2; *other 4 brands stay at stage 0 the whole time*; top up; watch recovery | Full demo UI with per-tenant wallet bars, live monologue stream |
| 3:30–4:30 | **Architecture reveal** — engine/adapter split slide, three Locus primitives chained (BuildWithLocus + Wrapped APIs + Tasks), one wallet, one ledger | Single architecture diagram, then overlay "same engine, swap adapter" for 3 alt verticals |
| 4:30–5:00 | **Close** — this isn't a moderation tool; it's a programmable fiscal policy engine for deployable agent workforces. Marketplace T&S is our demo; the engine is the product. | Tagline slide + repo + demo link |

Q&A prep (2 min):
- "Why per-tenant deploys instead of one multi-tenant service?" → Isolation, per-tenant fiscal policy, tenant-churn teardown matches Locus's flat-fee pricing
- "Cold start of 1-2 min — isn't that a problem?" → We pre-deploy tenant workers on onboarding, not on demand. Cold start is a one-time onboarding cost, not per-request
- "What happens to the tasks in-flight when kill-switch fires?" → Graceful checkpoint (Stage 1 and below workers finish current item, flush state, degrade on next breakpoint). No data loss.
- "How is this different from a rate limiter?" → A rate limiter caps calls; Sentinel reconfigures the *pipeline* — different API substrate, different accuracy trade-off, different human-loop behavior per stage.

---

## Judging Rubric Alignment

| Dimension | Points | Our play |
|---|---|---|
| **Technical Excellence** | 30 | Real BuildWithLocus per-tenant deploys, real wrapped API spend, real Tasks integration, graceful SIGTERM checkpoint, dead-man's-switch safety net |
| **Innovation & Creativity** | 25 | Per-tenant fiscal policy (novel), chaining three Locus primitives into one autonomous economy (unique), engine/adapter split (platform thinking) |
| **Business Impact** | 25 | T&S is a real $B+ market; the "budget-aware moderation" framing is a pain every marketplace has |
| **User Experience** | 20 | Demo UI with live monologue + per-tenant wallet bars + stage indicators — the visible surface is the engine's reasoning, which is the whole point |

**Weakest dimension**: UX. Mitigation — invest real time in the demo UI Tue PM. The monologue stream is the only thing judges see, so it has to look alive and legible.

---

## Risk Register

| # | Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|---|
| 1 | Wrapped API base URL differs between beta and production | M | H | Spike 2 pins it; fallback paths documented in `.env.example` |
| 2 | Tasks API not live on beta | M | M | Architecture still claims it; UI shows mocked escalation with footnote |
| 3 | Cold start of 1-2 min per tenant makes 5-tenant provision >10 min | H | M | Pre-provision all 5 tenants before pitch; show live spawn of ONE during demo only |
| 4 | Gift code approval takes too long (up to 24h) | H | H | Filed Sun AM (ID 52606d86-...); $1 free tier covers Spike 1-3; Tasks spike can wait |
| 4b | **Credit grant comes back at $5 not $50** (typical hackathon grant is ≤$10, schema-max ≠ approved-amount) | **H** | **H** | Plan rebuilt against $8–9 realistic budget: 3 tenants not 5, no Redis, 1–2 real Tasks calls captured for backup video and mocked live |
| 5 | Wallet balance poll lag creates race window on spend decisions | M | M | Pre-flight balance check at layer gate; polling is ambient only |
| 6 | Live demo deploy fails during pitch | M | H | Pre-recorded 90s backup demo as Plan B; switch within 10 seconds |
| 7 | Vision model latency makes demo feel slow | L | M | Throttle mock listing producer to match pipeline throughput; don't flood |
| 8 | Running over actual credit grant during dev | M | H | Use `/debug/simulate_drain` for kill-switch demo (no real burn); real spend only on actual processing; Stage 2 is ~$0/listing by design |
| 9 | Shopify mock is too obviously fake for judges | L | L | Style the UI like Shopify Admin; call it "a Shopify-style marketplace" openly |
| 10 | Tasks API per-call cost is undisclosed in docs; could be expensive | M | M | Capture 1–2 real submissions during dev for backup video, mock during live demo |

---

## Deferred / Out of Scope

- Real Shopify webhook integration
- Custom domains (use auto `svc-*.buildwithlocus.com`)
- Multi-region (us-east-1 only)
- Git push deploy (using pre-built image path)
- Auto-deploy on push
- Multi-environment (production only)
- Real OFAC / sanctions data (we're moderating listings, not customs)
- Data-loss-safe checkpointing for crashes (dead man's switch is the safety net, not per-byte durability)
- Wallet top-up automation (operator action in the demo)
- Any Week 4 LocusFounder-specific behaviors — those come later if needed

---

## Open Questions for Next Session

Things we couldn't resolve from docs alone; must validate or fetch in session 1:

1. **Wrapped API base URL on beta** — is it `api.paywithlocus.com` or `beta-api.paywithlocus.com`? (Spike 2)
2. **Tasks API exact endpoint shape** — pull from `beta-api.paywithlocus.com/api/skills/skill.md` in session 1
3. **Wallet balance endpoint** — is it `/api/pay/balance`, `/v1/billing/balance`, or both? Confirm whether it's workspace credit or USDC wallet (different numbers)
4. **Per-service env var patching** — can we mutate env vars on an existing service and trigger redeploy, or do we need to recreate? (affects how we toggle fiscal stages at deploy-layer vs runtime-layer)
5. **GHCR pull from Locus** — does Locus pull public GHCR images cleanly, or do we need to push to a specific registry? (affects Spike 3 setup)
6. **Redis addon provisioning time** — docs say 10-20s; confirm on real run
7. **Does `from-repo` work if we don't have a `.locusbuild` file yet?** — we may want to start with direct `POST /v1/services` instead

---

## Next Session Quick Start

```bash
# 1. Orient
cd /Users/fuad/Developer/Projects/personal/sentinel
cat PLAN.md | less

# 2. Verify .env is populated
cat .env

# 3. Gift code request already filed 2026-04-19 (ID 52606d86-658f-4442-a7b7-45b60f5ea0fd)
#    Check approval status; if >24h and no credits, contact Locus team via Discord

# 4. Run Spikes 1 → 4 in order
bash scripts/spikes/01_auth.sh
bash scripts/spikes/02_wrapped.sh
# Spike 3 requires building + pushing the image first
docker buildx build --platform linux/arm64 -t ghcr.io/fuadsn/sentinel-spike:latest -f infra/Dockerfile.spike --push .
bash scripts/spikes/03_deploy.sh
bash scripts/spikes/04_tasks.sh

# 5. If all green → start engine scaffold per §6 repo structure
```

**Session 1 success criteria**:
- Gift code request filed
- All 4 spikes green (or fail documented with redesign note)
- Control plane FastAPI skeleton deploys to Locus and reaches `healthy`
- Engine interfaces (`task_decomposer.py`, `worker_contract.py`, `result_interpreter.py`) written as ABCs
- `scripts/provision_tenants.sh` skeleton that can spawn one demo worker (doesn't have to process listings yet)

If Sun is done by end of day, Mon is on track.
