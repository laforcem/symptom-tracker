# Symptom Tracker — Product Requirements & Design

**Date:** 2026-06-11
**Status:** Approved — ready for iterative-development (survived three adversarial review passes)
**Glossary:** see [`/CONTEXT.md`](../../../CONTEXT.md) — the canonical domain language. This document uses those terms (Patient, Admin, Entry, Field, Observation, Tag, Vocabulary term, Watched symptom, Review sweep, Open-world rule) precisely.
**Decisions of record:** see [`/docs/adr/`](../../adr/).

---

## 1. Purpose

A chronically-ill patient (one person) logs daily symptoms and health factors with discipline so the data can be aggregated, charted, exported for doctors, and analyzed by an LLM to move toward a diagnosis. This replaces a free-text Google Sheet. The product's reason to exist is **low-friction, structured, honest data capture** plus **bespoke analysis** — the two things off-the-shelf trackers do worst.

**The existing data.** The Patient currently logs in a Google Sheet — one row per day (about a month so far) with category columns (Mental Symptoms, Physical Symptoms, Period?, Bowel Movements, Food, Water Intake, Vitamins, Exercise, Sleep, Red Light?, Other Factors), each cell a free-text comma-separated list (e.g. Physical Symptoms = "bloating, low-grade fever, joint pain"), with frequent blanks and "can't remember". A CSV export of this sheet is the source for the one-time import (§8.5) and informs the initial Field registry. The worked example in §7.1 is a real row from it.

## 2. Goals / Non-goals

**Goals**
- Effortless symptom capture from an iPhone (primary) and a Windows laptop, anywhere. The MVP is an **installable, online-only PWA** (app icon, full-screen, fast shell); a true **offline write-queue is a named later phase**, not an MVP promise — it interacts with SSR + forward-auth and needs a real sync/conflict design, so it is scoped deliberately rather than half-built.
- A schema the Admin can extend at runtime without code changes.
- Controlled, autocomplete-backed vocabularies that stay clean.
- Honest data: never infer the absence of a symptom from silence.
- A unified timeline, symptom/quantity charts, and doctor-ready exports.
- First-class LLM analysis over the data via an MCP server, and small in-app AI features via a provider-agnostic backend.
- Self-hosted on the existing homelab (Docker Compose behind Caddy), internet-accessible with strong auth.

**Non-goals**
- Multi-patient / multi-tenant support. There is exactly one Patient.
- A native mobile app (a PWA covers both devices).
- Doctor logins (doctors receive exports, not accounts).
- Hour-level analytical granularity (day-level is the analysis grain).
- Real medical advice. LLM output is hypothesis-generating, clearly labelled, never diagnosis.

## 3. Users & roles

- **Patient** — logs entries, uploads tests, views timeline/charts, runs exports.
- **Admin** — everything the Patient can do, plus schema management (fields), vocabulary cleanup, and data management. Identity and role come from an Authelia group; the app has no login of its own (see [ADR-0001](../../adr/0001-authentication-via-forward-auth-proxy.md)).

Doctors and clinics receive **exports**, not accounts; they are modeled as **Provider** records (§7.7), not users.

## 4. Guiding principles

1. **Open-world rule.** A logged tag asserts presence; a missing tag asserts nothing (unknown), never absence. Only an affirmative act ("nothing to report", a per-symptom "not today", or a review-sweep mark) records absence. At day-level rollup, presence wins over any same-day negative. Charts, exports, and the **analyzing agent** (the external LLM/agent that consumes data via the MCP server — see glossary) must honor this and treat gaps as *unknown*, never *absent*.
   *Why:* a tracker that infers "no symptom" from silence would feed doctors and the analyzing agent **fabricated negatives**. Honest "unknown" beats false precision, and an affirmatively-recorded absence is exactly what makes the central question — "did the intervention work / did the symptom stop?" — answerable instead of an ambiguous gap.
2. **Ship early, iterate in prod.** The first production deploy is the logging MVP, so real structured data starts accumulating immediately; analysis features land later as live updates.
   *Why:* data collection is the *only* time-sensitive thing here — charts and LLM analysis are worthless without months of data they don't yet have. Shipping logging first front-loads the sole urgent activity and lets every later feature launch against real data.
3. **The design is the Patient/Admin's.** The **implementing agent** presents options and executes; the human directs. It never finalizes a design unilaterally (see §15).
   *Why:* a deliberate creative boundary set by the owner — AI is a high-velocity iterator and implementer, not the author of the design. Recorded as a binding working agreement so the autonomous build loop honors it rather than overriding it.
4. **Data ownership.** Full-fidelity export (JSON) and standard backups keep the data portable and owned.
   *Why:* this is the Patient's medical data. It must never be trapped in the app — portability and backups keep it hers, and feed the broader ambition of consolidating real medical records (homelab issue #30).

## 5. Architecture overview

A single TypeScript **SvelteKit** application (server + client in one project), served as a **PWA**, talking to **PostgreSQL** via **Drizzle**. It is deployed as one container behind the existing **Caddy** reverse proxy; **Authelia** sits in front as a forward-auth proxy providing passkey login. Uploaded files live on a disk volume with metadata in Postgres. An **MCP server** exposes read tools over curated read-views plus one guarded write tool, consumed externally by any MCP-capable analyzing agent for analysis/import (Claude Desktop/Code on the Pro plan is the recommended default) and internally available to the app. The one-time historical import (§8.5) does **not** use the remote transport — it runs as an in-process / loopback-only call to the validated write service right after the walking skeleton, so it isn't gated on the deferred remote-MCP-auth decision. The **remote** MCP server (network-reachable read + write) is the later phase; it shares the app's **proxy-only network position and authentication** (no separate public port) and is never reachable unauthenticated (§8.6, §9).

```
iPhone / laptop ──HTTPS──> Caddy ──forward_auth──> Authelia (passkey login)
                              │  (Remote-User / Remote-Groups headers)
                              ▼
                       SvelteKit app (Node, adapter-node)
                         ├── PostgreSQL (Drizzle)
                         ├── files volume (PDFs, audio) served inline
                         └── MCP server (later) ── read-views + guarded write
                                                    ▲
                          MCP-capable agent (e.g. Claude Desktop/Code, Pro plan) — analysis
```

## 6. Tech stack (locked)

| Layer | Choice | Note |
|---|---|---|
| Meta-framework | **SvelteKit** (TypeScript, `adapter-node`) | SSR + server endpoints + PWA; one container. |
| DB | **PostgreSQL** | Relational fit + analytical read-views; `pgvector` headroom for later semantic search. |
| Data access | **Drizzle** + drizzle-kit | SQL-first (natural for read-views/pivots); typed; tracked migrations. |
| Validation/types | **Zod** | One schema validates API input and types the client; reused on the agent write path. |
| UI components | **shadcn-svelte + Tailwind** | Copy-in components the project owns; centralized theme tokens. |
| Charts | **LayerChart** (revisit at charting phase) | Svelte + d3; Observable Plot is the simpler alternative. |
| Tests | **Vitest** + **Playwright** | Unit/integration + E2E journeys (behavior evidence). |
| Package manager | **pnpm** | — |
| Files | disk volume + Postgres metadata, **write-once** | Served inline; never mutated in place. |
| Auth | **Authelia forward-auth + passkeys** | App reads trusted headers; no in-app login. |

**Why this stack (rationale, since it's locked):**
- **SvelteKit over Next/Nuxt** — the owner is a greenfield learner on all three, so "use what I know" favors none; the tiebreakers are the gentlest mental model, the least generated code (the owner reads and directs it; the budget is finite), the fewest footguns (no App-Router/RSC/caching complexity), a stable API, and proven viability for a self-hosted app (Immich). It's a personal tool, not a career bet, so framework job-market relevance is not a factor.
- **Meta-framework over a separate API + SPA** — the single strongest reason for a standalone API is *multiple first-class clients* (e.g. native mobile), which we deliberately ruled out by choosing a PWA. With one client and one developer, a separate API is ceremony without payoff; the MCP server, exports, and any future client still consume ordinary HTTP endpoints.
- **PostgreSQL over SQLite/Mongo** — the data is deeply relational and analytical (joins, window functions, day-rollups, read-views), which is Postgres's strength and Mongo's weakness; SQLite would work at this scale but Postgres matches the homelab ops pattern and keeps `pgvector` open for later semantic search. See [ADR-0002](../../adr/0002-hybrid-field-registry-data-model.md).
- **Authelia forward-auth over app-native auth** — passkeys for free, only this app's subdomain protected, and a path to homelab-wide SSO; the app owns zero auth code. See [ADR-0001](../../adr/0001-authentication-via-forward-auth-proxy.md).

**Validation boundaries.** Zod validates at **trust boundaries only** — HTTP endpoints, file uploads, the agent write tool, and any imported or LLM-produced data. Internal, already-validated data is trusted and not re-checked. This keeps validation where untrusted input actually enters. The shared schema also enforces basic value sanity (quantities ≥ 0; `occurred_on` not absurdly in the future; severity ∈ {1,2,3}; `choice_value` in the field's option set; upload MIME/size limits) so a garbage parse from the import or the agent can't become a fabricated data point.

**Local development (locked):** hybrid — Postgres runs in Docker Compose; the SvelteKit app runs natively on the host (WSL2, repo on the Linux filesystem) via `pnpm dev` for fast HMR. A **dev auth shim** in `hooks.server.ts` fakes `Remote-User`/`Remote-Groups` from env vars (`DEV_USER`, `DEV_ROLE`) so both roles can be exercised without Authelia. **The shim must be excluded from the production build at *build time* (fixed)** — gated by `import.meta.env.DEV`/a compile-time flag, never merely an env var. A header-faking path that can be re-enabled by a misconfigured env var in production would silently defeat the entire fail-closed auth model (§9). A Caddy+Authelia compose profile for real-auth integration testing is deferred to the deployment phase. Production parity is verified by building and running the production container before deploy.

## 7. Data model (the core)

A **hybrid** model: a typed field registry + controlled vocabularies + typed observations. Runtime-extensible where the data is heterogeneous; fixed tables where it is homogeneous. See [ADR-0002](../../adr/0002-hybrid-field-registry-data-model.md).

### 7.1 Logging core

- **Field** — admin-managed schema registry row. Declares a `type`: `tag_set`, `quantity` (+unit), `choice`, `text`, or `boolean`; plus label, key, display order, active flag, whether it supports severity, and a **`reference_period`** (`same_day` default, or `prior_night` — see Time below). For `quantity`, the **unit is declared at the Field level and is authoritative** (observations inherit it; no divergent unit, enforced at write), plus an **`aggregation`** attribute (`sum` for additive quantities like Water Intake | `latest` | `none`) governing day-rollup (§7.2). A `boolean` field is three-state via its value: `true` = present, `false` = affirmed-absent, no observation = unknown (it does **not** use `is_none`). Adding a field = inserting a row, no deploy.
- **Vocabulary term** — an autocomplete value scoped to one Field (e.g. "bloating" under Physical Symptoms). Created as the Patient types new values; Admin merges/deletes typos. Carries a normalized label for dedup.
- **Entry** — a logging event at a **local civil date** (required) and an optional local time. Multiple entries per day are allowed (supports retroactive, partial, and split-across-the-day logging). See **Time & timezone** below.
- **Observation** — the value of one Field within one Entry, stored in the typed slot matching the Field's type:
  - `quantity` → `numeric_value` (unit inherited from the Field) + an optional free-text **qualifier** capturing the measurement site/method (e.g. "left ear", "right ear", "oral")
  - `choice` → `choice_value` + an optional free-text **qualifier** (so clinically significant detail like Period "bright red blood" is captured, never dropped)
  - `text` → `text_value`
  - `boolean` → `bool_value`
  - `tag_set` → child **Tag** rows (no single value)

  Usually there is **one Observation per Field per Entry**, but a `quantity` field may hold **multiple observations in one Entry when they carry distinct qualifiers** — e.g. two ear-thermometer readings, `99.1°F (left ear)` and `98.7°F (right ear)`, are both recorded rather than one being dropped or the two averaged into a number never measured. How charts/exports reduce multiple readings to a single point (take the higher, per-site series, etc.) is a deferred analysis decision (§16); the raw readings are always preserved.
- **Tag** — one item on a `tag_set` Observation; references a Vocabulary term; a `present` flag (see §7.2); and optional **severity** (ordinal `1=mild | 2=moderate | 3=severe`), an optional free-text **qualifier** (e.g. "(knees)"), and an optional free-text **dose** (e.g. "3 pills", "1 tsp" — for Vitamins; exact dose representation is judgment, see §16).

**Import provenance.** Entries (and their child Observations/Tags) created by the one-time import carry an **`import_batch` id** and a per-row **source key** (hash of source row + field). These are null for normally-logged data. They make the import idempotent and let the whole batch be filtered in the timeline and rolled back as a unit, cascading to any vocabulary terms it created (§8.5).

**Time & timezone.** `occurred_on` is a **patient-asserted civil date** — stored as a bare `DATE`, never derived from a UTC instant or the device clock. It defaults to "today" in a single configured patient-home timezone (the Patient lives in Colorado → default `America/Denver`) but is an editable date field, so travel can never silently shift which day an entry lands on (the legacy data shows WA↔CO travel; the date the Patient picks is authoritative). Day-level rollup groups by that stored civil date; the home timezone only governs the default and any "today/this-week" relative ranges, not a conversion of stored dates. Optional times are display-only local times and do not affect day attribution. **`reference_period`** handles fields that describe a different window than the entry date: a `prior_night` field (seed: **Sleep**, the sheet's "Sleep (the night before)") logged on date *D* describes the night ending the morning of *D*. Charts and the analyzing agent attribute `prior_night` values to *D* with that meaning, so sleep aligns correctly as a *precursor* to day-*D* symptoms rather than being silently off by one.

**Worked example — the real 6/7/2026 row (verbatim from the CSV) becomes:**
```
Entry  occurred_on = 2026-06-07 (America/Denver),  time = (none)
 ├─ Obs [Mental Symptoms / tag_set]    tags → anxiety, brain fog, cognitive difficulty, depression, dread
 ├─ Obs [Physical Symptoms / tag_set]  tags → gut pain, low-grade fever, feeling super hot, sluggish,
 │                                             fatigue, exhausted, passing gas, bloating, joint pain
 ├─ Obs [Vitamins / tag_set]           tags → beef liver pills (dose "3"), solar D gem (dose "1"),
 │                                             vitamin K full-spectrum (dose "1"),
 │                                             bio-kult probiotics (dose "4"), s-boulardii (dose "2")
 ├─ Obs [Period / choice]              is_none  (affirmed absence — "no")
 ├─ Obs [Bowel Movements / choice]     is_none  (affirmed absence — "no stool was passed")
 ├─ Obs [Exercise / text]              "15 minute walk"
 ├─ Obs [Sleep / text, prior_night]    "excellent sleep quality, woke up 2x in the morning to pee"
 │                                      (describes the night of 6/6 → morning of 6/7)
 ├─ Obs [Food / text]                  "beef stew, eggs, dill, ... siete chips"
 ├─ Obs [Red Light / text]             "none"
 ├─ (Water Intake: "can't remember" → no Observation → unknown)
 └─ (Temperature: no such column in the legacy sheet → no Observation → unknown.
     Fever is logged as a *symptom* under Physical Symptoms, never as a thermometer reading.)
```
*(Illustrative of the mapping; tags shown verbatim from the source row.)*

**Initial field registry (seed).** The app launches with these fields, derived from the existing sheet (§1). Types are chosen to structure what's cleanly structurable and leave narrative as text rather than over-structuring it. **All of this is Admin-editable post-launch** (add/rename/retype/reorder); it is a starting point, not a fixed schema.

| Field | Type | Notes |
|---|---|---|
| Temperature | `quantity` (°F), `aggregation: none` | **No legacy data** — the sheet has no temperature column; the import maps nothing here. Populated by *future* logging only; today fever is a Physical Symptom, not a reading. (The owner explicitly wants thermometer tracking going forward.) Ear-thermometer readings vary by side, so both can be recorded in one entry via the site qualifier ("left ear"/"right ear"); `aggregation: none` keeps both rather than collapsing them. |
| Water Intake | `quantity` (oz), `aggregation: sum` | import parses the leading number; the textual source ("filtered", "well-water", "+ tea") is **kept in the qualifier** (environmental signal), never silently dropped; multiple same-day entries **sum** to a daily total |
| Mental Symptoms | `tag_set` | severity-enabled |
| Physical Symptoms | `tag_set` | severity-enabled |
| Vitamins | `tag_set` | tags carry an optional **dose** (e.g. "3 pills", "1 tsp") |
| Period | `choice` + qualifier | options are *positive states only*, including a neutral **"present (unspecified)"** so a legacy "yes, not super painful" maps without fabricating a flow level (spotting / light / heavy / painful / present-unspecified); "no" → `is_none` (§7.2), not an option; qualifier holds verbatim detail like "bright red blood" / "not super painful"; Admin-editable |
| Bowel Movements | `choice` + qualifier | count × form are orthogonal — options cover *form* (e.g. formed / loose / pebble-like / hard); "no stool was passed" → `is_none`; qualifier holds verbatim count/form detail ("1× harder pebble-like"); may later be split into separate count + form **fields** (a parallel-field migration, not an in-place retype — §8.1); Admin-editable |
| Food | `text` | narrative (long ingredient lists) — not structured into tags |
| Exercise | `text` | narrative |
| Sleep | `text`, `prior_night` | narrative quality; a *separate* `quantity` (hours) field may be added later (parallel field, not an in-place retype — §8.1) |
| Red Light | `text` | device + duration narrative; a *separate* `tag_set` field of devices may be added later (parallel field, not an in-place retype) |
| Other Factors | `text` | narrative life events / context |

### 7.2 Absence semantics (open-world)

Three states per symptom per day:
- **Present** — positively logged.
- **Absent** — affirmatively recorded ("nothing to report" for a whole field; or a per-symptom "not today"/review-sweep mark).
- **Unknown** — neither. This is the default; never converted to "absent".

Negatives are timestamped Observations like positives, scoped to their moment, not the whole day. Morning "no bloating" + evening "bloating" is **not** a contradiction — it is onset data; at day rollup **presence wins**. Mistaken inputs are corrected by ordinary edit/delete. **UX affordance:** when the Patient adds a symptom that already has a same-day negative, the UI surfaces it ("you marked no bloating earlier — developed later, or remove the earlier note as a mistake?").

**How absence is stored (recommended representation).** Field-level "nothing to report" → an Observation on that `tag_set` field with an `is_none` flag set and zero Tags. Per-symptom "not today" / review-sweep "absent" → a **Tag with `present = false`** (an occurrence is `present = true` and may carry severity; an affirmed-absence is `present = false`). The three states then fall out cleanly: a `present = true` tag → *present*; a `present = false` tag, or an `is_none` field → *affirmed absent*; no tag and no `is_none` → *unknown*. The implementing agent may refine the exact columns **provided the three states stay distinguishable and presence-wins holds at rollup.**

**Validity matrix (fixed):** a `present = false` tag must have null severity and null dose (an absence has no intensity); an `is_none` Observation must have zero Tags; presence-wins at rollup evaluates **only** `present = true` tags. The same-day-negative UX affordance (above) resolves a field that would otherwise carry both a negative and a later occurrence. **`is_none` and `present = false` tags are mutually exclusive on one Observation:** `is_none` means "nothing to report for the *whole* field" (zero tags); a **review sweep** that marks specific watched symptoms absent produces an Observation carrying one **`present = false` tag per swept symptom**, *not* `is_none`. Reserve `is_none` for the explicit whole-field "nothing to report".

**Three-state applies to *every* field type, not just `tag_set` (fixed).** The same present/absent/unknown model holds for `quantity`, `choice`, `text`, and `boolean`:
- **Present** = an Observation carrying a typed value.
- **Affirmed absent** = an Observation flagged **`is_none`** (carrying no typed value). `is_none` generalizes to all types. So "Period: no" and Bowel "no stool was passed" are recorded as `is_none` Observations on those fields — an affirmed *absence* — **not** as a choice value "no"/"none". A `choice` field's options are therefore the *positive* states only (Period: spotting/light/heavy/painful); "none/no" is `is_none`.
- **Unknown** = no Observation.
- **Exception — `boolean`:** `false` *is* the affirmed-absent state, so a boolean field uses `true` = present / `false` = affirmed-absent / no observation = unknown, and does **not** use `is_none` (which would create a spurious fourth state).

**Day-level rollup for non-tag fields (fixed):** presence wins uniformly — any present value on a day overrides a same-day `is_none` for that field. Reduction of multiple same-day *present* observations of one field is **per field type**: an **additive** `quantity` field (e.g. Water Intake) **sums** the day's values; a non-additive `quantity` and all `choice`/`text`/`boolean` fields take the **latest by time, then latest-created** as the day's representative value. **All observations are always retained**, nothing is discarded. This is driven by a Field **`aggregation`** attribute (`sum` | `latest` | `none`; see §7.1). Note the legacy import creates **one entry per day**, so the tie-break never bites historical data; it governs live multi-entry days. For `quantity`, multiple same-entry readings distinguished by qualifier (e.g. both ears) are preserved and reduced only at chart/export time (deferred, §16).

### 7.3 Watched symptoms & review sweep

- **Watched symptom** — a curated subset of vocabulary terms the Patient actively monitors.
- **Review sweep** — an optional fast end-of-day pass listing only watched symptoms, each marked present / absent / skip in seconds — the practical mechanism that turns "unknown" days into affirmatively present/absent data without fabricating absence.

### 7.4 Read model for analysis

Curated, read-only SQL views in **tidy/long** form (e.g. `v_daily_observations`: `date, field_key, term, severity, numeric_value, unit, text_value`) are the contract for charts, export, and the LLM. The normalized write tables are never queried directly by analysis. Lab results (§7.6) surface into the same time-series read model so markers can be overlaid with symptoms. The "wide daily" spreadsheet shape is produced in application code (pivot over the live field registry), used only by the wide-CSV export.

Because Fields are runtime-mutable, **the views are plain (dynamic) over the live registry (fixed)** — a newly added Field appears in `v_daily_observations` with no separate refresh step to own. *Inactive* (deactivated) Fields are excluded from default analysis views but their historical rows remain queryable; the MCP schema-introspection tool reports the registry as of query time (point-in-time), so an agent re-checks the schema rather than trusting a cached copy mid-session. A materialized view is permitted later *only* for a measured performance need and *only* with an explicit refresh trigger on field/registry changes (judgment, §16).

Indexes: `entry(occurred_on)`, `observation(field_id)`, `observation_tag(vocabulary_term_id)`.

### 7.5 Files

Generic File entity: id, path/key, mime, size, original name, kind, plus FK from the owning record. Bytes on a mounted volume; metadata in Postgres. **Write-once** (re-upload creates a new file). Served **inline** via an app endpoint (`Content-Disposition: inline`) — a server-side copy rendered in a new browser tab, never a forced download of a local copy. **On delete:** when an owning record (LabTest, Appointment) is deleted, its attached file bytes are deleted with it (no orphaned files accumulating on the volume or in backups). **Uploads are validated** (allowed MIME types — PDF, audio, image — and a size limit) at the boundary.

### 7.6 Lab/blood tests (fixed-schema)

- **LabTest** — `test_date`, panel/name, optional link to ordering Provider, source PDF (File), notes.
- **LabResult** (child) — `analyte` (FK to Analyte catalog), `value`, `unit`, `reference_range_low/high`, `flag` (H/L/normal). The *shape* is fixed; *which* analytes are recorded is data, varying per test.
- **Analyte catalog** — canonical analyte identities (canonical name, default unit, typical range) so the same marker is recognized across tests and is reliably chartable; Admin dedupes name variants.

### 7.7 Appointments & providers (fixed-schema)

- **Provider** — name, office/clinic, specialty, optional contact. Reusable; linkable as LabTest orderer.
- **Appointment** — `appointment_date`, link to Provider, free-text summary/notes, optional audio attachment + transcript, optional links to discussed/ordered LabTests.

New fixed-table fields (cost, follow-up date, diagnosis, prescriptions) are added later via routine non-destructive migrations — not speculatively now.

## 8. Domains & features

### 8.1 Logging
Entry input defaults to current date/time; time optional; all fields optional — submit whatever, whenever. Field values autocomplete from the vocabulary with type-to-filter and an "+ add new" option; new values are saved for reuse. Per-tag severity (3-level) and qualifier. One-tap "nothing to report" per `tag_set` field. Optional per-symptom "not today". Optional end-of-day review sweep over watched symptoms. Entries and observations are editable/deletable (Patient and Admin).

**Vocabulary management.** Duplicate detection in autocomplete uses each term's normalized label (case/whitespace-folded). **Merging** two terms re-points every Tag from the duplicate to the canonical term, then retires the duplicate (so existing data is preserved, not orphaned); merges are **intra-field only** (terms are field-scoped). **Deleting** a term that still has Tags is blocked until merged or those Tags are removed. If a merged/deleted term is a **watched symptom** (§7.3), the watch follows the merge to the canonical term (or is dropped on delete) — never left dangling. **Changing a Field's type** is guarded: it is blocked while the Field has observations (their typed values can't be silently reinterpreted). The supported way to evolve a field (e.g. add a Sleep-hours `quantity`, split Bowel into count+form) is therefore to **add a new parallel field** and optionally migrate — *not* an in-place retype. The seed notes that say a field "may become X later" mean exactly this parallel-field path. Admin-only.

**Edit/delete semantics.** Edits to entries/observations are **immediate and hard** by default (this is the owner's own data; correcting a mistyped entry should be simple). **Deleting a record that owns irreplaceable file bytes** (a LabTest's PDF, an Appointment's audio) requires an explicit confirmation and a short grace window (soft-delete + purge) — those source documents can't be re-created, so they get this protection even though general soft-delete stays deferred. **Concurrency:** records carry an optimistic-lock **version that read responses and the MCP read tools expose and every write must echo**; a write against a stale version is rejected and the editor is shown current state to retry, so a second device or the agent can't silently clobber an edit. The MCP write tool is **insert-only** (it creates entries/observations; it does not update or delete), so optimistic-locking governs UI edits while the write tool uses an idempotency key for safe create-retries (§8.6). An audit trail / general soft-delete is **optional and deferred** (judgment, see §16).

### 8.2 Timeline (read view)
A **unified** reverse-chronological feed of entries, lab tests, and appointments, **grouped by day**, newest first. Each day header nests its entries/tests/appointments as typed cards. Filters (MVP): date range, record type, tag, and **import batch** (so the imported data can be reviewed, accepted, or rolled back as a unit — §8.5). Free-text search deferred. Tap to edit/delete. Paginated/infinite scroll.

### 8.3 Charts / aggregation
- **Tier 1 (foundation):** single-series — a `quantity` over time (line); a symptom over time (**calendar heatmap**). The encoding must render **five visually distinct states**, since most real/imported tags have *no* severity: present-with-severity on a 3-step ramp (mild→moderate→severe), **present-without-severity** as a distinct neutral "present" color (not on the ramp), **affirmed-absent** as a distinct "zero" shade, and **unknown** as blank/hatched. Crucially, affirmed-absent and unknown must never look the same (that would violate the open-world rule). A day cell reduces multiple same-day tags to **max severity** (the worst reading wins). The `quantity` line chart **must not interpolate across unknown days** — gaps are gaps, not straight lines implying values that were never measured (Temperature, with no legacy data, is mostly gaps). Relative (last 7/30/90d) and absolute date-range selection; day-level grain.
- **Tier 2:** **overlay** multiple series on one chart over the same window (e.g. joint-pain severity + CRP + temperature), dual-axis where units differ — the core correlation view.
- **Tier 3 (deferred):** saved charts + drag-and-drop auto-updating dashboard.

### 8.4 Export
- **PDF report** — curated, doctor-ready: situation header, selected range, symptom summary (frequencies + severity trends), lab-results table, appointment history. Optional toggle to embed selected charts (off by default). Generated server-side by rendering an HTML template via headless Chromium (reuse Playwright). **Build requirement:** Chromium and its system libraries must be present in the *production* image (not just the dev host) and PDF export must be verified **in-container** — a slim Node image will silently lack them and crash at export time.
- **CSV** — default **tidy/long**; optional **wide daily** (pivot in app code).
- **JSON** — full-fidelity structured dump preserving relationships; for data ownership and as the cleanest input to hand an LLM.
- All exports take a date range + section/field selection.

### 8.5 CSV import (one-time, post-deploy)
No dedicated import *feature*. The existing Google Sheet is imported once via the agent, which introspects the self-describing schema, normalizes the messy free-text, and writes through the **validated** write path (never raw inserts).

**Decoupled from the remote MCP (fixed).** The import runs as a **one-time, in-process / loopback-only** invocation of the validated write service — it does **not** use the remote (network-reachable) MCP transport, so it is **not** blocked by the deferred remote-MCP-auth decision (§14). Only a minimal read-back (counts/dupes/vocab) is needed to self-verify.

**Hard invariants (fixed):**
- Writes go through the validated write path; never raw inserts.
- **No fabrication** — the importer never invents a value, a severity, or a present symptom the source doesn't clearly state.
- **Idempotent & reversible** — every imported Entry/Observation/Tag carries an **`import_batch` id** and a **source key keyed at the Observation/Tag grain** (`hash(source row + field + ordinal-within-cell)`, so the nine tags of one Physical-Symptoms cell are individually de-dupable), making a crashed/retried run non-duplicating. The whole batch can be deleted (rollback), cascading to vocabulary terms it created — **except** any term now also referenced by non-batch (real-logged) Tags or merged into a live canonical term, which is kept. Rejected imports are rolled back **before the next nightly backup** so garbage isn't captured off-site.
- **Acceptance oracle (this is how "no fabrication" is *tested*, not just asserted)** — a **hand-verified expected mapping** of the 38-row CSV is the import's acceptance fixture, plus a **reconciliation report**: every source cell must be accounted for as *mapped*, *dropped* (with reason), or *unknown* — none silently lost or invented. The agent's **proposed vocabulary-term list is reviewed and merged as part of acceptance, before the batch commits**, so typo-variants (bio-kult/biokult, monterey/monterray, protien) don't pollute autocomplete from day one.

**Normalization is the agent's *heuristic, applied under mandatory human review*** — not rigid rules, because the data is too messy for rigid rules to be safe (a naive severity-strip, for instance, corrupts real symptoms). The agent proposes a mapping; the human verifies it in the timeline before acceptance. Guidance:
- **Column mapping:** the CSV headers (`Period?`, `Red Light?`, `Sleep (the night before)`, `Bowel Movements`, …) map to the seed field keys (`period`, `red_light`, `sleep`, `bowel_movements`, …); this explicit header→key map is part of the import fixture, and the field identity used in source keys is the registry key (stable).
- **Dates:** parse `M/D/YYYY` as `America/Denver` civil dates **verbatim** — no timezone conversion, travel rows not reinterpreted. Use an RFC-4180 parser (cells contain commas and quotes); sparse/partial rows are valid (only a structural parse failure aborts); assert the expected row count.
- **Symptoms → tags + severity.** This is a *proposal the human reviews*, not a deterministic rule (a rigid severity-strip provably corrupts this data). The agent works over the **whole sheet** (so it knows which base terms exist before deciding) and is *conservative*: it only proposes a severity ordinal for a clear leading intensity modifier on a symptom (`mild`/`low-grade`→1, `moderate`→2, `severe`→3), splitting "severe X" into base term `X` + severity 3 — and otherwise leaves the phrase intact with `severity = null` rather than inventing a split. It does **not** treat `extreme`/`intense` as severity (non-symptomatic: "intense mood swings"); it keeps `low-grade fever` as its own canonical term (never `fever` sev 1); and when the same base term appears twice in one cell at different intensities ("bloating, extreme bloating") it proposes a single tag at the higher severity. Heavy spelling-variant dedup (bio-kult/biokult, monterey/monterray) is surfaced for the human merge that gates acceptance (above). Every such proposal is confirmed in the reconciliation review before commit.
- **Non-symptoms in symptom cells:** affect/normalcy phrases are **not** present symptoms. "felt okay/happy/super ecstatic" → dropped or routed to Other Factors (human decides); **"normal temperature"** → the agent *proposes* a `present = false` (affirmed-absent) tag for the Patient's canonical fever term ("low-grade fever", the one she actually uses — not a bare "fever" that appears nowhere), for human confirmation; if the target term is ambiguous, **drop rather than guess**.
- **Vitamins → tags with dose:** keep the **whole dose phrase including unit** ("1 tsp …", "3 … pills"); a supplement named with a digit ("5-MTHF") is a *name*, not a dose; unit-less entries get `dose = null` meaning "taken, amount unstated" (still present).
- **Period / Bowel → choice or `is_none`:** "no"/"no stool was passed" → `is_none` (§7.2); positive states → the closest option, with extra detail (count, "bright red blood") in the qualifier.
- **Narrative columns (Food, Exercise, Sleep, Red Light, Other Factors) → text** (Sleep is `prior_night`). Literal "none"/"no" imports as **verbatim text** (a present value), distinct from a blank (unknown).
- **Water Intake → quantity:** parse the leading number; keep the textual unit/source ("filtered", "well-water", "+ tea") in the **qualifier** rather than discarding it (environmental signal).
- **Blank or "can't remember"/"forgot to log" → no Observation (unknown)** — both map to unknown (decided). A large fraction of legacy cells are blank (a sheet blank meant "didn't fill this column"), so **expect high unknown density in historical data** — this is by design, and the reconciliation report surfaces the rate so the human and the analyzing agent aren't surprised (gaps are unknown, never absent).

**Cutover (fixed).** The sheet is **frozen at a stated cutover date**, imported **once**, and primary logging then switches to the app — there is no ongoing two-way sync, and re-importing *edited* historical rows is out of scope (post-cutover corrections happen in the app). Because the MVP ships logging first, the import does **not** assume an empty database: it relies on the `import_batch` marker + human review to coexist with any entries already logged, rather than a "verified-empty" precondition.

**Priority.** Because an empty timeline at launch is unmotivating and charts/export are only meaningful with data, the import is prioritized immediately after the walking skeleton, ahead of charts/export. It needs only the in-process write path + minimal read-back; the human reviews the result in the timeline, where a **batch filter** lets them view, accept, or roll back the import. The full *remote* MCP read surface for the ongoing analyzing agent follows later.

### 8.6 LLM analysis & in-app AI
- **MCP server** — the single integration seam. **Read tools** (schema introspection, query observations/read-views, lab results, timeline) backed by the read-only views; **one guarded write tool** — **insert-only** (creates entries/observations through the Zod-validated path; never updates or deletes, so corrections happen in the UI). Safe-retry idempotency: the import uses `import_batch` + source key; **live (non-import) creates use a client-supplied request-id** so a retried create after a network blip doesn't double-insert. **Hard prerequisite (fixed):** the write tool — and any network-reachable MCP transport — is **never exposed without authentication**, from its very first deployment (it is a write path into medical data on an internet-exposed host). The endpoint auth *mechanism* is deferred (§14), but "no auth" is never an acceptable interim state. The **one-time import invokes this write path in-process / loopback-only** (not the remote transport), so it is not gated on that deferred decision (§8.5). When the remote transport ships, it sits behind the app's **proxy-only network position and the same auth** (§9) — not a second public port.
- **Heavy diagnostic analysis** — the MCP server is consumed by any MCP-capable **analyzing agent**; it is open by design, not Claude-exclusive. Using Claude Desktop/Code on the existing Pro plan is the **recommended default** (no metered cost, most capable), but any MCP client works. The agent produces tentative, clearly-labelled hypotheses and next-step suggestions; it is instructed on the open-world semantics (gaps = unknown, not absent) and reports three-state honestly.
- **In-app subtle features (later)** — narrow data-extraction/summarization (lab-PDF field extraction, vocab-match suggestions, and — gated on transcription existing — appointment-transcript summaries). These call the **feature model** (see glossary) via the **OpenAI-compatible chat-completions interface** (configurable `base_url`/`model`/`api_key`), so they run against OpenRouter, a local Ollama on the planned GPU box, the Claude API, etc. Long-running work runs on a background-job seam, never in a web request.
- **Local inferencing is an intended target, not hypothetical.** A home machine with a capable GPU is planned; running local models (via Ollama / an OpenAI-compatible endpoint) for the feature model — especially the privacy-sensitive, repetitive extraction and transcription work — is expected to be attempted. The provider-agnostic integration must therefore work with a local backend from the start, not only hosted APIs.
- **Data egress note:** routing health data to an external LLM is a deliberate, consensual choice; the local-GPU backend is the privacy-preserving option for routine repetitive features.
- **Third-party PII.** Food, Other Factors, **and appointment transcripts** narrate identifiable third parties and events (named people, a friend's pregnancy, a family member's surgery, and — in transcripts — clinicians by name). All of it flows verbatim into the JSON export and the MCP read tools, and thence to whatever analyzing agent is connected. Policy for now: **accept and document** — it is the Patient's own narrative/records, and the egress is consensual. The deferred **redaction toggle** (§14) for exports / agent views covers transcript text as well, should sharing widen beyond the household.

### 8.7 Audio transcription (deferred, optional)
Audio upload/playback is in scope now (§7.7). Transcription is a **late, optional phase** and gates the transcript-summary feature. Recommended path when built: **Whisper** for transcription; **speaker diarization** via **WhisperX (Whisper + pyannote)** run as a background job on the GPU box, or a managed API (AssemblyAI/Deepgram) for zero-setup. Schema stores `audio_file` + `transcript_text` from day one, with per-segment speaker/timing data as an **opaque JSON blob** (not relational columns) until the engine is chosen — WhisperX and managed APIs emit different segment/speaker structures, so committing relational columns now would guess the wrong shape.

## 9. Authentication & access
Internet-exposed behind Caddy with **Authelia forward-auth** and **passkey (WebAuthn) login** — tap → Face ID on iPhone, Windows Hello / cross-device on the laptop; passkeys may live in Bitwarden. No public sign-up; accounts seeded by Admin in Authelia. The app has **no login UI**: Caddy `forward_auth`s only the app's subdomain (other homelab apps untouched), and the app reads identity from trusted `Remote-User`/`Remote-Groups` headers, mapping the `admins` group → Admin role. **Header trust must be hardened (fixed):** the app container **publishes no host port** — it is reachable *only* over the internal proxy network, so the forwarded headers cannot be spoofed by a direct hit; Caddy sets `trusted_proxies`; and the app **fails closed** — any request arriving without a valid `Remote-User` is rejected (not treated as anonymous). Every mutating and admin endpoint re-derives the role from the headers server-side rather than trusting the client. Recovery: register passkeys on multiple devices; Admin re-issues enrollment. **PWA capture safety (fixed).** The Patient is mobile-first and logs from anywhere, including dead zones — so the MVP must **never lose a typed entry to a connectivity or session lapse.** On offline/connectivity loss or an expired Authelia session, the app **preserves the in-progress entry as a local draft** (the entry being composed — not a full multi-entry sync queue) and submits it once connectivity/session returns; it never reports a save as succeeded unless the server confirmed it, and never silently discards input. A background `fetch` receiving an auth *redirect* (not a clean 401) is treated as session-expired and routes the user through an in-app re-auth that returns them to the **draft-preserved** entry — no redirect loop in the standalone window, no lost input. (This single-entry local-draft safety net is in the MVP; the broader **multi-entry offline write-queue with sync/conflict handling** remains the deferred phase, §14.) See [ADR-0001](../../adr/0001-authentication-via-forward-auth-proxy.md). IP allowlisting is recorded as a *home-only* deployment fallback, not the primary control (the Patient roams on mobile); fail2ban/crowdsec-style rate limiting and optional geo-limiting fit a roaming user better.

## 10. Deployment (principles; wiring at deploy phase)
One app container + one Postgres container in a Compose file in the `laforcem/homelab` repo, slotted into the existing Caddy wildcard-TLS pattern (a new `symptoms.{$DOMAIN}` route with `forward_auth` → Authelia).

**Build & image distribution.** The app is packaged as a container image **published to GHCR** (`ghcr.io/laforcem/symptom-tracker`) by **GitHub Actions** in this repo, tagged by **semantic version** on release. The `laforcem/homelab` Compose file references the image **by tag**; the production node simply **pulls** it (like Immich/Tandoor) and never clones this repo or builds locally. The multi-stage `adapter-node` Dockerfile and the CI workflow live in this repo; only the Compose file and `.env` live in `laforcem/homelab`. **Backups accompany the first production ship**, mirroring the existing Immich routine: nightly `pg_dump` + the files volume → rclone to off-site. Because this is medical data going to consumer storage (Dropbox today), the off-site copy must be **client-side encrypted** (e.g. `restic`, or `age`/`gpg` before rclone) — not plaintext — and the first ship includes a **documented restore test**, not just a backup job. Once real data exists, all schema changes go through non-destructive drizzle-kit migrations — no destructive resets.

## 11. Deployment cadence (phasing principle)
**MVP-to-prod-then-iterate.** iterative-development (the autonomous, audited build loop this project uses — see §12) keeps the product working/testable at every iteration boundary; *we* choose when it is useful-and-hardened enough to go live. The **first production ship** = daily logging (entries/observations/tags/severity/vocab autocomplete/"nothing to report") + unified timeline + auth + deployment + backups. Subsequent live updates, roughly ordered: **write path + minimal read-back + one-time historical import** → charts → export → full analysis MCP read surface → in-app subtle features → (optional) transcription → saved dashboards. The import is front-loaded so the timeline holds real data before charts/export arrive (§8.5). The roadmap itself is produced by iterative-development's `scoping-the-simplest-core`, not hand-authored here; this records the front-loading intent it should honor.

## 12. Implementation framework
**iterative-development** is a Claude Code plugin (Prime Radiant) for building a large or ambiguous spec via an *autonomous, audited loop*: it extracts requirements with proof obligations and behavior scenarios, defines a walking skeleton, then runs audited sprints — completion means every externally-observable requirement has *passing behavior evidence*, not just "stories done." It is the deliberate alternative to `writing-plans → subagent-driven-development` for a project this size. This document is its **human spec collateral**, feeding `extracting-requirements` → `scoping-the-simplest-core` (walking skeleton + roadmap) → the audited build loop. The walking skeleton's first task is the E2E test harness. **Scope note:** because the historical import is front-loaded *immediately* after the skeleton (§8.5) and exercises nearly the whole logging model — `is_none`, `present`/severity/qualifier/dose on Tags, the `aggregation`/`reference_period` Field attributes, import-provenance columns, and the validated in-process write path — the walking skeleton (and the iterations right after it) must stand up that full Observation/Tag model, not merely J1's happy-path single-tag entry. `scoping-the-simplest-core` should account for this when ordering the first iterations.

## 13. Journeys (end-to-end flows for requirement extraction)
- **J1 — Log a quick entry:** open app → (already authed via passkey) → add Physical Symptoms tags with severity + a temperature quantity → save → see it on the timeline.
- **J2 — Retroactive/partial entry:** add an entry for an earlier date with only some fields; later add a second entry the same day; timeline day rollup unions them.
- **J3 — Nothing-to-report / review sweep:** mark a clean day; run the end-of-day sweep over watched symptoms.
- **J4 — Upload a lab test:** upload a PDF + test date + panel + provider → open the PDF inline in a new tab → (later) structured results appear and chart alongside symptoms.
- **J5 — Record an appointment:** create provider + appointment with notes and an audio attachment; play it back.
- **J6 — View & filter the timeline:** scroll the unified feed; filter by date range / type / tag.
- **J7 — Chart a symptom and overlay a marker:** view a symptom heatmap; overlay a lab marker over a custom window.
- **J8 — Export for a doctor:** choose range + sections → produce a PDF (optionally with charts) and a CSV/JSON.
- **J9 — Admin schema/vocab management:** add a new field; merge a vocabulary typo.
- **J10 — LLM analysis via MCP:** connect a Claude client to the MCP server; ask for patterns/next steps over honest three-state data.
- **J11 — One-time import:** the agent reads the sheet and writes historical entries through the guarded write tool.

## 14. Open / deferred decisions
- **Plan upgrade for *building* the app** (not running it): to be decided at the transition into iterative-development, once build size is estimable — the subagent-heavy loop is where a temporary upgrade or credits might pay off.
- **Chart library** (LayerChart vs Observable Plot): finalize at the charting phase.
- **Transcription engine** (WhisperX-local vs managed API): finalize at the transcription phase.
- **Per-symptom "not today" toggle**: included as cheap/optional; confirm during logging implementation.
- **MCP server transport & auth** (decide at the MCP phase): transport leans **Streamable HTTP** (remote endpoint) over stdio, given a self-hosted app and a remote analyzing agent; the endpoint's own authentication (Authelia vs. a dedicated agent credential/OAuth) and the exact read/write tool granularity are to be locked when the feature is built. *(Note: that the write path is authenticated **at all** is fixed (§8.6); only the mechanism is deferred.)*
- **Offline write-queue** (later phase): the MVP preserves a single in-progress entry as a local draft (§9); a true multi-entry offline capture queue with sync/conflict handling against SSR + forward-auth is a named future phase, not an MVP promise.
- **Period cycle-event model** (later consideration): the per-day `choice`/`is_none` model captures daily state, but cycle-level analysis (period start/end, cycle day) — relevant given the "estrogen off balance" note — may warrant a dedicated period-event model later. Per-day modeling is sufficient for the MVP.
- **Third-party-PII redaction toggle** (deferred): an opt-in redaction pass for exports / agent views, should data sharing ever widen beyond the household (§8.6).

## 15. Frontend design process (binding working agreement)
**Governing rule:** at every stage the **implementing agent** presents 2–4 distinct options with rationale; the human picks/blends/redirects in plain language; the implementing agent executes all code; it never finalizes a design unilaterally.
- **Stage 1 — Flow & wireframes (low-fi):** screens, navigation, contents — structure before style.
- **Stage 2 — Visual direction:** 2–4 distinct directions as shadcn theme tokens + typography; pick/blend.
- **Stage 3 — High-fidelity in the real stack:** implement the chosen direction as SvelteKit/shadcn components; iterate specifics in-stack (what is tuned is what ships).

**Tooling:** **Claude Design** (included in Pro, research preview) drives Stages 1–2 — its canvas + chat/inline-comment loop *is* the options-first, direct-in-plain-language workflow — then its **Export to Claude Code** handoff bundle feeds Stage 3, where implementation/tuning happens in-stack. Caveats: research preview (rough edges); design-system inheritance is strongest once the codebase/tokens exist (early exploration is generic); the handoff is a strong starting point that Claude Code still adapts into idiomatic shadcn-svelte. The boundary holds as long as the human drives prompts/edits.

## 16. Decision latitude (fixed vs. judgment)

This section tells the implementing agent and the orchestrator **where decisions are settled and where they may use judgment**, so gaps in the spec are filled deliberately rather than silently.

**Fixed — do not deviate without a spec change:**
- The Open-world rule and the three-state (present / absent / unknown) semantics, including presence-wins at day rollup.
- The hybrid data-model *shape* (Field / Vocabulary term / Entry / Observation / Tag) and fixed-schema tables for tests, appointments, providers, and files ([ADR-0002](../../adr/0002-hybrid-field-registry-data-model.md)).
- Auth via Authelia forward-auth with **no in-app login**; identity from trusted headers ([ADR-0001](../../adr/0001-authentication-via-forward-auth-proxy.md)).
- The locked tech stack (§6) and hybrid local-dev model.
- MVP-first deploy cadence (§11) with the historical import front-loaded ahead of charts/export (§8.5).
- The frontend design process and its options-first rule (§15).
- Files are write-once and served inline (§7.5), and their bytes cascade-delete with the owning record; exports are PDF + tidy/wide CSV + JSON (§8.4).
- `occurred_on` is a local civil date in a single configured patient-home timezone, and day-rollup is computed in that timezone; `prior_night` fields (Sleep) attribute to the wake date (§7.1).
- `quantity` unit is declared at the Field level (observations inherit it); the absence validity matrix holds (§7.2).
- Header-trust hardening: the app publishes no host port, Caddy sets `trusted_proxies`, the app fails closed without a valid `Remote-User`, and mutating/admin endpoints re-derive role server-side (§9).
- The MCP write path / any network-reachable MCP transport is never exposed without authentication, from first deployment (§8.6).
- Off-site backups are client-side encrypted, with a restore test at first ship (§10).
- The import writes through the validated path, **fabricates nothing**, is idempotent and reversible (`import_batch` + per-row key; rollback before the next backup), and runs **in-process / loopback** (not the remote transport). Its field-by-field *normalization* is the agent's **heuristic under mandatory human review**, not a set of rigid rules; "can't remember" maps to unknown (§8.5).
- Field **keys are immutable once a Field has observations** (they are the read-view contract, §7.4) — renaming changes the display label only; type changes are blocked while observations exist (§8.1).

**Judgment allowed — implement the simplest thing that satisfies the fixed constraints, and document the choice:**
- Exact table/column names and physical layout; the precise absence-flag representation (so long as the three states stay distinguishable).
- Whether to add a *materialized* read-view later for a measured performance need (plain/dynamic views are the fixed default, §7.4; a materialized one is allowed only with an explicit refresh trigger on registry changes).
- Autocomplete and other UI interaction details; pagination sizes; error/empty-state behavior and copy.
- The seed `choice` options for Period and Bowel Movements (Admin-editable anyway).
- Soft-vs-hard delete and whether to add an audit trail (default hard; §8.1).
- The background-job mechanism for long-running work (DB-backed queue vs. worker container).
- The exact contents of the import severity-adjective lexicon, the dose field's representation (free-text vs. structured), and the heatmap's exact palette — provided the five visual states stay distinct (§8.3).
- How charts/exports reduce multiple same-field `quantity` readings in one entry (e.g. two ear-thermometer readings) to a single plotted point — take the higher, per-site series, etc. — provided the raw readings are preserved (§7.1).

When the spec is silent, prefer the simplest option consistent with the fixed list, implement it, and record the decision in the iteration log rather than blocking.

## 17. References
- Glossary: [`/CONTEXT.md`](../../../CONTEXT.md)
- [ADR-0001 — Authentication via forward-auth proxy](../../adr/0001-authentication-via-forward-auth-proxy.md)
- [ADR-0002 — Hybrid field-registry data model](../../adr/0002-hybrid-field-registry-data-model.md)
