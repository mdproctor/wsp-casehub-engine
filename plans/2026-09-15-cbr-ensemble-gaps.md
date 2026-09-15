# CBR Ensemble Gaps Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #1097 — CBR: add ensemble analysis support to retrieveForSelection overloads
**Issue group:** #1097, #1098

**Goal:** Fill CBR parity gaps across YAML schema, annotations, and retrieveForSelection; build three-pathway example demonstrating ensemble consensus.

**Architecture:** Four independent changes ordered by dependency. Schema parity fix is trivial. @Cbr annotation follows the established Jandex→CaseDescriptor→CaseDefinitionRecorder pipeline. retrieveForSelection ensemble reuses existing `computeEnsemble` infrastructure with raw (unadapted) plan traces. Three-pathway example seeds divergent CBR cases and asserts ensemble classifications.

**Tech Stack:** Java 21, Quarkus 3.32.2, Jackson, Jandex, casehub-neocortex-memory-api

## Global Constraints

- All new types in `api/` or `annotations/` — never in `runtime/` or `runtime-core/`
- Example modules use `<maven.deploy.skip>true</maven.deploy.skip>`
- TDD: write failing test → verify failure → implement → verify pass → commit
- Use `ide_insert_member` / `ide_replace_member` for Java edits, `ide_refactor_rename` for renames
- Run `mvn install -DskipTests -q` before any module-specific test run
- Always include `TESTCONTAINERS_RYUK_DISABLED=true` for test runs

---

## Batch 1: Schema Parity + @Cbr Annotation Foundation

### Task 1: YAML Schema Parity — add crossType and problemDescription to Cbr

**Files:**
- Modify: `schema/src/main/resources/schema/CaseDefinition.yaml:1154-1202`

**Interfaces:**
- Consumes: nothing
- Produces: two new schema fields on `Cbr` definition — no runtime impact, already parsed by `CbrConfigDeserializer`

- [ ] **Step 1: Add `crossType` and `problemDescription` to the Cbr schema definition**

In `schema/src/main/resources/schema/CaseDefinition.yaml`, inside the `Cbr` definition properties block (after `minCostSamples`), add:

```yaml
      crossType:
        type: boolean
        default: false
      problemDescription:
        type: string
        description: "JQ expression evaluated against context to produce problem description for CBR case"
```

- [ ] **Step 2: Verify schema regeneration compiles**

Run: `TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn compile -pl schema -q`
Expected: BUILD SUCCESS (generated YAML records include the new fields)

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add schema/src/main/resources/schema/CaseDefinition.yaml
git commit -m "fix: add crossType and problemDescription to Cbr YAML schema

The CbrConfigDeserializer already handles both fields but they were
missing from the JSON Schema — YAML validation didn't know about them.

Refs #1098"
```

### Task 2: @Cbr annotation — annotation types + CbrDescriptor + CaseDescriptor change

**Files:**
- Create: `annotations/runtime/src/main/java/io/casehub/engine/annotations/Cbr.java`
- Create: `annotations/runtime/src/main/java/io/casehub/engine/annotations/Feature.java`
- Create: `annotations/runtime/src/main/java/io/casehub/engine/annotations/Weight.java`
- Create: `annotations/runtime/src/main/java/io/casehub/engine/annotations/runtime/CbrDescriptor.java`
- Modify: `annotations/runtime/src/main/java/io/casehub/engine/annotations/runtime/CaseDescriptor.java` — add 22nd field

**Interfaces:**
- Consumes: nothing
- Produces: `@Cbr`, `@Feature`, `@Weight` annotations; `CbrDescriptor` record; `CaseDescriptor` with nullable `CbrDescriptor cbr` field

- [ ] **Step 1: Create @Feature annotation**

Create `annotations/runtime/src/main/java/io/casehub/engine/annotations/Feature.java`:

```java
package io.casehub.engine.annotations;

import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Retention(RetentionPolicy.RUNTIME)
@Target({})
public @interface Feature {
  String name();
  String expression();
}
```

- [ ] **Step 2: Create @Weight annotation**

Create `annotations/runtime/src/main/java/io/casehub/engine/annotations/Weight.java`:

```java
package io.casehub.engine.annotations;

import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Retention(RetentionPolicy.RUNTIME)
@Target({})
public @interface Weight {
  String name();
  double value();
}
```

- [ ] **Step 3: Create @Cbr annotation**

Create `annotations/runtime/src/main/java/io/casehub/engine/annotations/Cbr.java`:

```java
package io.casehub.engine.annotations;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface Cbr {
  Feature[] features();
  Weight[] weights() default {};
  int topK() default 5;
  double minSimilarity() default 0;
  String domain() default "";
  String caseType() default "";
  double vectorWeight() default 0;
  String timing() default "per-evaluation";
  String cbrType() default "";
  int temporalDecayHalfLifeDays() default -1;
  int minCostSamples() default -1;
  boolean crossType() default false;
  String problemDescription() default "";
}
```

- [ ] **Step 4: Create CbrDescriptor record**

Create `annotations/runtime/src/main/java/io/casehub/engine/annotations/runtime/CbrDescriptor.java`:

```java
package io.casehub.engine.annotations.runtime;

import java.util.Map;

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

- [ ] **Step 5: Add CbrDescriptor to CaseDescriptor**

Modify `annotations/runtime/src/main/java/io/casehub/engine/annotations/runtime/CaseDescriptor.java` — add `CbrDescriptor cbr` as the 22nd (last) field in the record:

```java
public record CaseDescriptor(
    String namespace,
    String name,
    String version,
    String title,
    String summary,
    String planningStrategy,
    String implClassName,
    String interfaceName,
    List<WorkerDescriptor> workers,
    List<BindingDescriptor> bindings,
    List<GoalDescriptor> goals,
    List<MilestoneDescriptor> milestones,
    List<GoapActionDescriptor> goapActions,
    Map<String, List<String>> goalToEffectKeys,
    List<CompletionDescriptor> completions,
    List<CustomizerDescriptor> customizers,
    List<String> standaloneCapabilities,
    List<CompoundDescriptor> compounds,
    List<SubCaseDescriptor> subCases,
    List<JudgmentDescriptor> judgments,
    CbrDescriptor cbr) {}
```

- [ ] **Step 6: Fix all CaseDescriptor construction sites**

Use `ide_find_references` on `CaseDescriptor` constructor to find all call sites. Add `null` as the 22nd argument to each existing construction. The key site is in `EngineAnnotationsProcessor.buildDescriptor()` at line 396 — add `null` as the last argument (will be replaced with the real value in Task 3).

- [ ] **Step 7: Verify compilation**

Run: `TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn compile -pl annotations/runtime -q`
Expected: BUILD SUCCESS

- [ ] **Step 8: Commit**

```bash
git add annotations/runtime/src/main/java/io/casehub/engine/annotations/Cbr.java annotations/runtime/src/main/java/io/casehub/engine/annotations/Feature.java annotations/runtime/src/main/java/io/casehub/engine/annotations/Weight.java annotations/runtime/src/main/java/io/casehub/engine/annotations/runtime/CbrDescriptor.java annotations/runtime/src/main/java/io/casehub/engine/annotations/runtime/CaseDescriptor.java
git commit -m "feat: add @Cbr, @Feature, @Weight annotations and CbrDescriptor

Annotation parity with YAML cbr: block and Java CbrConfig.builder().
CaseDescriptor gains nullable CbrDescriptor as 22nd field.

Refs #1098"
```

### Task 3: @Cbr annotation — deployment scanning + recorder codegen + tests

**Files:**
- Modify: `annotations/deployment/src/main/java/io/casehub/engine/annotations/deployment/EngineAnnotationsProcessor.java:154-417` — scan `@Cbr`, build `CbrDescriptor`
- Modify: `annotations/deployment/src/main/java/io/casehub/engine/annotations/deployment/AnnotationValidationStep.java:34-99` — validate `@Cbr`
- Modify: `annotations/runtime/src/main/java/io/casehub/engine/annotations/runtime/CaseDefinitionRecorder.java:56-410` — map `CbrDescriptor` → `CbrConfig`
- Test: `annotations/deployment/src/test/java/...` — build-time integration test

**Interfaces:**
- Consumes: `@Cbr` annotation, `CbrDescriptor`, `CaseDescriptor.cbr()` from Task 2
- Produces: `@Case` + `@Cbr` → `CaseDefinition` with populated `CbrConfig` at build time

- [ ] **Step 1: Write failing test — @Cbr produces CbrConfig on CaseDefinition**

Create a test `@Case` + `@Cbr` interface in the deployment test sources. Assert the produced `CaseDefinition` has a non-null `CbrConfig` with the correct fields.

Find the existing test pattern in `annotations/deployment/src/test/` and follow it. The test should:
- Define a `@Case @Cbr(features = {@Feature(name = "amount", expression = ".amount")}, domain = "test", caseType = "TestCase")` interface
- Build the descriptor via `EngineAnnotationsProcessor.buildDescriptor()`
- Assert `descriptor.cbr() != null`
- Assert `descriptor.cbr().features()` contains `"amount" → ".amount"`
- Assert `descriptor.cbr().domain()` equals `"test"`

- [ ] **Step 2: Run test to verify it fails**

Run: `TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn test -pl annotations/deployment -Dtest=<TestClass>#<testMethod> -q`
Expected: FAIL — `@Cbr` not scanned yet

- [ ] **Step 3: Add @Cbr scanning to EngineAnnotationsProcessor.buildDescriptor()**

In `EngineAnnotationsProcessor.java`:

Add DotName constant:
```java
private static final DotName CBR = DotName.createSimple("io.casehub.engine.annotations.Cbr");
```

In `buildDescriptor()`, after the existing processing loops (before the `return new CaseDescriptor(...)` at line 396), add `@Cbr` scanning:

```java
CbrDescriptor cbrDescriptor = null;
AnnotationInstance cbrAnn = caseClass.annotation(CBR);
if (cbrAnn != null) {
    Map<String, String> features = new LinkedHashMap<>();
    for (AnnotationInstance feat : cbrAnn.value("features").asNestedArray()) {
        features.put(feat.value("name").asString(), feat.value("expression").asString());
    }
    Map<String, Double> weights = new LinkedHashMap<>();
    AnnotationValue weightsValue = cbrAnn.value("weights");
    if (weightsValue != null) {
        for (AnnotationInstance w : weightsValue.asNestedArray()) {
            weights.put(w.value("name").asString(), w.value("value").asDouble());
        }
    }
    cbrDescriptor = new CbrDescriptor(
        features,
        weights,
        (int) intValueOrDefault(cbrAnn, index, "topK", 5),
        doubleValueOrDefault(cbrAnn, index, "minSimilarity", 0),
        stringValueOrDefault(cbrAnn, index, "domain", ""),
        stringValueOrDefault(cbrAnn, index, "caseType", ""),
        doubleValueOrDefault(cbrAnn, index, "vectorWeight", 0),
        stringValueOrDefault(cbrAnn, index, "timing", "per-evaluation"),
        stringValueOrDefault(cbrAnn, index, "cbrType", ""),
        (int) intValueOrDefault(cbrAnn, index, "temporalDecayHalfLifeDays", -1),
        (int) intValueOrDefault(cbrAnn, index, "minCostSamples", -1),
        booleanValueOrDefault(cbrAnn, index, "crossType", false),
        stringValueOrDefault(cbrAnn, index, "problemDescription", ""));
}
```

Add helper if needed (check existing helpers — `stringValueOrDefault` and `booleanValueOrDefault` exist; may need `intValueOrDefault` and `doubleValueOrDefault`):

```java
private static int intValueOrDefault(AnnotationInstance ann, IndexView index, String name, int defaultValue) {
    AnnotationValue val = ann.value(name);
    return val != null ? val.asInt() : defaultValue;
}
private static double doubleValueOrDefault(AnnotationInstance ann, IndexView index, String name, double defaultValue) {
    AnnotationValue val = ann.value(name);
    return val != null ? val.asDouble() : defaultValue;
}
```

Update the `return new CaseDescriptor(...)` to pass `cbrDescriptor` as the last argument (replacing the `null` from Task 2 Step 6).

- [ ] **Step 4: Add @Cbr validation to AnnotationValidationStep**

Add DotName:
```java
private static final DotName CBR = DotName.createSimple("io.casehub.engine.annotations.Cbr");
```

In `validate()`, inside the `for (AnnotationInstance caseAnn : index.getAnnotations(CASE))` loop, after method iteration:

```java
AnnotationInstance cbrAnn = caseClass.annotation(CBR);
if (cbrAnn != null) {
    AnnotationValue featuresValue = cbrAnn.value("features");
    if (featuresValue == null || featuresValue.asNestedArray().length == 0) {
        errors.add(caseClass.name() + ": @Cbr requires at least one @Feature");
    }
}
```

- [ ] **Step 5: Add CbrDescriptor → CbrConfig mapping in CaseDefinitionRecorder**

In `createCaseDefinitionInternal()`, before `return builder.build()` (line 409):

```java
if (descriptor.cbr() != null) {
    CbrDescriptor cbr = descriptor.cbr();
    io.casehub.api.model.cbr.CbrConfig.Builder cbrBuilder =
        io.casehub.api.model.cbr.CbrConfig.builder();
    cbr.features().forEach(cbrBuilder::feature);
    cbr.weights().forEach(cbrBuilder::weight);
    cbrBuilder.topK(cbr.topK()).minSimilarity(cbr.minSimilarity())
        .vectorWeight(cbr.vectorWeight()).crossType(cbr.crossType());
    if (!cbr.domain().isEmpty()) cbrBuilder.domain(cbr.domain());
    if (!cbr.caseType().isEmpty()) cbrBuilder.caseType(cbr.caseType());
    if (!cbr.cbrType().isEmpty()) cbrBuilder.cbrType(cbr.cbrType());
    if (!cbr.problemDescription().isEmpty())
        cbrBuilder.problemDescription(cbr.problemDescription());
    if (!cbr.timing().isEmpty())
        cbrBuilder.timing(io.casehub.api.model.cbr.CbrConfig.CbrRetrievalTiming.valueOf(
            cbr.timing().toUpperCase().replace("-", "_")));
    if (cbr.temporalDecayHalfLifeDays() > 0)
        cbrBuilder.temporalDecayHalfLifeDays(cbr.temporalDecayHalfLifeDays());
    if (cbr.minCostSamples() > 0)
        cbrBuilder.minCostSamples(cbr.minCostSamples());
    builder.cbrConfig(cbrBuilder.build());
}
```

- [ ] **Step 6: Run tests to verify pass**

Run: `TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn test -pl annotations/deployment -q`
Expected: all tests PASS

- [ ] **Step 7: Commit**

```bash
git add annotations/
git commit -m "feat: wire @Cbr scanning in deployment processor and recorder

EngineAnnotationsProcessor scans @Cbr on @Case interfaces, builds
CbrDescriptor. AnnotationValidationStep validates features non-empty.
CaseDefinitionRecorder maps CbrDescriptor to CbrConfig.builder().

Refs #1098"
```

---

## Batch 2: retrieveForSelectionWithEnsemble

### Task 4: New overloads on CbrRetrievalService returning CbrRetrievalResult

**Files:**
- Modify: `runtime-core/src/main/java/io/casehub/engine/internal/routing/CbrRetrievalService.java:204-247`
- Test: `runtime/src/test/java/io/casehub/engine/internal/routing/CbrRetrievalServiceTest.java`

**Interfaces:**
- Consumes: existing `mapResults()`, `invokeEnsembleAnalyzer()`, `buildOutcomeOnlyConsensus()`, `ensembleAnalyzer` field
- Produces: `retrieveForSelectionWithEnsemble(tenancyId, domain, features, topK, minSimilarity, weights, caseType)` → `CbrRetrievalResult`; generic overload with `Class<C> caseClass`

- [ ] **Step 1: Write failing test — ensemble invoked for plan-type with caseType**

Add to `CbrRetrievalServiceTest.java`:

```java
@Test
void retrieveForSelectionWithEnsemble_invokes_ensemble_for_plan_type() {
    // Seed 2 ResolvedCase entries in cbrStore
    cbrStore.setResult(List.of(
        buildScoredResolvedCase("case-1", 0.9, List.of(step("triage", "SUCCESS"), step("investigate", "SUCCESS"))),
        buildScoredResolvedCase("case-2", 0.8, List.of(step("triage", "SUCCESS"), step("escalate", "SUCCESS")))
    ));

    CbrRetrievalResult result = service.retrieveForSelectionWithEnsemble(
        "tenant-1", "test-domain",
        Map.of("severity", FeatureValue.of("HIGH")),
        5, 0.0, Map.of(), "TestCase");

    assertNotNull(result.ensemble());
    // NoOpPlanEnsembleAnalyzer returns inputPlanCount=1 → null,
    // so this test needs a recording analyzer — see Step 3
}
```

- [ ] **Step 2: Replace NoOpPlanEnsembleAnalyzer with a recording one in setUp**

Create a `RecordingPlanEnsembleAnalyzer` inner class (or update setUp) that returns an `EnsemblePlan` with `inputPlanCount >= 2` and step analysis. Update `setUp()` to use it.

- [ ] **Step 3: Implement retrieveForSelectionWithEnsemble**

Add two new methods to `CbrRetrievalService` after the existing `retrieveForSelection` methods (after line 247):

```java
public CbrRetrievalResult retrieveForSelectionWithEnsemble(
    String tenancyId, String domain,
    Map<String, FeatureValue> features, int topK,
    double minSimilarity, Map<String, Double> weights,
    @Nullable String caseType) {
  return retrieveForSelectionWithEnsemble(
      tenancyId, domain, features, topK, minSimilarity, weights,
      caseType, ResolvedCase.class);
}

@SuppressWarnings("unchecked")
public <C extends CbrCase> CbrRetrievalResult retrieveForSelectionWithEnsemble(
    String tenancyId, String domain,
    Map<String, FeatureValue> features, int topK,
    double minSimilarity, Map<String, Double> weights,
    @Nullable String caseType, Class<C> caseClass) {
  try {
    if (features.isEmpty()) {
      return CbrRetrievalResult.empty();
    }
    CbrQuery query = CbrQuery.crossType(tenancyId, new MemoryDomain(domain),
            io.casehub.platform.api.path.Path.root(), features, topK)
        .withMinSimilarity(minSimilarity).withWeights(weights);

    List<ScoredCbrCase<C>> scoredCases = cbrStore.retrieveSimilar(query, caseClass);
    List<RetrievedExperience> experiences = List.copyOf(mapResults(scoredCases, features));

    if (experiences.size() < 2) {
      return new CbrRetrievalResult(experiences, null);
    }

    List<ScoredCbrCase<ResolvedCase>> planCases = new ArrayList<>();
    List<AdaptedPlan> rawAdaptedPlans = new ArrayList<>();
    for (ScoredCbrCase<C> sc : scoredCases) {
      if (sc.cbrCase() instanceof ResolvedCase rc) {
        planCases.add((ScoredCbrCase<ResolvedCase>) (ScoredCbrCase<?>) sc);
        List<AdaptedStep> retainedSteps = rc.plan().stream()
            .map(step -> new AdaptedStep(step.bindingName(), step.capabilityName(),
                step.workerName(), step.stepOutcome(), step.priority(),
                step.parameters(), AdaptationAction.RETAINED, null))
            .toList();
        rawAdaptedPlans.add(new AdaptedPlan(retainedSteps));
      }
    }

    EnsembleConsensus ensemble;
    if (caseType != null && planCases.size() >= 2) {
      ensemble = invokeEnsembleAnalyzer(caseType, planCases, rawAdaptedPlans, features);
    } else {
      ensemble = buildOutcomeOnlyConsensus(experiences, scoredCases);
    }

    return new CbrRetrievalResult(experiences, ensemble);
  } catch (Exception failure) {
    LOG.warnf(failure,
        "CBR selection retrieval with ensemble failed for domain '%s'"
            + " — proceeding without experiences", domain);
    return CbrRetrievalResult.empty();
  }
}
```

Note: `AdaptedStep` and `AdaptedPlan` are from `io.casehub.neocortex.memory.cbr` — verify exact constructor parameters via `ide_file_structure` during implementation.

- [ ] **Step 4: Run tests to verify pass**

Run: `TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn test -pl runtime -Dtest=CbrRetrievalServiceTest -q`
Expected: PASS

- [ ] **Step 5: Add additional test cases**

Add tests for:
- `null_caseType_returns_outcome_only` — pass `null` caseType → ensemble has `OUTCOME_ONLY` scope
- `single_result_returns_null_ensemble` — 1 result → ensemble is null
- `timeout_degrades_gracefully` — use a slow mock analyzer → ensemble is null, experiences present

- [ ] **Step 6: Run full test suite to verify**

Run: `TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn test -pl runtime -Dtest=CbrRetrievalServiceTest -q`
Expected: all PASS

- [ ] **Step 7: Commit**

```bash
git add runtime-core/ runtime/
git commit -m "feat: add retrieveForSelectionWithEnsemble overloads

New overloads on CbrRetrievalService returning CbrRetrievalResult for
selection queries. Takes nullable caseType for ensemble analysis. Raw
plan traces used (no adaptation) — RETAINED AdaptedPlan. Same timeout
and error handling as retrieveInternal().

Closes #1097"
```

---

## Batch 3: Three-Pathway Example

### Task 5: YAML example — cbr-ensemble-consensus.yaml

**Files:**
- Create: `examples/yaml/cbr-ensemble-consensus.yaml`

**Interfaces:**
- Consumes: YAML `cbr:` schema (including `crossType`, `problemDescription` from Task 1)
- Produces: standalone YAML case definition demonstrating CBR ensemble configuration

- [ ] **Step 1: Write the YAML example**

Create `examples/yaml/cbr-ensemble-consensus.yaml`:

```yaml
##
## Example: CBR Ensemble Consensus
##
## Scenario: An incident triage system retrieves past incident responses
## via CBR. The PlanEnsembleAnalyzer classifies step-level agreement
## across retrieved plans. The triage agent reads .cbrEnsemble from the
## case context to identify CONTESTED steps and adjust its approach.
##
## Pathway: YAML (pure — no Java required)
## See also: examples/cbr-ensemble-annotated/ (annotations)
##           examples/cbr-ensemble-dsl/ (DSL)
##
## Key features demonstrated:
##   - cbr: block with feature extraction, domain, caseType
##   - crossType and problemDescription (schema parity)
##   - Agent worker reading .cbrEnsemble.stepAnalysis from context
##   - JQ queries for CONTESTED/UNANIMOUS step classification
##

dsl: "1.0.0"
namespace: example
name: ensemble-triage
version: "1.0.0"
title: "Ensemble Triage"
summary: "Incident triage informed by CBR ensemble consensus — identifies agreed and contested response steps"
types:
  - example/cbr
labels:
  - example/ensemble

spec:

  ## ─── CBR Configuration ────────────────────────────────────────────
  ## Feature extraction defines how the current incident maps to past
  ## cases. Domain scopes retrieval to incident-triage history.
  cbr:
    features:
      severity: ".incident.severity"
      category: ".incident.category"
      affectedSystem: ".incident.affectedSystem"
    domain: incident-triage
    caseType: ensemble-triage
    topK: 5
    minSimilarity: 0.3
    crossType: false
    problemDescription: ".incident.summary"

  ## ─── Capabilities ─────────────────────────────────────────────────
  capabilities:
    - name: triage
      description: "Triages incident using ensemble consensus from past responses"
      inputProjection: |
        {
          incident: .incident,
          ensemble: .cbrEnsemble
        }
      outputProjection: |
        {
          triageResult: {
            severity: .severity,
            approach: .approach,
            contestedSteps: .contestedSteps,
            confidence: .confidence
          }
        }

    - name: respond
      description: "Executes the triage-recommended response"
      inputProjection: "{ incident: .incident, triageResult: .triageResult }"
      outputProjection: "{ responseResult: { status: .status, actions: .actions } }"

  ## ─── Workers ──────────────────────────────────────────────────────
  workers:
    - name: triage-analyst
      description: "Analyses incident with CBR ensemble context"
      capabilities:
        - triage
      agent:
        model:
          anthropic:
            modelName: claude-sonnet-4-20250514
        systemPrompt: |
          You are an incident triage analyst. You receive an incident and
          CBR ensemble consensus from past similar incidents.

          Check .cbrEnsemble.scope — if "STEP_LEVEL", examine stepAnalysis:
          - UNANIMOUS steps: high confidence — follow these
          - CONSENSUS steps: reasonable confidence — follow with monitoring
          - CONTESTED steps: disagreement — flag for human review
          - MINORITY/UNIQUE steps: low evidence — skip unless clearly relevant

          JQ to find contested steps:
            .cbrEnsemble.stepAnalysis[] | select(.agreement == "CONTESTED")

          Return your triage assessment with severity, recommended approach,
          any contested steps that need human attention, and confidence level.
        inputProjection: "."
        outputProjection: "."

    - name: responder
      description: "Executes the recommended response actions"
      capabilities:
        - respond
      agent:
        model:
          anthropic:
            modelName: claude-sonnet-4-20250514
        systemPrompt: |
          You are an incident responder. Execute the triage-recommended
          response actions for this incident.
        inputProjection: "."
        outputProjection: "."

  ## ─── Bindings ─────────────────────────────────────────────────────
  bindings:
    - name: triage-on-incident
      capability: triage
      on:
        contextChange:
          filter: '.incident != null and .triageResult == null'

    - name: respond-after-triage
      capability: respond
      on:
        contextChange:
          filter: '.triageResult != null and .responseResult == null'

  ## ─── Goals & Completion ───────────────────────────────────────────
  goals:
    - name: incidentResolved
      description: "Incident has been triaged and responded to"
      condition: '.responseResult != null'
      kind: success

  completion:
    success:
      allOf:
        - incidentResolved
```

- [ ] **Step 2: Verify YAML validates against schema**

Run: `TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn compile -pl schema -q`
Expected: BUILD SUCCESS

- [ ] **Step 3: Commit**

```bash
git add examples/yaml/cbr-ensemble-consensus.yaml
git commit -m "feat: add CBR ensemble consensus YAML example

Demonstrates cbr: block with feature extraction, domain, caseType,
crossType, problemDescription. Agent reads .cbrEnsemble.stepAnalysis
from context to identify CONTESTED steps.

Refs #1098"
```

### Task 6: DSL example — cbr-ensemble-dsl module

**Files:**
- Create: `examples/cbr-ensemble-dsl/pom.xml`
- Create: `examples/cbr-ensemble-dsl/src/main/java/io/casehub/examples/CbrEnsembleTriageCase.java`
- Modify: `pom.xml` (root) — add module to `<modules>`

**Interfaces:**
- Consumes: `CaseDefinition.builder()`, `CbrConfig.builder()`, `Capability.of()`, `Worker.builder()`, `Binding.builder()`
- Produces: `CbrEnsembleTriageCase.define()` → `CaseDefinition` with `CbrConfig`

- [ ] **Step 1: Create pom.xml**

Create `examples/cbr-ensemble-dsl/pom.xml` following the pattern from `examples/goap-dsl/pom.xml`. Key dependencies: `casehub-engine-api`, `casehub-worker-api`. Set `<maven.deploy.skip>true</maven.deploy.skip>`.

- [ ] **Step 2: Add module to root pom.xml**

Add `<module>examples/cbr-ensemble-dsl</module>` to the `<modules>` section in root `pom.xml`.

- [ ] **Step 3: Write CbrEnsembleTriageCase.java**

Create `examples/cbr-ensemble-dsl/src/main/java/io/casehub/examples/CbrEnsembleTriageCase.java`:

A `define()` method that builds a `CaseDefinition` with:
- `CbrConfig.builder().feature("severity", ".incident.severity").feature("category", ".incident.category").domain("incident-triage").caseType("ensemble-triage").topK(5).build()`
- Capability "triage" with input projection including `.cbrEnsemble`
- Capability "respond"
- Workers, bindings, goals, completion — same scenario as the YAML example

Follow the exact pattern from `GoapContractReviewCase.java`.

- [ ] **Step 4: Verify compilation**

Run: `TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn compile -pl examples/cbr-ensemble-dsl -q`
Expected: BUILD SUCCESS

- [ ] **Step 5: Commit**

```bash
git add examples/cbr-ensemble-dsl/ pom.xml
git commit -m "feat: add CBR ensemble consensus DSL example

Java DSL pathway showing CbrConfig.builder() with feature extraction,
domain, caseType. Same triage scenario as YAML and annotated examples.

Refs #1098"
```

### Task 7: Annotated example — cbr-ensemble-annotated module

**Files:**
- Create: `examples/cbr-ensemble-annotated/pom.xml`
- Create: `examples/cbr-ensemble-annotated/src/main/java/io/casehub/examples/CbrEnsembleTriageCase.java`
- Modify: `pom.xml` (root) — add module to `<modules>`

**Interfaces:**
- Consumes: `@Case`, `@Cbr`, `@Feature`, `@Worker`, `@Bind`, `@SystemPrompt` annotations from Task 2
- Produces: annotated `@Case` interface demonstrating `@Cbr` usage

- [ ] **Step 1: Create pom.xml**

Create `examples/cbr-ensemble-annotated/pom.xml` following the pattern from `examples/incident-response-annotated/pom.xml`. Key dependencies: `casehub-engine-annotations`, `casehub-engine-annotations-deployment`. Set `<maven.deploy.skip>true</maven.deploy.skip>`.

- [ ] **Step 2: Add module to root pom.xml**

Add `<module>examples/cbr-ensemble-annotated</module>` to the `<modules>` section in root `pom.xml`.

- [ ] **Step 3: Write CbrEnsembleTriageCase.java**

Create `examples/cbr-ensemble-annotated/src/main/java/io/casehub/examples/CbrEnsembleTriageCase.java`:

```java
package io.casehub.examples;

import io.casehub.api.model.GoalExpression;
import io.casehub.engine.annotations.*;

@Case(
    namespace = "example",
    name = "EnsembleTriage",
    version = "1.0.0",
    title = "Ensemble Triage",
    summary = "Incident triage informed by CBR ensemble consensus")
@Cbr(
    features = {
        @Feature(name = "severity", expression = ".incident.severity"),
        @Feature(name = "category", expression = ".incident.category"),
        @Feature(name = "affectedSystem", expression = ".incident.affectedSystem")
    },
    domain = "incident-triage",
    caseType = "EnsembleTriage",
    topK = 5,
    minSimilarity = 0.3,
    problemDescription = ".incident.summary"
)
public interface CbrEnsembleTriageCase {

    @Worker(capability = "triage",
        description = "Triages incident using ensemble consensus from past responses")
    @Bind(contextChange = ".incident != null and .triageResult == null")
    @SystemPrompt("""
        You are an incident triage analyst. Check .cbrEnsemble.scope —
        if STEP_LEVEL, examine stepAnalysis for CONTESTED steps.
        Return triage assessment with severity and confidence.""")
    TriageResult triage(Incident incident);

    @Worker(capability = "respond",
        description = "Executes the triage-recommended response")
    @Bind(contextChange = ".triageResult != null and .responseResult == null")
    @SystemPrompt("Execute the triage-recommended response actions.")
    ResponseResult respond(Incident incident, TriageResult triageResult);

    @Goal(value = "Incident resolved", condition = ".responseResult != null")
    @Completion
    default GoalExpression resolved() {
        return GoalExpression.goal("resolved");
    }

    record Incident(String severity, String category, String affectedSystem, String summary) {}
    record TriageResult(String severity, String approach, java.util.List<String> contestedSteps, double confidence) {}
    record ResponseResult(String status, java.util.List<String> actions) {}
}
```

- [ ] **Step 4: Verify compilation**

Run: `TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn compile -pl examples/cbr-ensemble-annotated -q`
Expected: BUILD SUCCESS

- [ ] **Step 5: Commit**

```bash
git add examples/cbr-ensemble-annotated/ pom.xml
git commit -m "feat: add CBR ensemble consensus annotated example

Annotation pathway showing @Cbr with @Feature for CBR config.
Same triage scenario as YAML and DSL examples.

Closes #1098"
```

---

## Batch 4: CLAUDE.md Updates + Issue Creation

### Task 8: Update CLAUDE.md and create @Cbr annotation issue

**Files:**
- Modify: `CLAUDE.md`

**Interfaces:**
- Consumes: completed implementation from Tasks 1-7
- Produces: updated documentation, GitHub issue for tracking

- [ ] **Step 1: Update CLAUDE.md — annotations section**

Add after the `@Cost` paragraph in `## casehub-engine-annotations Module`:

```
`@Cbr` — case-level CBR retrieval configuration. Nested `@Feature(name, expression)` for JQ feature extractors, `@Weight(name, value)` for feature weights. All `CbrConfig` fields supported: `topK`, `minSimilarity`, `domain`, `caseType`, `vectorWeight`, `timing`, `cbrType`, `temporalDecayHalfLifeDays`, `minCostSamples`, `crossType`, `problemDescription`. Sentinel `-1` for optional integers (Java annotations don't support null). Processor: `AnnotationValidationStep` validates features non-empty; `CaseDefinitionRecorder` maps to `CbrConfig.builder()`. Refs engine#1098.
```

- [ ] **Step 2: Update CLAUDE.md — CBR Retrieval Bridge section**

Add after the existing `CbrRetrievalResult` paragraph:

```
`retrieveForSelectionWithEnsemble()` — overloads on `CbrRetrievalService` returning `CbrRetrievalResult` for selection queries. Takes `@Nullable String caseType` for ensemble analysis. No adaptation (no `CaseDefinition` available) — raw plan traces wrapped as RETAINED `AdaptedPlan`. Same timeout and error handling as `retrieveInternal()`. Refs engine#1097.
```

- [ ] **Step 3: Update CLAUDE.md — annotations list**

In `## casehub-engine-annotations Module`, update the **Annotations** list to include `@Cbr`, `@Feature`, `@Weight`.

- [ ] **Step 4: Commit**

```bash
git add CLAUDE.md
git commit -m "docs: update CLAUDE.md with @Cbr and retrieveForSelectionWithEnsemble

Refs #1097, #1098"
```

## References

- `specs/issue-1097-cbr-ensemble-retrieve-for-selection/2026-09-15-cbr-ensemble-gaps-design.md` — design spec
- `runtime-core/src/main/java/io/casehub/engine/internal/routing/CbrRetrievalService.java:204` — retrieveForSelection overloads
- `runtime-core/src/main/java/io/casehub/engine/internal/routing/CbrRetrievalService.java:373` — computeEnsemble
- `api/src/main/java/io/casehub/api/model/cbr/CbrConfig.java:27` — CbrConfig record
- `schema/src/main/resources/schema/CaseDefinition.yaml:1154` — Cbr schema
- `annotations/deployment/src/main/java/.../EngineAnnotationsProcessor.java:154` — buildDescriptor
- `annotations/runtime/src/main/java/.../CaseDefinitionRecorder.java:56` — createCaseDefinitionInternal
- `annotations/runtime/src/main/java/.../CaseDescriptor.java:21` — CaseDescriptor record
- `examples/goap-dsl/src/main/java/.../GoapContractReviewCase.java` — DSL example pattern
- `examples/incident-response-annotated/` — annotated example pattern
- `docs/specs/issue-1051-cbr-ensemble-consensus/` — parent design spec
- GitHub #1097 — retrieveForSelection ensemble
- GitHub #1098 — three-pathway example
