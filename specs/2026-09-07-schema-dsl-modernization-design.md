# Schema DSL Modernization — Design Spec

**Date:** 2026-09-07
**Companion epic:** engine#978 (field-level DSL coverage gaps)

## Problem

Engine schema coverage is ~85%. The gaps are concentrated in agent descriptors, judgment targets, worker extensions, compounds, GOAP, and HTN decomposition. Beyond coverage, the schema infrastructure has three structural issues:

1. **Missing shared modules** — neocortex has SealedHierarchyModule (now in platform) and ShorthandModule (still neocortex-only) that make schemas less verbose and more intuitive. Engine has neither.
2. **Hand-written records** — ~15 YAML deserialization records are hand-written instead of generated from the schema. Blocks reduced hand-written records from 40 to 5 by generating from the canonical DSL. Engine should target the same ratio.
3. **No drift detection** — nothing prevents someone from hand-writing a new record instead of adding to the schema. No build failure when generated and hand-written diverge.

## Decisions

### D1: Scope — both tracks together
**Choice:** DSL improvements and coverage gaps in one epic.
**Rationale:** SealedHierarchyModule adoption naturally closes some gaps (e.g., JudgmentTarget becomes a oneOf variant automatically). Doing them separately means rework.

### D2: SealedHierarchyModule adoption
**Choice:** Adopt from `casehub-platform-schema-generator` (already available).
**Rationale:** Engine has ~10 sealed types that currently require hand-coded oneOf patterns. Auto-generation from sealed interfaces is more maintainable.

### D3: ShorthandModule extraction
**Choice:** Extract from neocortex to `casehub-platform-schema-generator`.
**Rationale:** Engine already does scalar-or-object polymorphism ad-hoc (e.g., `adaptation: adaptive` vs full object). Shared module makes this systematic and schema-expressed.

### D4: Worker schema — discriminated oneOf
**Choice:** Replace `additionalProperties: true` with discriminated oneOf (agent / a2a / mcp / react / none variants).
**Rationale:** Currently zero validation on worker extension blocks. Discriminated oneOf provides full validation and aligns with SealedHierarchyModule approach.

### D5: Schema as source of truth — generate records
**Choice:** Generate all YAML deserialization records from schema except the most complex (~5). Delete hand-written copies.
**Rationale:** Blocks' experience — reduced 40 hand-written to 5. Same pipeline already exists in engine (CasehubRecordCodegen). The gap is types that bypassed the pipeline.

### D6: Drift detection — platform shared tooling
**Choice:** Build drift detection as a platform Maven plugin/enforcer rule. Extract codegen to platform too.
**Rationale:** All repos (engine, neocortex, blocks) need the same pattern. Platform is the right home.

### D7: Module deduplication
**Choice:** Engine's EnumInliningModule and UnevaluatedPropertiesModule → use platform's copies.
**Rationale:** Exact duplicates. Engine already depends on platform.

## Architecture

### Phase 1 — Platform Foundation (prerequisite)

Three platform additions to `casehub-platform-schema-generator`:

1. **ShorthandModule** — extracted from neocortex. Enables scalar-or-object polymorphism for types with a dominant simple form.
2. **Shared codegen** — generalized from engine's `CasehubRecordCodegen`. Schema + mappings file → generated YAML deserialization records. Each repo provides its own schema and mappings.
3. **Drift detection** — generic Maven plugin or enforcer rule for any codegen-vs-hand-written parity enforcement. Not YAML-record-specific — usable wherever generated code has hand-written counterparts (REST DTOs, test fixtures, schema records, etc.). Config: generated-sources dir, target package, allow-list file, optional naming convention. Scans for classes in target package not in generated output or allow-list. Build failure on drift.

### Phase 2 — Engine Module Adoption

1. **SealedHierarchyModule** — apply to: `CallerConfig` (Human/Llm/A2A/Any), `GoalExpression` (AllOf/AnyOf/Single), `SubCaseMapping` (Expression/Lambda), `AdaptationCause` (StepCompleted/StepFailed), `JudgmentPayload` (BindingPayload/GatePayload), `CallerRef` (PlanItemRef/GateRef/JudgmentRef). Evaluate whether `BindingTargetModule` can be replaced.
2. **ShorthandModule** — apply to: adaptation presets (`adaptive`/`conservative`/`progress` → full object), expression evaluators (string → JQ, object → explicit language).
3. **Dedup modules** — delete engine's `EnumInliningModule` and `UnevaluatedPropertiesModule`, use platform imports.
4. **Worker discriminated oneOf** — new `WorkerTypeModule` replacing `WorkerSchemaModule`. Discriminator based on which extension block is present.

### Phase 3 — Coverage Gaps (schema → generate → delete)

Covered by engine#978 for field-level work. This epic adds:

1. **Judgment target** — add as binding target variant in the schema's binding target oneOf. Not in #978.
2. **Record generation audit** — for each hand-written YAML record, determine if it can be generated. Migrate to schema + mappings. Document justified exceptions in allow-list.

### Phase 4 — Enforcement

1. **Activate drift detection** in engine build with allow-list.
2. **Schema validation tests** — round-trip every test YAML fixture through the generated schema. Fixture that doesn't validate = test failure.
3. **DSL conventions doc** — following neocortex's `yaml-schema-conventions.md` pattern.

## Relationship to engine#978

This epic owns **infrastructure and tooling**. Engine#978 owns **field-level coverage gaps** (GOAP actions, compounds, agent descriptors, etc.). Together they achieve 100% coverage. The two epics share no child issues but have ordering dependencies:

- SealedHierarchyModule adoption (this epic) should land before #978's agent descriptor work (cleaner schema generation for sealed inner types)
- Worker discriminated oneOf (this epic) should land before #978's a2a/mcp/react schema work (changes the schema structure those fields live in)
- Record generation audit (this epic) should follow #978's schema additions (more types to evaluate for generation)

## References

- neocortex `docs/specs/2026-09-01-yaml-schema-conventions.md` — DSL conventions
- neocortex `schema-generator/` — SealedHierarchyModule and ShorthandModule source
- platform `casehub-platform-schema-generator` — shared module home
- engine `schema/src/main/resources/schema/CaseDefinition.yaml` — canonical schema
- engine `schema/src/main/resources/schema/yaml-record-mappings.yaml` — codegen mappings
- engine `codegen/` — CasehubRecordCodegen
- engine#978 — companion epic for field-level coverage
- engine#1027 — companion epic for DSL/annotations audit
