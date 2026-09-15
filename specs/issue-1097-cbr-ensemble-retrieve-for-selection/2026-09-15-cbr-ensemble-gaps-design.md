# CBR Ensemble: retrieveForSelection, @Cbr Annotation, Schema Parity, Three-Pathway Example

**Issues:** engine#1097, engine#1098
**Date:** 2026-09-15

## Problem

Three gaps exposed by attempting to build a CBR ensemble consensus tutorial:

1. `retrieveForSelection()` cannot perform ensemble analysis (engine#1097)
2. No `@Cbr` annotation — the annotated pathway can't declare CBR configuration
3. YAML schema missing `crossType` and `problemDescription` fields (deserializer handles them, schema doesn't validate them)

## Scope

Four changes, ordered by dependency:

1. **YAML schema parity** — add missing fields to `Cbr` in `CaseDefinition.yaml`
2. **`@Cbr` annotation** — new annotation + deployment scanning + recorder codegen
3. **`retrieveForSelection()` ensemble** — new overloads returning `CbrRetrievalResult`
4. **Three-pathway example** — annotated + DSL + YAML showing ensemble consensus end-to-end

## 1. YAML Schema Parity

Add to `Cbr` definition in `schema/src/main/resources/schema/CaseDefinition.yaml`:

```yaml
crossType:
  type: boolean
  default: false
problemDescription:
  type: string
  description: "JQ expression evaluated against context to produce problem description for CBR case"
```

`CbrConfigDeserializer` already handles both fields (lines 62-64). This is schema-only — no runtime change.

## 2. @Cbr Annotation

### Annotation types

`annotations/runtime/src/main/java/io/casehub/engine/annotations/`:

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface Cbr {
    Feature[] features();
    Weight[] weights() default {};
    int topK() default 5;
    double minSimilarity() default 0;
    String domain() default "";
    String caseType() default "";
    double vectorWeight() default 0.5;
    String timing() default "per-evaluation";
    String cbrType() default "";
    int temporalDecayHalfLifeDays() default -1;   // -1 = not set
    int minCostSamples() default -1;               // -1 = not set
    boolean crossType() default false;
    String problemDescription() default "";
}

@Retention(RetentionPolicy.RUNTIME)
@Target({})
public @interface Feature {
    String name();
    String expression();
}

@Retention(RetentionPolicy.RUNTIME)
@Target({})
public @interface Weight {
    String name();
    double value();
}
```

`@Cbr` goes on the `@Case` interface alongside `@Case`. `Feature` and `Weight` are nested-only annotations (`@Target({})` — cannot appear standalone). Default values use `-1` sentinel for optional integers (Java annotations don't support `null`).

### CbrDescriptor

`annotations/runtime/src/main/java/io/casehub/engine/annotations/runtime/`:

```java
public record CbrDescriptor(
    Map<String, String> features,
    Map<String, Double> weights,
    int topK,
    double minSimilarity,
    String domain,
    String caseType,
    double vectorWeight,
    String timing,
    String cbrType,
    int temporalDecayHalfLifeDays,
    int minCostSamples,
    boolean crossType,
    String problemDescription) {}
```

### CaseDescriptor change

Add `@Nullable CbrDescriptor cbr` as the 22nd field. Backward-compatible — deployment processors that don't scan `@Cbr` pass `null`.

### AnnotationValidationStep (deployment)

In `validate()`, after scanning `@Case`:
- Check for `@Cbr` on the same type
- Validate: `features` must be non-empty, `topK >= 1`, `minSimilarity` in `[0, 1]`, `vectorWeight` in `[0, 1]`
- Build `CbrDescriptor` from annotation values

### CaseDefinitionRecorder (runtime)

In `createCaseDefinitionInternal()`, after existing builder calls:
```java
if (descriptor.cbr() != null) {
    CbrDescriptor cbr = descriptor.cbr();
    CbrConfig.Builder cbrBuilder = CbrConfig.builder();
    cbr.features().forEach((name, expr) -> cbrBuilder.feature(name, expr));
    cbr.weights().forEach(cbrBuilder::weight);
    cbrBuilder.topK(cbr.topK())
              .minSimilarity(cbr.minSimilarity())
              .vectorWeight(cbr.vectorWeight())
              .crossType(cbr.crossType());
    if (!cbr.domain().isEmpty()) cbrBuilder.domain(cbr.domain());
    if (!cbr.caseType().isEmpty()) cbrBuilder.caseType(cbr.caseType());
    if (!cbr.cbrType().isEmpty()) cbrBuilder.cbrType(cbr.cbrType());
    if (!cbr.problemDescription().isEmpty())
        cbrBuilder.problemDescription(cbr.problemDescription());
    if (!cbr.timing().isEmpty())
        cbrBuilder.timing(CbrRetrievalTiming.valueOf(
            cbr.timing().toUpperCase().replace("-", "_")));
    if (cbr.temporalDecayHalfLifeDays() > 0)
        cbrBuilder.temporalDecayHalfLifeDays(cbr.temporalDecayHalfLifeDays());
    if (cbr.minCostSamples() > 0)
        cbrBuilder.minCostSamples(cbr.minCostSamples());
    builder.cbrConfig(cbrBuilder.build());
}
```

### Tests

- `AnnotationValidationStepTest` — `@Cbr` with empty features produces error, valid `@Cbr` passes
- `CaseDefinitionRecorderTest` — `CbrDescriptor` → `CbrConfig` mapping, sentinel values handled correctly
- Build-time integration test in `annotations/deployment` — `@Case` + `@Cbr` produces correct `CaseDefinition`

## 3. retrieveForSelection() Ensemble Support

### New overloads

Add two new overloads to `CbrRetrievalService` that return `CbrRetrievalResult`:

```java
public CbrRetrievalResult retrieveForSelectionWithEnsemble(
    String tenancyId,
    String domain,
    Map<String, FeatureValue> features,
    int topK,
    double minSimilarity,
    Map<String, Double> weights,
    @Nullable String caseType) {
  return retrieveForSelectionWithEnsemble(
      tenancyId, domain, features, topK, minSimilarity, weights,
      caseType, ResolvedCase.class);
}

public <C extends CbrCase> CbrRetrievalResult retrieveForSelectionWithEnsemble(
    String tenancyId,
    String domain,
    Map<String, FeatureValue> features,
    int topK,
    double minSimilarity,
    Map<String, Double> weights,
    @Nullable String caseType,
    Class<C> caseClass) {
  // 1. Build query, retrieve scoredCases (same as existing)
  // 2. Map to experiences via mapResults() (same as existing)
  // 3. If caseType != null && plan-type results >= 2:
  //    - Build adaptedPlans from raw plan traces (no PlanAdapter — RETAINED steps)
  //    - Call ensembleAnalyzer.analyze(caseType, scoredCases, adaptedPlans, features)
  //    - Post-analysis guard: inputPlanCount < 2 → null
  //    - Map EnsemblePlan → EnsembleConsensus with STEP_LEVEL scope
  // 4. If caseType == null || plan-type < 2 but total >= 2:
  //    - buildOutcomeOnlyConsensus() (same as retrieveInternal)
  // 5. Return CbrRetrievalResult(experiences, ensemble)
}
```

### No adaptation

`retrieveForSelection()` operates without `CaseDefinition`, so there is no `PlanAdapter` to call. Raw plan traces are wrapped in fallback `AdaptedPlan` with all steps `RETAINED` — same pattern as the partial-adaptation-failure fallback in `retrieveInternal()`. This preserves the sizing invariant for `PlanEnsembleAnalyzer.analyze()`.

### Timeout

Uses the same `ensembleTimeoutMs` config (`casehub.engine.cbr.ensemble-timeout-ms`, default 5000) as `retrieveInternal()`. Same timeout pattern (virtual thread + `Future.get()`).

### Error handling

Same graceful degradation as `retrieveInternal()` — any exception returns `CbrRetrievalResult(experiences, null)`.

### Existing overloads unchanged

The two existing `retrieveForSelection()` overloads returning `List<RetrievedExperience>` are unchanged. They remain for callers that don't need ensemble analysis.

### Tests

- `retrieveForSelectionWithEnsemble_invokes_ensemble_for_plan_type` — 2+ plan results with caseType → `STEP_LEVEL` ensemble
- `retrieveForSelectionWithEnsemble_null_caseType_returns_outcome_only` — null caseType → `OUTCOME_ONLY`
- `retrieveForSelectionWithEnsemble_single_result_returns_null_ensemble` — < 2 results → null ensemble
- `retrieveForSelectionWithEnsemble_uses_raw_traces_not_adapted` — verify no PlanAdapter call, all steps RETAINED
- `retrieveForSelectionWithEnsemble_timeout_degrades_gracefully` — analyzer timeout → null ensemble, experiences preserved

## 4. Three-Pathway Example

### 4a. YAML — `examples/yaml/cbr-ensemble-consensus.yaml`

Demonstrates CBR ensemble configuration in pure YAML. Shows:
- `cbr:` block with features, domain, caseType, crossType
- Capabilities with input/output projections
- Bindings with context-change triggers
- Agent worker reading `.cbrEnsemble.stepAnalysis` via JQ
- Comments explaining how ensemble consensus works

Example structure: incident triage scenario where multiple past incident responses are retrieved. Agent reads ensemble consensus to identify CONTESTED steps and adjusts its triage approach.

### 4b. Annotated — `examples/cbr-ensemble-annotated/`

Single `@Case` interface with `@Cbr` annotation. Same triage scenario.

```java
@Case(namespace = "example", name = "EnsembleTriage", ...)
@Cbr(
    features = {
        @Feature(name = "severity", expression = ".severity"),
        @Feature(name = "category", expression = ".category")
    },
    domain = "incident-triage",
    caseType = "EnsembleTriage",
    topK = 5
)
public interface EnsembleTriageCase {
    @Worker(capability = "triage", ...)
    @Bind(contextChange = ".incident != null")
    @SystemPrompt("... query .cbrEnsemble.stepAnalysis for CONTESTED steps ...")
    TriageResult triage(Incident incident);
    // ...
}
```

Module structure: `pom.xml` (deploy skip, depends on `casehub-engine-annotations` + deployment), single interface file. `@QuarkusTest` that:
1. Pre-seeds 5 divergent CBR cases (3 agree on step A, 2 disagree → CONTESTED)
2. Starts a case with matching features
3. Asserts `cbrEnsemble` in context has expected step classifications
4. Verifies JQ query `.cbrEnsemble.stepAnalysis[] | select(.agreement == "CONTESTED")` returns the divergent step

### 4c. DSL — `examples/cbr-ensemble-dsl/`

Java `CaseHub` subclass with `CbrConfig.builder()`. Same triage scenario. Shows the programmatic path.

### Shared test infrastructure

All three examples need pre-seeded CBR data. The `@QuarkusTest` in both annotated and DSL modules uses `InMemoryCbrCaseMemoryStore` (from `casehub-neocortex-memory-cbr-inmem`). Seeds 5 `ResolvedCase` instances with overlapping but divergent plan traces.

A shared test utility class in a common test-jar is overkill for 2 modules — each module seeds its own data inline. The seeding pattern is simple enough to duplicate:

```java
// Seed 5 cases: 3 agree on [triage→investigate→resolve], 2 diverge on [triage→escalate→resolve]
store.store(buildCase("case-1", List.of(step("triage", SUCCESS), step("investigate", SUCCESS), step("resolve", SUCCESS))));
store.store(buildCase("case-2", List.of(step("triage", SUCCESS), step("investigate", SUCCESS), step("resolve", SUCCESS))));
store.store(buildCase("case-3", List.of(step("triage", SUCCESS), step("investigate", SUCCESS), step("resolve", SUCCESS))));
store.store(buildCase("case-4", List.of(step("triage", SUCCESS), step("escalate", SUCCESS), step("resolve", SUCCESS))));
store.store(buildCase("case-5", List.of(step("triage", SUCCESS), step("escalate", DECLINED), step("resolve", SUCCESS))));
```

Expected ensemble for step "investigate": CONSENSUS (3/5). Step "escalate": CONTESTED (2/5). Step "triage" and "resolve": UNANIMOUS (5/5).

## CLAUDE.md Updates

Add to `## casehub-engine-annotations Module` after the existing `@Cost` paragraph:

```
`@Cbr` — case-level CBR retrieval configuration. Nested `@Feature(name, expression)` for JQ feature extractors, `@Weight(name, value)` for feature weights. All `CbrConfig` fields supported: `topK`, `minSimilarity`, `domain`, `caseType`, `vectorWeight`, `timing`, `cbrType`, `temporalDecayHalfLifeDays`, `minCostSamples`, `crossType`, `problemDescription`. Sentinel `-1` for optional integers (Java annotations don't support null). Processor: `AnnotationValidationStep` validates features non-empty; `CaseDefinitionRecorder` maps to `CbrConfig.builder()`. Refs engine#1098.
```

Add to `## CBR Retrieval Bridge` section after the existing `CbrRetrievalResult` paragraph:

```
`retrieveForSelectionWithEnsemble()` — overloads on `CbrRetrievalService` returning `CbrRetrievalResult` for selection queries. Takes `@Nullable String caseType` for ensemble analysis. No adaptation (no `CaseDefinition` available) — raw plan traces wrapped as RETAINED `AdaptedPlan`. Same timeout and error handling as `retrieveInternal()`. Refs engine#1097.
```

## References

- `runtime-core/.../CbrRetrievalService.java:204` — retrieveForSelection overloads
- `api/.../cbr/CbrConfig.java` — CbrConfig record (13 fields)
- `api/.../converter/deser/CbrConfigDeserializer.java` — YAML deserializer (already handles crossType, problemDescription)
- `schema/.../CaseDefinition.yaml:1154` — Cbr JSON schema definition (missing 2 fields)
- `annotations/runtime/.../CaseDescriptor.java` — deployment→runtime data carrier
- `annotations/deployment/.../AnnotationValidationStep.java` — Jandex scanning
- `annotations/runtime/.../CaseDefinitionRecorder.java` — Gizmo codegen
- `examples/incident-response-annotated/` — annotated example pattern reference
- `examples/yaml/goap-contract-review.yaml` — YAML example pattern reference
- `docs/specs/issue-1051-cbr-ensemble-consensus/` — parent design spec
