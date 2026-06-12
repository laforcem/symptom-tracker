# Symptom Tracker — agent orientation

A self-hosted PWA for one chronically-ill patient to log daily symptoms with discipline, then
aggregate / chart / export / LLM-analyze that data toward a diagnosis. Greenfield.

## Read these before doing anything

- **The spec (north star):** [`docs/superpowers/specs/2026-06-11-symptom-tracker-design.md`](docs/superpowers/specs/2026-06-11-symptom-tracker-design.md)
- **Glossary (speak this language exactly):** [`CONTEXT.md`](CONTEXT.md) — Patient, Admin, Entry, Field, Observation, Tag, Vocabulary term, Watched symptom, Review sweep, Open-world rule, Day-level rollup, Analyzing agent / Implementing agent / Feature model.
- **Decisions of record:** [`docs/adr/`](docs/adr/) — 0001 auth via forward-auth proxy; 0002 hybrid field-registry data model.

## How this gets built

This spec is the human collateral for the **iterative-development** plugin. Build flow: invoke
`iterative-development` → `extracting-requirements` → `scoping-the-simplest-core` (walking
skeleton + roadmap) → audited loop. All process state lives in `docs/superpowers/iterations/`.
Resume any time with "continue iterative development with the existing plan."

## Non-negotiable invariants (don't let the loop drift off these)

- **Open-world rule** — a logged tag asserts presence; a gap is *unknown*, never *absent*. Only an
  affirmative act records absence. Charts/exports/analyzing-agent must honor it. (Spec §4, §7.2)
- **Decision latitude** — spec §16 says exactly what is *fixed* vs. where you may use judgment.
  When the spec is silent, pick the simplest thing consistent with §16 and record it.
- **Auth** — no in-app login (Authelia forward-auth); app publishes no host port, fails closed on
  missing `Remote-User`, re-derives role server-side. Dev auth shim is build-time excluded from prod. (§9, ADR-0001)
- **Import** — agent *proposes* the normalization, human *reviews* it; hard invariants only:
  validated write path, no fabrication, idempotent + reversible, acceptance oracle. (§8.5)
- **Frontend design** — the human directs; the implementing agent presents 2–4 options and executes,
  never finalizes design unilaterally. (§15)

## Stack (locked, spec §6)

SvelteKit (TS, `adapter-node`, PWA) · PostgreSQL · Drizzle · Zod · shadcn-svelte + Tailwind ·
LayerChart · Vitest + Playwright · pnpm · files on disk + Postgres metadata. Self-hosted via
Docker Compose behind Caddy; image published to GHCR by CI. Local dev: Postgres in Docker, app on
host (`pnpm dev`); dev auth shim via `DEV_USER`/`DEV_ROLE`.

## Conventions

- Match the surrounding code's style; keep files focused.
- This is medical data — correctness and honesty over cleverness.
