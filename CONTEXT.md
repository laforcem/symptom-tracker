# Symptom Tracker

A full-stack web app (PWA) for one chronically-ill patient and her admin to log daily
symptoms and health factors with discipline, then aggregate, analyze, and export that
data to aid diagnosis. Replaces a free-text Google Sheet.

## Language

### People

**Patient**:
The single person whose health data the system tracks. Logs entries, uploads tests,
views the timeline/charts, runs exports. There is exactly one patient.

**Admin**:
The technical caretaker. Can do everything the patient can, plus manage the schema
(fields), clean up vocabulary, and manage data. Identity comes from an Authelia group.
_Avoid_: owner, superuser.

**Provider**:
A doctor, specialist, or clinic the patient sees. Subject of appointment records.
Providers do not have accounts; they receive exports.
_Avoid_: doctor (too narrow), care team.

### Logging core

**Entry**:
A single logging event at a date and optional time. Multiple entries may exist per day.
Analogous to one row in the old spreadsheet.
_Avoid_: log, record, row, day (a day may contain several entries).

**Field**:
An admin-defined category of thing that can be logged (e.g. Physical Symptoms,
Temperature, Sleep). Each field declares a type that governs how its values are stored:
`tag_set`, `quantity`, `choice`, `text`, or `boolean`. The set of fields is data, not
hardcoded — admins add fields at runtime.
_Avoid_: column, attribute, category (informal use only).

**Observation**:
The value of one field within one entry — "this entry's value for this field."
Analogous to one filled-in cell in the old spreadsheet. A field with no observation in
an entry means "not logged" (distinct from logging "none").
_Avoid_: value, cell, datum.

**Tag**:
One selected item on a `tag_set` observation (e.g. "bloating" on a Physical Symptoms
observation). Points at a Vocabulary term and may carry optional severity and a free-text
qualifier (e.g. "joint pain (knees)").
_Avoid_: symptom (a tag may be a vitamin, food, etc.), item, value.

**Vocabulary term**:
A reusable, autocomplete-able value scoped to a single field (e.g. "bloating" under
Physical Symptoms). Grows as the patient types new values; admins merge or delete typos.
_Avoid_: option, choice (reserved for the `choice` field type), label.

**Watched symptom**:
A vocabulary term the patient is actively monitoring (e.g. joint pain, fever, brain fog),
as opposed to the long tail of incidental terms. Watched symptoms are the ones the review
sweep asks about, bounding the effort of recording absence.
_Avoid_: tracked symptom, key symptom.

**Review sweep**:
An optional fast end-of-day pass where the patient marks each watched symptom as
present / absent / skip, converting "unknown" days into affirmatively present-or-absent
data without fabricating absence.
_Avoid_: daily check-in, end-of-day review (informal use only).

### Data semantics

**Open-world rule**:
The governing interpretation of symptom data: a logged tag asserts presence; a missing
tag asserts nothing (unknown), never absence. Only an affirmative act ("nothing to report"
or a per-symptom "not today" / review-sweep mark) records absence. At day-level rollup,
presence wins over any same-day negative. The analyzing agent and exports must honor this
and treat gaps as unknown, never absent.

**Day-level rollup**:
The aggregation of a day's entries and observations into a single per-day view for the
timeline, charts, and analysis. When a symptom has conflicting same-day signals, presence
wins over a negative. Hour-level detail is captured but not used analytically.
_Avoid_: daily aggregate, daily summary.

### AI roles

**Analyzing agent**:
The external LLM/agent that consumes the data through the MCP server to produce diagnostic
hypotheses and next-step suggestions (and to perform the one-time import). Provider-neutral —
any MCP-capable agent. Must honor the Open-world rule.
_Avoid_: the LLM, the AI, the model (all ambiguous in this project).

**Implementing agent**:
The AI that builds the software — Claude Code driving the iterative-development workflow,
under human direction. Distinct from the analyzing agent (consumes data) and the feature
model (powers in-app features); it writes code, it does not touch patient data at runtime.
_Avoid_: the AI, the assistant.

**Feature model**:
The LLM backend powering small in-app data-extraction/summarization features, reached
through a provider-agnostic OpenAI-compatible interface (OpenRouter, a local Ollama on the
planned GPU box, the Claude API, etc.). Distinct from the analyzing agent (heavy diagnosis
via MCP) and the implementing agent (builds the app).
_Avoid_: the LLM, the AI.
