## D1: How to add ensemble support to retrieveForSelection

**Choice:** New overloads returning `CbrRetrievalResult` with nullable `caseType` parameter
**Alternatives:**
- Pass `CbrConfig` directly — carries irrelevant fields (timing, problemDescription, featureExtractor) since callers already have raw features
- Pass full `CaseDefinition` — defeats the purpose of `retrieveForSelection()`, which exists for callers without a definition
**Rationale:** Ensemble analysis needs exactly one thing the current signature lacks: `caseType` for `PlanEnsembleAnalyzer.analyze()`. Adaptation is correctly skipped — raw selection queries, not case-bound retrievals. Nullable allows outcome-only fallback.
**Trade-offs:** Two parallel method families (old returns `List`, new returns `CbrRetrievalResult`). Old overloads retained for backward compat.
**Sources:** CbrRetrievalService.java:204-247, parent spec §retrieveForSelection
**Exploration:** quick
**Status:** captured

## D2: Example structure — three-pathway pattern

**Choice:** Three examples — annotated (`@Case` + `@Cbr`), DSL (Java builder), YAML — matching the existing triple-pathway convention (goap-annotated / goap-dsl / yaml/goap-*.yaml)
**Alternatives:**
- YAML + DSL only — avoids the `@Cbr` annotation work but leaves the annotated path unable to configure CBR, which is a parity gap
- YAML only — insufficient for a "meaty" tutorial; can't show runtime behavior
**Rationale:** The example exposes the annotation gap. Filling gaps is the point — no parity gaps between YAML, DSL, and annotations.
**Trade-offs:** Requires adding `@Cbr` annotation support (new annotation, Jandex scanning, CaseDefinitionRecorder codegen). Self-contained work following established patterns.
**Depends on:** D3 (@Cbr annotation design)
**Sources:** examples/ directory structure, AnnotationValidationStep.java, CaseDefinitionRecorder.java, CaseDescriptor.java
**Exploration:** quick
**Status:** captured

## D3: @Cbr annotation design

**Choice:** `@Cbr` on `@Case` interface with nested `@Feature` and `@Weight` annotations
**Alternatives:**
- Single annotation with string arrays — less type-safe, harder to read: `@Cbr(features = {"amount:.amount", "type:.type"})` requires custom parsing
- Programmatic-only via `@Customize` — works but defeats the purpose of declarative annotations
**Rationale:** Java annotations can't have Map fields. Nested annotations (`@Feature(name="amount", expression=".amount")`) are the established Java pattern for key-value pairs in annotations. Consistent with `@Param` pattern in this codebase.
**Trade-offs:** Slightly more verbose than YAML `features:` block, but type-safe and IDE-completable.
**Sources:** CbrConfig.java fields, Capability.java / Cost.java annotation patterns
**Exploration:** quick
**Status:** captured

## D4: YAML schema parity fix

**Choice:** Add `crossType` (boolean) and `problemDescription` (string/expression) to Cbr definition in CaseDefinition.yaml
**Alternatives:** None — these are bugs, not design choices
**Rationale:** The deserializer already handles both fields. Schema validation doesn't know about them. Gap introduced when the fields were added to CbrConfig.
**Trade-offs:** None.
**Sources:** CaseDefinition.yaml:1154-1202 vs CbrConfig.java:27-40, CbrConfigDeserializer.java:62-64
**Exploration:** quick
**Status:** captured
