# Hybrid field-registry data model (not fixed columns, not pure EAV)

Symptom data is heterogeneous (numbers, tag-sets with severity, narrative) and the Admin must add new fields at runtime, so neither fixed columns nor pure EAV fits. We use a **hybrid**: a typed **Field** registry + per-field **Vocabulary** + **Entry**/**Observation**/**Tag**, where each Observation stores its value in the typed slot matching its Field's declared type. Homogeneous records (lab tests, appointments) stay **fixed-schema** tables. Analysis reads **curated read-only views in tidy/long form**, never the normalized write tables.

## Considered options
- **Fixed relational columns:** simplest queries, but fails the runtime-extensible-schema requirement and models tag-sets/severity poorly.
- **Pure EAV:** infinitely flexible but stringly-typed — kills type-correct aggregation and integrity.
- **Hybrid registry (chosen):** EAV-like flexibility for heterogeneous, runtime-defined fields; relational typing where it matters (numeric values for charts, FKs); fixed tables for homogeneous records.

## Consequences
- The UI and queries interpret Field *types* generically — that genericity is exactly what powers the extensible-schema + autocomplete features.
- The schema is self-describing (Field/Vocabulary are data), which an LLM/agent can introspect — a strength for analysis and the one-time import.
- "Wide" spreadsheet output requires an app-code pivot over the live registry (SQL can't pivot an unknown column set); analysis otherwise uses tidy/long.
