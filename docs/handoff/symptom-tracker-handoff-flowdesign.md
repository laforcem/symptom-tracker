# Handoff — Symptom Tracker: frontend flow design (Stage 1)

**Repo:** `/home/malc/repos/symptom-tracker`
**Date of handoff:** 2026-06-12
**Focus of next session:** help the owner design **user flows / information architecture + low-fi wireframes** for the app — Stage 1 of the spec's frontend-design process (§15). No code.

## The owner's design boundary (read first — this is firm)
- The owner **directs** the design; the agent **proposes 2–4 options with rationale and executes**, and **never finalizes a design unilaterally**. This is a stated creative boundary (the *imago dei* concern), recorded as a binding working agreement in spec §15.
- The owner enjoys **design and functionality thinking, not coding** — directs in plain language ("layouts 2 and 3," "move that," "warmer palette"); the agent does all implementation. AI is a high-velocity iterator, not the author.

## Workflow context (the owner just learned this; reinforce it)
Frontend design goes **structure → looks**, never the reverse:
1. **User flows / IA** — *not visual.* What screens exist, how you navigate between them, each screen's job.
2. **Low-fi wireframes** — boxes-and-labels layout, no color/type.
3. **Visual direction** — color/type/components (shadcn-svelte theme tokens).
4. **High-fidelity / build.**
(This maps to §15: Stage 1 = flows/wireframes, Stage 2 = visual, Stage 3 = build.) The owner's "what do I design first?" anxiety dissolves because **Stage 1 is the whole system of screens, not one screen's looks.**

## Where to start (already sketched with the owner)
**Flows first (the whole map), lightweight.** The screen set is roughly:
> **Log entry** (the core) · **Timeline** · **Entry detail/edit** · **Lab test** (upload + inline server-side PDF view) · **Appointment** · **Charts** · **Export** · **Review sweep** · **Admin** (fields / vocabulary / watched symptoms)

…with a mobile-first primary nav across the few main surfaces (candidate: **Log / Timeline / Charts / More**).

**Then the first screen to wireframe is the daily logging / intake flow** — most-used, MVP-core, and it forces the hard decisions: fast symptom + 3-level severity entry, vocabulary autocomplete with type-to-filter + "+ add new", one-tap "nothing to report", optional per-symptom "not today", optional end-of-day review sweep. **Timeline is the natural second** (unified, day-grouped, reverse-chronological).

## Ground the screens in the real requirements (reference, don't re-derive)
- Features per surface: spec §8 (`docs/superpowers/specs/2026-06-11-symptom-tracker-design.md`).
- What data each screen shows: spec §7 (data model) + `CONTEXT.md` glossary.
- The design process + tooling: spec §15.

## Tooling plan (from the design discussion)
- **Claude Design** (included in the owner's Pro plan, research preview) drives **Stages 1–2** — its canvas + chat/inline-comment loop *is* the options-first, plain-language workflow. Its **Export to Claude Code** handoff bundle then feeds **Stage 3** (implementation in the real SvelteKit/shadcn-svelte stack, where final tuning happens in-stack).
- Caveats: research preview (rough edges); design-system inheritance is strongest once the codebase/tokens exist (early exploration is generic); the handoff is a strong starting point Claude Code still adapts into idiomatic shadcn-svelte.
- If working *in this CLI* instead of Claude Design, the `prototype` skill (several UI variations toggleable from one route) and `frontend-design` skill (distinctive, non-generic aesthetics) fit Stages 1–3; but Stage 1 here is mostly collaborative flow/screen mapping, which can be done as plain proposals.

## Suggested skills
- `brainstorming` (if the owner wants to formalize flow decisions before sketching)
- `prototype` (UI-variations branch) and/or `frontend-design` — for when sketching/iterating actual layouts in-CLI
- Otherwise: Claude Design (external, owner's Pro plan) is the primary Stage 1–2 tool
