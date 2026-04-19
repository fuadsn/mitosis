# Mitosis — Master Plan

> **Recursively self-replicating agents on Locus.** An agent receives a task, decides whether to execute or *split*, and if it splits — spawns N child services on BuildWithLocus with portions of its wallet. Children may spawn grandchildren. The tree expands and collapses, bounded only by the root budget.
>
> **Hackathon**: Paygentic Week 2 (BuildWithLocus). **Submission**: Wed 22 Apr. **Prize target**: 1st place → 30-min YC founder call.
>
> **Demo task**: Pre-acquisition codebase technical due diligence — root agent detects the tech stacks in a target repo, deterministically spawns per-stack specialists, one specialist autonomously decides to spawn a deep-analyzer grandchild for a suspect file. Real depth-3 recursion. Output: a security + code-quality findings report.

---

## Table of Contents

1. [TL;DR](#tldr)
2. [The Pitch](#the-pitch)
3. [The Demo Task](#the-demo-task)
4. [Architecture](#architecture)
5. [The Recursion Decision](#the-recursion-decision)
6. [Wallet Splitting and Bounds](#wallet-splitting-and-bounds)
7. [Coordination Protocol](#coordination-protocol)
8. [Locus Primitives and Endpoints](#locus-primitives-and-endpoints)
9. [Tech Stack and Repo Structure](#tech-stack-and-repo-structure)
10. [Build Sequence](#build-sequence)
11. [Spike Validations](#spike-validations)
12. [Pitch Outline (5 min)](#pitch-outline-5-min)
13. [Judging Rubric Alignment](#judging-rubric-alignment)
14. [Risk Register](#risk-register)
15. [Deferred / Out of Scope](#deferred--out-of-scope)
16. [Open Questions for Next Session](#open-questions-for-next-session)
17. [Carryover from Sentinel Planning](#carryover-from-sentinel-planning)
18. [Next Session Quick Start](#next-session-quick-start)

---

## TL;DR

**Mitosis** is a platform for **recursive autonomous computation**. Every agent runs as its own deployed service on BuildWithLocus, holds its own USDC wallet, and decides at runtime whether to do its task itself or **fork into N child agents** — each spawned as a fresh Locus deployment with a portion of the parent's budget.

The tree of agents grows and collapses dynamically. The only thing preventing infinite recursion is **fiscal bounding**: each split divides the parent's wallet, and a minimum-budget threshold forces leaves to execute directly. When children finish, parents synthesize and self-terminate.

This is `fork()` + `wait()` for the agent economy, with USDC as the bound.

---

## The Pitch

**Hook**: *"Pre-acquisition technical due diligence is a $200,000 engagement that takes 3 weeks. We're going to do it in 20 minutes for $20."*

**Problem**: When a company acquires another, they pay specialist firms (Shea & Company, Lincoln, ICX) to audit the target's codebase — security posture, dependency risks, code quality, key-person dependencies. The work decomposes naturally (per stack, per file, per concern) but the decomposition shape changes with every codebase. You can't pre-script it. So firms throw analysts at it manually for weeks.

**Solution**: Mitosis is a recursive autonomous orchestrator. Submit a target repo with a budget. The root agent scans the repo manifest, **deterministically** spawns one specialist per detected stack (Python, JS, Docker, GitHub Actions). Each specialist then **autonomously decides via LLM** whether to execute directly or to spawn a deep-analyzer grandchild for a specific file. Some branches go deep, others stay shallow — driven by what the agent actually finds.

Every agent runs as its own deployed service on BuildWithLocus, holds its own portion of the parent's wallet, and self-terminates when done.

**Why Locus is the only platform that makes this work**:
- **BuildWithLocus** turns a service deployment into an API call — agents *can* spawn agents
- **PayWithLocus** gives every agent a real wallet — fiscal bounding is enforceable, not just simulated
- **Wrapped APIs** are pay-per-call USDC — costs are knowable at decision time, so the split-vs-execute economics are real

Three Locus primitives, one substrate for autonomous decomposition.

### Business case (the buyer)

| | Manual incumbent | Mitosis |
|---|---|---|
| Provider | Shea & Co / ICX / Lincoln | Mitosis platform |
| Time | 2–4 weeks | 20 minutes |
| Cost | $50,000 – $200,000 | $20 |
| Output | DD report PDF | DD report PDF (same shape) |
| Scalability | Limited by analyst headcount | Limited only by Locus capacity |

Initial customer profile: corporate development teams at PE / strategic acquirers running >5 deals/year. Secondary: cyber-insurance underwriters needing pre-policy code risk scores. Tertiary: any dev team auditing a third-party dependency before adoption.

---

## The Demo Task

**Pre-Acquisition Codebase Technical Due Diligence**

Submit: *"Run technical due diligence on [target-repo URL]. Output: security posture, code-quality concerns, dependency risks. Budget: $5 USDC."*

### Why this task — and not Series A investment DD

The original Series A DD task fakes depth-2 (a "Market Analyst spawns TAM/SAM/SOM" is contrived — those aren't real sub-agents in real DD). Codebase technical DD has *natural* depth-3 driven by structure: repo → stack → suspect file. Each level has a legitimate reason to split.

It also has a **real, named buyer market**: pre-acquisition technical DD is a $50k–$200k engagement typically run by firms like Shea & Company, ICX, or Lincoln International over 2–4 weeks. We do the same deliverable in 20 min for $20 in compute.

### Expected tree (live demo)

```
                           ┌──────────────────────────┐
                           │  ROOT — DD Coordinator   │
                           │  Scans repo manifest     │
                           │  Detects stacks present  │
                           │  Budget: $5              │
                           └────────────┬─────────────┘
                                        │ DETERMINISTIC SPLIT
                                        │ (one child per stack detected)
              ┌──────────────┬──────────┼──────────┬───────────────┐
              ▼              ▼          ▼          ▼               ▼
        ┌──────────┐   ┌──────────┐  ┌────────┐  ┌─────────┐
        │ Python   │   │ JS       │  │ Docker │  │ GHA     │
        │ Auditor  │   │ Auditor  │  │ Auditor│  │ Auditor │
        │ $0.80    │   │ $0.80    │  │ $0.80  │  │ $0.80   │
        └────┬─────┘   └────┬─────┘  └───┬────┘  └────┬────┘
             │              │            │            │
             │ LLM-DECIDED  │ LLM says   │ LLM says   │ LLM says
             │ SPLIT        │ "execute"  │ "execute"  │ "execute"
             │              │            │            │
             ▼              ▼            ▼            ▼
        ┌──────────┐    (direct      (direct       (direct
        │ auth.py  │     execute)     execute)      execute)
        │ Deep     │
        │ Analyzer │
        │ $0.40    │
        └──────────┘

  Total: 6 services, depth 3, fixed cost $1.50, wrapped APIs ~$2-3
```

### The two-mechanism honesty demo

The split decision is real — proven by showing **two different mechanisms** producing two different outcomes:

| Level | Mechanism | Demo proof |
|---|---|---|
| Root → 4 stack specialists | **Deterministic** (file-extension scan triggers per-stack spawn) | Show the rule firing — "Python files detected → spawn Python auditor" |
| Python specialist → auth.py grandchild | **LLM-decided** (Claude reads file summaries, decides which to deep-dive) | Show Claude's actual structured output: *"app.py and routes.py look standard. auth.py uses raw SQL string interpolation in get_user_by_email() — recommend deep analysis."* |
| JS / Docker / GHA specialists | **LLM-decided** but Claude says "execute" | Show Claude's reasoning explicitly choosing NOT to split — proves the decision isn't always-yes |

This is the answer to the *"how do we know the autonomy is real?"* question every sharp judge will ask.

### Per-specialist work

Each stack auditor:
1. Lists files matching its scope (via repo file tree fetched once at root)
2. Reads a sample of file contents (wrapped Anthropic for code reasoning)
3. Runs decide() — if Claude flags a file as suspect, spawn a deep-analyzer grandchild
4. Either executes its findings directly or synthesizes child findings
5. Reports structured findings to root

Root synthesizes a markdown DD report covering: security posture, dependency risks, code quality, key-file concerns. Report is the **demo artifact** — judges see a real, audit-grade document.

### Demo target repo

Pick a small open-source repo (~20-30 files) deliberately seeded with:
- Multiple stacks visible (Python, JS, Docker, GHA) — guarantees the deterministic split fires
- One file with an obvious-but-real issue (e.g., raw SQL injection vector) — guarantees the LLM-decided deep-dive fires
- Other files that are routine — guarantees other branches choose "execute" over "split"

Likely candidate: a small intentionally-vulnerable demo app like a fork of `OWASP/NodeGoat` or a custom small-repo we control. Pick during session 1.

---

## Architecture

### Topology

```
┌─────────────────────────────────────────────────────────────┐
│  SPAWNER  (1 Locus service, always on)                       │
│  - Public API: POST /tasks (submit a root task)              │
│  - Demo UI: live tree visualizer (SSE stream)                │
│  - Reaper: dead man's switch for orphaned children           │
│  - Result store reader (GET /tasks/{id})                     │
└──────────┬──────────────────────────────────────────────────┘
           │ POST /v1/services  (spawns root agent)
           ▼
┌─────────────────────────────────────────────────────────────┐
│  AGENT  (1 Locus service per agent in the tree)              │
│  Same image, different env vars                              │
│  Env: TASK_ID, PARENT_URL, BUDGET, DEPTH, TASK_DESC          │
│                                                              │
│  Lifecycle:                                                  │
│   1. Boot, register with Postgres                            │
│   2. DECIDE — split or execute? (LLM introspection)          │
│   3a. If SPLIT — spawn N children via BuildWithLocus API     │
│        wait for all child reports                            │
│        synthesize, report to PARENT_URL                      │
│        DELETE self                                           │
│   3b. If EXECUTE — call wrapped APIs for the work            │
│        report to PARENT_URL                                  │
│        DELETE self                                           │
└──────────┬──────────────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────────────┐
│  POSTGRES (shared addon)                                     │
│  - tree_nodes: id, parent_id, depth, status, budget, spent   │
│  - agent_results: agent_id, output_json, synthesis_input     │
│  - audit_log: append-only ledger of every decision/spawn     │
└─────────────────────────────────────────────────────────────┘
```

### Service Count for Demo

| Service | Count | Cost |
|---|---|---|
| Spawner | 1 | $0.25 |
| Postgres addon | 1 | $0.25 |
| Root agent | 1 | $0.25 |
| Stack specialist children (Python, JS, Docker, GHA) | 4 | $1.00 |
| Deep-analyzer grandchild (auth.py) | 1 | $0.25 |
| **Total fixed** | **8** | **$2.00** |

Plus wrapped API calls (~$2-3 for repo scan + Claude reasoning across all agents). Lands within the realistic $5-9 hackathon credit grant.

Tasks API live integration deferred to post-demo polish; mocked in UI for demo.

---

## The Recursion Decision

Every agent's first action is to **decide whether to split or execute**. This is the core of the system.

### Decision Inputs

| Variable | Source | Notes |
|---|---|---|
| `task_description` | Env var | Free-form natural language |
| `budget` | Env var | USDC available to this agent and its descendants |
| `depth` | Env var | Current depth in tree (root = 0) |
| `MAX_DEPTH` | Constant = 4 | Hard ceiling on recursion |
| `MIN_BUDGET_FOR_SPLIT` | Constant = $0.50 | Below this, must execute |
| `MAX_CHILDREN_PER_SPLIT` | Constant = 5 | Controls fan-out |
| `RETAINER_RATIO` | Constant = 0.20 | Parent keeps 20% of budget for synthesis |

### Decision Logic (in pseudo-code)

```python
def decide(task, budget, depth):
    if depth >= MAX_DEPTH:
        return EXECUTE  # forced — recursion ceiling

    if budget < MIN_BUDGET_FOR_SPLIT:
        return EXECUTE  # forced — can't afford to split

    # LLM introspection
    plan = anthropic_call(SPLIT_OR_EXECUTE_PROMPT, task=task, budget=budget)
    # plan is structured: {"action": "split"|"execute", "children": [...] | None}

    if plan.action == "execute":
        return EXECUTE
    else:
        return SPLIT(children=plan.children)
```

### The Split Prompt (sketch)

```
You are an autonomous agent considering a task.
Task: {task}
Budget: ${budget} USDC
Depth: {depth} of {MAX_DEPTH}

Decide ONE of:
1. EXECUTE — do this task directly with one focused LLM call
2. SPLIT — decompose into 2-{MAX_CHILDREN_PER_SPLIT} independent sub-tasks

Choose EXECUTE if:
- The task is atomic or already narrow
- Sub-tasks would have heavy overlap
- Budget is too small to fund meaningful children

Choose SPLIT if:
- Distinct sub-tasks are clearly separable
- Each sub-task benefits from its own specialist context
- Budget supports at least 2 children with non-trivial allocations

Respond JSON:
{
  "action": "execute" | "split",
  "reasoning": "...",
  "children": [
    {"task_description": "...", "budget_share_pct": 20},
    ...
  ]  // null if action=execute
}
```

---

## Wallet Splitting and Bounds

### Split Math

```
parent_budget = $5.00
retainer = parent_budget * 0.20 = $1.00   # parent keeps for synthesis + overhead
distributable = parent_budget * 0.80 = $4.00

# Distributed across N children proportional to LLM-recommended shares
# E.g., 5 equal children: $0.80 each
```

### Per-Agent Spend Tracking

Each agent tracks its own spend in real time:
- Wrapped API calls: deduct estimated cost on call (truth-up after response)
- Pre-flight check before each call: `budget - spent_so_far > estimated_call_cost`
- If a wrapped call would exceed budget → emergency-return with partial work + budget-exceeded flag

### Recursion Bounds (defense in depth)

| Bound | Value | Failure mode it prevents |
|---|---|---|
| `MAX_DEPTH` | 4 | Infinite recursive descent |
| `MIN_BUDGET_FOR_SPLIT` | $0.50 | Splits with sub-cent children |
| `MAX_CHILDREN_PER_SPLIT` | 5 | Pathological fan-out |
| `MAX_TREE_TOTAL_SERVICES` | 50 | Spawner refuses new spawns once total exceeded |
| `MAX_AGENT_LIFETIME_SEC` | 600 | Reaper kills agents older than this |

The fiscal substrate is the *primary* bound. The numerical bounds are belt-and-suspenders.

---

## Coordination Protocol

### Spawn

When parent decides to SPLIT:
1. Insert `tree_nodes` row per child (`status = pending`)
2. For each child:
   ```
   POST $LOCUS_BUILD_API_URL/v1/services
   { source: { type: image, imageUri: ghcr.io/fuadsn/mitosis-agent:latest },
     env: { TASK_ID, PARENT_URL: <my INTERNAL_URL>, BUDGET, DEPTH+1, TASK_DESC },
     runtime: { ... }
   }
   ```
3. Then `POST /v1/deployments` for each
4. Parent polls `tree_nodes` for child statuses (or accepts inbound webhook)

### Report

When child completes:
```
POST {PARENT_URL}/agent/{AGENT_ID}/report
{
  "agent_id": "...",
  "result": { ... },          # the actual output
  "spent_usdc": 0.45,
  "status": "complete" | "partial" | "failed",
  "child_tree": [...]         # if this child also split, recursive view
}
```

Parent tracks `expected_children = N`, `received_reports = M`. When `M == N` (or all timed out), trigger synthesis.

### Synthesis

Parent calls Anthropic with all child results:
```
prompt: "Here are reports from {N} specialist agents. Synthesize a unified output for: {original_task}.\n\n{child_reports_json}"
```

Output goes to parent's own report payload, sent up the chain.

### Self-Termination

After reporting (or if parent is root, after writing final result to Postgres):
```
DELETE $LOCUS_BUILD_API_URL/v1/services/{my_service_id}
```

### Reaper (in spawner)

Every 60s, scan `tree_nodes` for agents past `MAX_AGENT_LIFETIME_SEC`. For each:
1. Force `DELETE` on the service
2. Mark `tree_nodes.status = reaped`
3. If parent is still alive, send synthetic "child_failed" report so parent can synthesize without waiting

---

## Locus Primitives and Endpoints

### Authentication (RESOLVE FIRST IN SESSION 1)

⚠️ **Confirmed during planning probe**: `POST https://api.buildwithlocus.com/v1/auth/exchange` with the beta `claw_dev_*` key returns `{"error":"Invalid API key"}`. Beta has no `/v1/auth/exchange` either.

**Resolution path** for session 1:
1. Read `https://beta-api.paywithlocus.com/api/skills/skill.md` end-to-end
2. Try `Authorization: Bearer $LOCUS_API_KEY` directly against Build API endpoints (no exchange)
3. If still failing, ask in hackathon Discord
4. Last resort: register a separate production key

**This is the gate. Nothing deploys until resolved.**

### Confirmed Working Endpoints

| Operation | Endpoint | Auth |
|---|---|---|
| Wallet balance | `GET https://beta-api.paywithlocus.com/api/pay/balance` | `Bearer $LOCUS_API_KEY` |
| Gift code request | `POST https://beta-api.paywithlocus.com/api/gift-code-requests` | `Bearer $LOCUS_API_KEY` |

### BuildWithLocus — Provisioning (auth pending)

| Operation | Endpoint |
|---|---|
| Create project | `POST $LOCUS_BUILD_API_URL/v1/projects` |
| Create environment | `POST $LOCUS_BUILD_API_URL/v1/projects/{id}/environments` |
| Create service (pre-built image) | `POST $LOCUS_BUILD_API_URL/v1/services` with `source.type: "image"` |
| Trigger deployment | `POST $LOCUS_BUILD_API_URL/v1/deployments` |
| Poll deployment | `GET $LOCUS_BUILD_API_URL/v1/deployments/{id}` |
| Patch service env vars | `PUT $LOCUS_BUILD_API_URL/v1/variables/service/{id}` |
| Delete service | `DELETE $LOCUS_BUILD_API_URL/v1/services/{id}` |
| Provision Postgres addon | per docs (verify in skill.md) |

**Constraints**:
- Pre-built images must be `linux/arm64`
- Containers listen on `PORT=8080` (auto-injected)
- Health check endpoint `/health` returning HTTP 200 required
- Cold start: 1-2 min for pre-built image to reach `healthy`

### Wrapped APIs (auth confirmed via API key Bearer)

Demo uses three providers:

| Job | Provider | Endpoint |
|---|---|---|
| Search (per specialist analyst) | Exa | `wrapped/exa/search` |
| Reasoning + synthesis (every agent) | Anthropic Claude Haiku | `wrapped/anthropic/messages` |
| Embeddings (de-dup of search results, optional) | OpenAI | `wrapped/openai/embeddings` |

### Tasks API

Reserved for the **recorded backup video** (1-2 real submissions). Use case: when an agent decides a sub-task requires human judgment (e.g., "interpret a non-public legal filing"), it submits a Locus Task instead of executing or splitting. Demo shows architectural diagram + recorded clip; live demo doesn't burn budget on Tasks.

### Wallet + Credits

- **Current**: $5 promo on `ws_d75521d6` (verified 2026-04-19)
- **Pending**: gift code request `52606d86-658f-4442-a7b7-45b60f5ea0fd` ($50 asked, $5-10 expected)
- Watch with: `bash scripts/check_balance.sh`

---

## Tech Stack and Repo Structure

| Component | Choice | Rationale |
|---|---|---|
| Spawner + Agent | **Python 3.11 + FastAPI** | Same image for all roles; behavior controlled by env var (`AGENT_ROLE=spawner|agent`) |
| Container base | **`python:3.11-slim` on `linux/arm64`** | Locus requires ARM64 |
| Image registry | **GHCR** (`ghcr.io/fuadsn/mitosis-agent:latest`) | Free for public, native to GitHub |
| Demo UI | **Next.js 15 (App Router) + SSE** | Live tree visualization streams from Postgres deltas |
| Persistence | **Postgres addon only** (no Redis) | Single source of truth for tree state |

```
mitosis/
├── agent/                            # the recursive unit
│   ├── main.py                       # FastAPI entry, decides role from AGENT_ROLE
│   ├── decision.py                   # split-or-execute LLM call
│   ├── splitter.py                   # spawn children via BuildWithLocus
│   ├── executor.py                   # do the actual work via wrapped APIs
│   ├── synthesizer.py                # merge child reports
│   ├── reporter.py                   # POST to parent, then self-DELETE
│   ├── budget.py                     # spend tracking + pre-flight checks
│   └── prompts/
│       ├── decide.md                 # split-or-execute prompt
│       ├── execute_dd_specialist.md  # one per analyst type
│       └── synthesize_dd_memo.md     # root synthesis prompt
├── spawner/                          # always-on control plane
│   ├── main.py                       # FastAPI: POST /tasks, GET /tasks/{id}
│   ├── reaper.py                     # dead man's switch
│   ├── locus_client.py               # BuildWithLocus API wrapper
│   └── sse.py                        # demo UI live stream
├── ui/                               # Next.js demo
│   ├── app/
│   ├── components/TreeViz.tsx        # the live tree
│   └── lib/sse.ts
├── infra/
│   ├── Dockerfile                    # ONE Dockerfile, works for spawner + agent
│   └── locusbuild_sample.json        # for documentation, may not be used
├── scripts/
│   ├── spikes/                       # validation scripts
│   ├── check_balance.sh
│   ├── build_and_push.sh
│   └── seed_demo_task.sh             # submits the DD task for the demo
├── .env.example
├── .gitignore
├── README.md
└── PLAN.md
```

---

## Build Sequence

> Effective build days remaining: Sun (today, partial), Mon, Tue. Wed = polish + submit.

| Day | Goal | "Done" signal |
|---|---|---|
| **Sun (today)** | Build API auth resolved + spikes 1-3 green + spawner skeleton deploys | Can `POST /v1/services` from local; spawner reaches `healthy` on Locus |
| **Mon** | Single-agent path end-to-end | One agent spawns from spawner, makes one wrapped API call, reports back, self-deletes |
| **Tue AM** | Tree of depth 2 working | Root agent splits into 5 specialists, all complete, root synthesizes, demo task produces a real DD memo |
| **Tue PM** | UI + recorded backup video | Tree visualizer is presentable; 90s backup recording in hand |
| **Wed AM** | Live demo dry-run + polish | Full pitch executes live without intervention; backup tested |
| **Wed PM** | Submit | Repo + demo video link + 5-min pitch summary in README |

**Cut list if we slip**:
1. Drop demo UI — show terminal output + Postgres queries (ugly but proves the architecture)
2. Drop synthesis quality — root just concatenates child outputs (ugly but proves recursion)
3. Drop the depth-2 backup variant — demo only depth-1 (5 specialists, no grandchildren)

**Never cut**: real Locus deploys for every agent, real wrapped API calls, real self-DELETE, observable tree shape.

---

## Spike Validations

Before sinking days into building, prove the architecture is feasible.

### Spike 0 — Build API Auth (15 min, BLOCKER)

```bash
source .env

# Plan A: try direct Bearer with API key (no exchange)
curl -sS "$LOCUS_BUILD_API_URL/v1/projects" -H "Authorization: Bearer $LOCUS_API_KEY"

# Plan B: fetch the skill file and read auth section
curl -sS "$LOCUS_BASE_URL/api/skills/skill.md" > /tmp/locus-skill.md
grep -A 30 -i "auth\|exchange\|bearer" /tmp/locus-skill.md | head -50

# Plan C: Discord ask
```

**Pass criteria**: any Build API endpoint returns 200 with our key. Document the working pattern in `.env` as `LOCUS_BUILD_AUTH_HEADER=...`

### Spike 1 — Wallet Read + Wrapped API + Deduction (10 min)

```bash
# Balance before
BEFORE=$(bash scripts/check_balance.sh | jq -r '.promo_credit_balance')

# Cheapest wrapped API — OpenAI moderation
curl -sS -X POST "https://api.paywithlocus.com/api/wrapped/openai/moderations" \
  -H "Authorization: Bearer $LOCUS_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"input": "test"}' | jq

# Balance after
AFTER=$(bash scripts/check_balance.sh | jq -r '.promo_credit_balance')
echo "$BEFORE → $AFTER"
```

**Pass criteria**: response valid, balance decreased.
**Fallback**: if wrapped not on `api.paywithlocus.com`, try `beta-api.paywithlocus.com/api/wrapped/...`

### Spike 2 — Single Service Deploy from Pre-Built Image (20 min)

Prep — build a tiny ARM64 image:

```dockerfile
# infra/Dockerfile.spike
FROM python:3.11-slim
RUN pip install --no-cache-dir fastapi uvicorn
RUN printf 'from fastapi import FastAPI\napp=FastAPI()\n@app.get("/health")\ndef h(): return {"status":"ok"}\n@app.get("/")\ndef r(): return {"hello":"mitosis"}' > /app.py
EXPOSE 8080
CMD ["uvicorn","app:app","--host","0.0.0.0","--port","8080"]
```

Build + push:
```bash
docker buildx build --platform linux/arm64 \
  -t ghcr.io/fuadsn/mitosis-spike:latest \
  -f infra/Dockerfile.spike --push .
```

Deploy via Build API (uses auth pattern from Spike 0).

**Pass criteria**: deployment reaches `healthy` within 3 min; service URL returns `{"hello":"mitosis"}`.

### Spike 3 — Recursive Spawn Sanity (30 min)

The critical spike. Prove an agent can spawn another agent on Locus, *from inside a deployed worker*.

1. Modify the spike image to: on startup, count itself; if `DEPTH < 1`, spawn a child via Build API; report child's URL; exit.
2. Deploy with `DEPTH=0`. Watch a child appear automatically.

**Pass criteria**: depth-0 service spawns depth-1 service; depth-1 reaches `healthy`; depth-1 does NOT spawn further (depth bound holds).

If this fails, the entire architecture is invalid. Resolve before building anything else.

### Spike 4 — Tasks API Submission (10 min, optional for demo)

For the recorded backup video. Pull endpoint shape from `skill.md`, submit one tier-1 task, observe.

---

## Pitch Outline (5 min)

| Time | Beat | Visual |
|---|---|---|
| 0:00–0:30 | **Hook** — *"Pre-acquisition technical due diligence is a $200k engagement that takes 3 weeks. We're going to do it in 20 minutes for $20."* | Single line on a black slide. Shea & Co logo + price tag → arrow → Mitosis logo + price tag |
| 0:30–1:15 | **Problem** — every codebase decomposes differently; you can't pre-script the audit. Today firms throw analysts at it for weeks. | Two manual-process diagrams; comparison table from §The Pitch |
| 1:15–4:00 | **Live demo** — submit "audit this repo" with $5 budget. Watch root spawn. Root scans manifest, deterministically spawns 4 specialists (one per stack). 4 services appear on Locus. Each specialist's Claude decision shown live in monologue stream. Python's Claude says "auth.py is suspect" → spawns deep-analyzer grandchild. JS/Docker/GHA's Claude says "execute directly" → no further spawn. Grandchild finds the SQL injection. Reports flow back. Root synthesizes. Markdown DD report appears on screen. Tree collapses. | Split-screen: live tree visualizer (left) + DD report materializing (right) + Claude's split-decision JSON tooltip-able on hover |
| 4:00–4:40 | **Architecture reveal** — recursive spawn diagram. Three Locus primitives chained. Two split mechanisms (deterministic + LLM-decided) shown side by side. Fiscal bounding as the only governor of recursion. | Architecture diagram with annotation arrows |
| 4:40–5:00 | **Close** — *"Mitosis isn't an app, it's a primitive. Codebase DD today. Legal contract review, scientific literature synthesis, regulatory impact analysis — same engine, different prompts. Whatever decomposes, runs on Mitosis."* | Tagline + repo + demo link |

### Q&A Prep

- **"How do we know the LLM split decision is real and not scripted?"** → The demo shows it both ways. Deterministic split (root → 4 specialists) is rule-based by design, no LLM. Then JS/Docker/GHA specialists' Claude calls return `"action": "execute"` — visible in the monologue stream — while Python's Claude returns `"action": "split"` for auth.py specifically. Same code, different decision per branch, all live.
- **"What stops infinite recursion?"** → Fiscal substrate. Each split divides the budget. `MIN_BUDGET_FOR_SPLIT = $0.50` forces leaves. `MAX_DEPTH = 4` is belt-and-suspenders, but the wallet is the real bound.
- **"Cold start of 1-2 min per spawn — isn't that slow?"** → Yes, intentionally. Spawning is expensive, so agents only split when the LLM judges the task genuinely benefits from specialization. Cheap for narrow tasks, slower for big ones — that's the right shape. Counter-question: how long does Shea & Co take?
- **"What if a child crashes mid-task?"** → Reaper in the spawner (60s sweep) force-kills orphaned services and sends synthetic "failed" reports. Parent synthesizes with what it has.
- **"How is this different from LangGraph / CrewAI?"** → Those are libraries that orchestrate inside one process. Mitosis's agents are *separate deployments* with *separate wallets* and *real OS-level isolation*. No shared state, no shared budget, no shared crash domain. The difference between threads and microservices — and the reason for-pay platforms like Locus matter.
- **"Who pays for this?"** → Corp-dev teams running >5 acquisitions/year (current spend on technical DD: $250k–$1M/year). Cyber insurance underwriters (need scalable code risk scoring). Any team auditing a 3rd-party dependency before adoption.
- **"What's the moat once anyone can use Locus?"** → The architecture, prompt engineering, and trust calibration. The platform is a substrate; the IP is in the orchestration logic and the calibration of split decisions. We're early because no one else is treating BuildWithLocus as a programmable substrate for recursive computation.

---

## Judging Rubric Alignment

| Dimension | Points | Our play |
|---|---|---|
| **Technical Excellence** | 30 | Real BuildWithLocus per-agent deploys (not faked), real wrapped API metering, real self-DELETE, real fiscal bounding via on-chain wallet, dead-man's switch reaper |
| **Innovation & Creativity** | 25 | Recursive self-spawning agents is genuinely novel — not in the example list, no team will do this. Fiscal bounding *as the recursion governor* is the architectural insight. |
| **Business Impact** | 25 | Demo task (Series A DD) is a real workflow VCs in the room recognize. Memo is a real artifact, not a toy. The platform generalizes to any decomposable knowledge work. |
| **User Experience** | 20 | Tree visualizer is the product surface. Live spawning is the wow moment. Memo as artifact gives the demo a concrete payoff. |

---

## Risk Register

| # | Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|---|
| 0 | **Build API auth blocker (Spike 0) cannot be resolved** | M | **Catastrophic** | Read skill.md immediately; Discord ask in parallel; if hard-blocked by Mon noon, fall back to mocked-deploy demo (architecture story still valid, but loses "real Locus" credibility) |
| 1 | Cold start 1-2 min per spawn makes 6-agent demo take 6-10 min | H | H | Pre-deploy root agent before pitch starts; only the 5 specialists spawn live (parallel, ~2 min total); narrate the wait |
| 2 | LLM decision (split vs execute) is unreliable / inconsistent | M | M | Stable seed, low temperature, structured output validation. Demo target repo seeded so JS/Docker/GHA branches reliably get "execute" and Python's auth.py reliably gets "split". Pre-rehearse prompts on actual repo content. |
| 2b | Demo target repo doesn't reliably trigger the asymmetric split pattern (Python splits, others don't) | M | H | Pre-bake the repo: own the file contents. Test the decision prompt on these exact files 10x in a row before pitch — needs >=8/10 correct splits to use live; otherwise switch to a recorded demo segment. |
| 3 | Recursive spawn from inside a worker fails (Spike 3) | M | **Catastrophic** | Spike 3 IS the early-warning. If it fails, redesign as "central spawner orchestrates all spawns" instead of agent-spawns-agent |
| 4 | Synthesis quality is bad → memo looks junk | M | M | Strong synthesis prompt with explicit structure; demo memo on a startup with rich public info to reduce dependency on search quality |
| 5 | Credit grant comes back at $5 not $50 | H | H | $2 fixed + $3 wrapped + buffer = ~$5 minimum demo cost. Possible. If granted only $5, drop synthesis grandchild calls and use template-based synthesis |
| 6 | Live demo deploy fails during pitch | M | H | Pre-recorded 90s backup with voiceover ready; switch within 10s |
| 7 | Tree visualizer doesn't render fast enough | L | M | Server-render the static post-completion view as fallback |
| 8 | A specialist agent overspends its budget | M | L | Pre-flight cost check before each wrapped call; emergency-return on overspend; parent synthesizes with partial results |
| 9 | Two children try to spawn simultaneously and exceed total budget | L | M | Spawner enforces global service-count cap; parent's distributable budget is locked at split-time |
| 10 | Service spawn API has rate limits we don't know about | M | M | Spike 0 includes spawning 6 services in quick succession to verify; if rate-limited, serialize spawns at 5s intervals |

---

## Deferred / Out of Scope

- Multi-region deploys (us-east-1 only)
- Custom domains
- Git-push deploy (using pre-built image only)
- Multi-environment isolation (single production env)
- Persistent agent memory across runs (each task is fresh)
- Real Tasks API live in demo (recorded only)
- Generic adapter SDK for other domains (DD prompts hardcoded)
- Tree depth > 2 in live demo (recorded only)
- Cost optimization beyond pre-flight checks (no caching, no de-dup of wrapped calls across agents)

---

## Open Questions for Next Session

1. ⚠️ **Build API auth with beta `claw_dev_*` key** — the gating question. See Spike 0.
2. **Wrapped API base URL** — confirm `api.paywithlocus.com` vs `beta-api.paywithlocus.com`
3. **Tasks API endpoint shape** — pull from skill.md (only matters for backup recording)
4. **Per-service env var injection at create time** — confirm we can pass arbitrary env vars in the `POST /v1/services` body, not via separate variables PUT (avoids extra round-trip per spawn)
5. **Service spawn rate limits** — undocumented; Spike 3 will flush them out
6. **GHCR pull from Locus** — confirm public GHCR pulls work; if not, push to a registry Locus prefers
7. **`DELETE /v1/services/{id}` cascade behavior** — does it instantly tear down running containers, or schedule? Affects how fast the tree collapses visually.

---

## Carryover from Sentinel Planning

Salvaged from the previous (Sentinel) plan:

- **Repo**: `https://github.com/fuadsn/sentinel` (will rename to `mitosis` next)
- **API key + .env**: same beta key, unchanged
- **Gift code request**: `52606d86-658f-4442-a7b7-45b60f5ea0fd` filed Sun, $50 asked
- **`scripts/check_balance.sh`**: still works as-is
- **Build API auth blocker**: confirmed during Sentinel planning, carries over as Spike 0
- **Tech stack**: Python/FastAPI/ARM64/GHCR/Postgres — all unchanged
- **GHCR setup**: still needed before Spike 2

Discarded:
- All Shopify / T&S / per-tenant moderation logic
- 3-tenant brand fixtures
- Staged-degradation kill switch (Mitosis bounds via fiscal recursion, not staged degradation)
- Engine/adapter split (Mitosis is one cohesive system, not a platform)

---

## Next Session Quick Start

```bash
# 1. Orient
cd /Users/fuad/Developer/Projects/personal/sentinel  # or mitosis after rename
cat PLAN.md | less

# 2. Verify wallet + credit status
bash scripts/check_balance.sh

# 3. CRITICAL — resolve Build API auth (Spike 0). Nothing deploys until this passes.
curl -sS "$LOCUS_BUILD_API_URL/v1/projects" -H "Authorization: Bearer $LOCUS_API_KEY"
# If 401, fetch skill.md and read auth section
curl -sS "$LOCUS_BASE_URL/api/skills/skill.md" > /tmp/locus-skill.md

# 4. Once auth resolved, build + push the spike image
docker buildx build --platform linux/arm64 \
  -t ghcr.io/fuadsn/mitosis-spike:latest \
  -f infra/Dockerfile.spike --push .

# 5. Run Spikes 1 → 3 in order
# 6. Start spawner skeleton
```

**Session 1 success criteria**:
- Spike 0 passes (Build API auth pattern documented in `.env`)
- Spikes 1, 2, 3 all green
- Spawner FastAPI skeleton deploys to Locus and reaches `healthy`
- Agent FastAPI skeleton built (decide → execute path only, no split yet)
- One end-to-end submission works: spawner accepts a task, deploys one agent, agent makes one wrapped API call, reports back, self-deletes.

If Sun ends with all of the above, Mon is on track.
