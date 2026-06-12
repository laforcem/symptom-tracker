# Handoff — Symptom Tracker: kick off the build (iterative-development)

**Repo:** `/home/malc/repos/symptom-tracker` (local git, 5 commits; remote not yet created — the owner is creating it + settling a name).
**Date of handoff:** 2026-06-12

## What happened before this
A full design phase (superpowers `brainstorming` + `grill-with-docs`) produced an **approved spec** that then survived **three adversarial review passes** (two competing reviewers each). The design is done. Nothing is half-finished.

## Your job
Start implementation. In a **fresh session in the repo**, invoke the **`iterative-development`** orchestrator on the spec. It will chain: `extracting-requirements` → `scoping-the-simplest-core` (walking skeleton + roadmap) → the audited build loop. All process state lives in `docs/superpowers/iterations/`. Resume any time with "continue iterative development with the existing plan."

## Read these (do NOT duplicate — they are the source of truth)
- `CLAUDE.md` — orientation + the non-negotiable invariants, distilled.
- `docs/superpowers/specs/2026-06-11-symptom-tracker-design.md` — the spec (north star). Note especially §7 (data model), §8.5 (import), §11 (phasing), §12 (build framework + walking-skeleton scope note), §16 (decision latitude: fixed vs. judgment).
- `CONTEXT.md` — glossary; speak this language exactly.
- `docs/adr/0001`, `0002` — auth and data-model decisions of record.

## Critical reminders for the autonomous loop
- It runs **autonomously**; the owner injects changes via **interrupt at iteration boundaries**, not per-step approval.
- Honor: the **open-world rule** (gaps = unknown, never absent), **§16 latitude** (don't invent where the spec is fixed; pick simplest + record where it's silent), **auth hardening** (no in-app login, no published host port, fail-closed), and the **import discipline** (agent proposes / human reviews / acceptance oracle).
- **Walking-skeleton scope:** the historical import is front-loaded right after the skeleton and exercises nearly the whole logging model — so the skeleton + early iterations must stand up the full Observation/Tag model (is_none, present/severity/qualifier/dose, Field aggregation/reference_period, import provenance, validated in-process write path), not just a happy-path single-tag entry.
- **Human touchpoints** (loop hands back): the **frontend design** (separate track — see the flow-design handoff) and the **import acceptance/reconciliation review**.

## Practical
- **Usage:** owner is on Claude **Pro**; this loop is subagent-heavy and will likely throttle the 5-hr window. Owner has committed to **upgrading to Max reactively** when that happens. The on-disk state makes throttle-and-resume safe.
- **Deploy target** (later phases): Docker Compose on the homelab behind Caddy; image to GHCR via CI. Not needed for the walking skeleton.

## Suggested skills
- `iterative-development` (entry point; it invokes the others)
- It will chain `extracting-requirements`, `scoping-the-simplest-core`, `running-an-iteration`, `implementing-tasks`, `auditing-progress` itself.
