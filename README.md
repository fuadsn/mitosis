# Mitosis

> **Recursively self-replicating agents on Locus.**

**Paygentic Hackathon — Week 2 (BuildWithLocus)**

An agent receives a task. It decides whether to do the task itself, or split into N specialist children — each spawned as its own deployment on BuildWithLocus, each with a portion of the parent's USDC wallet. Children may spawn grandchildren. The tree expands and collapses dynamically, bounded only by the root budget.

`fork()` and `wait()` for the agent economy, with USDC as the only governor of recursion.

## What Locus uniquely enables

Three primitives, one substrate for autonomous decomposition:

- **BuildWithLocus** — agents *can* spawn agents because deployment is an API call
- **PayWithLocus** — every agent has a real wallet, so fiscal bounding is enforceable
- **Wrapped APIs** — pay-per-call USDC means costs are knowable at decision time

## Demo task

**Pre-acquisition codebase technical due diligence.** Submit a target repo URL with a budget. The root agent scans the repo manifest and deterministically spawns one specialist per detected stack (Python, JS, Docker, GitHub Actions). Each specialist's LLM autonomously decides whether to execute directly or to spawn a deep-analyzer grandchild for a suspect file. Real depth-3 recursion driven by what the agents actually find. Output: a markdown DD report covering security posture, dependency risks, and code quality.

A $200k engagement done by firms like Shea & Co in 2–4 weeks, executed in 20 minutes for $20.

See [`PLAN.md`](./PLAN.md) for full architecture, build sequence, demo plan, and risk register.

## Status

Pre-build — planning complete, execution starts in next session.
