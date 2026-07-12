# CBR Generalization, Feature Similarity & DAG Plan Implementation

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #704 — CbrRetrievalService generalize beyond PlanCbrCase
**Issue group:** #704, #672, #694

**Goal:** Generalize CBR retrieval beyond PlanCbrCase, surface per-feature similarity breakdown, and replace flat task lists with a DAG plan structure.

**Architecture:** Three independent issues implemented sequentially across three repos. #704 and #672 modify the engine's CBR retrieval pipeline (with #672 requiring upstream neocortex changes first). #694 introduces `ExecutionPlan<T>` in the blocks repo as a DAG replacement for `List<TaskNode<T>>`.

**Tech Stack:** Java 21, Quarkus 3.32.2, Mutiny (reactive), jackson-jq, JUnit 5

## Global Constraints

- Pre-release: breaking changes cost nothing
- IntelliJ MCP mandatory for all `.java` edits — no bash grep, no Edit tool on source files
- TDD: write failing test → verify fail → implement → verify pass → commit
- Each issue committed separately with `Refs #N` or `Closes #N`
- Cross-repo dependency: #672 requires neocortex changes installed to local Maven before engine changes
- Neocortex changes need a branch in `/Users/mdproctor/claude/casehub/neocortex`
- Blocks changes need a branch in `/Users/mdproctor/claude/casehub/blocks`

---

### Task 1: #704 — CbrConfig gains `cbrType` + CbrCaseTypeRegistration + CbrRetrievalService generalization

This is the complete #704 implementation — config, SPI, service, YAML, tests.

**Repos:** engine only
**Files:**
- Modify: `api/src/main/java/io/casehub/api/model/cbr/CbrConfig.java` — add `cbrType` field, builder method
- Create: `api/src/main/java/io/casehub/api/model/cbr/CbrCaseTypeRegistration.java` — CDI marker interface
- Modify: `runtime/src/main/java/io/casehub/engine/internal/routing/CbrRetrievalService.java` — generic overload, type resolution, generic mapping
- Modify: `api/src/main/java/io/casehub/api/model/converter/CaseDefinitionYamlMapper.java` — parse `cbrType`
- Modify: `runtime/src/test/java/io/casehub/engine/internal/routing/CbrRetrievalServiceTest.java` — tests for generic path
- Modify: `api/src/test/java/io/casehub/api/model/cbr/CbrConfigTest.java` — cbrType validation tests
- Modify: `api/src/test/java/io/casehub/api/model/converter/CaseDefinitionYamlMapperCbrTest.java` — YAML cbrType test

**Interfaces:**
- Produces: `CbrConfig.cbrType()` — `String`, nullable, defaults to `"plan"` when null
- Produces: `CbrCaseTypeRegistration` — `cbrType(): String`, `caseClass(): Class<? extends CbrCase>`
- Produces: `CbrRetrievalService.retrieve(CaseDefinition, CaseInstance, Class<C>)` — generic overload

- [ ] **Step 1: Write failing test — CbrConfig cbrType field**

In `api/src/test/java/io/casehub/api/model/cbr/CbrConfigTest.java`, add:

```java
@Test
void cbrType_defaults_to_null() {
    CbrConfig config = CbrConfig.builder()
        .featureExtractor(ctx -> Map.of("f1", "v1"))
        .build();
    assertNull(config.cbrType());
}

@Test
void cbrType_set_via_builder() {
    CbrConfig config = CbrConfig.builder()
        .featureExtractor(ctx -> Map.of("f1", "v1"))
        .cbrType("feature-vector")
        .build();
    assertEquals("feature-vector", config.cbrType());
}

@Test
void cbrType_blank_rejected() {
    assertThrows(IllegalArgumentException.class, () ->
        CbrConfig.builder()
            .featureExtractor(ctx -> Map.of("f1", "v1"))
            .cbrType("  ")
            .build());
}
```

- [ ] **Step 2: Run tests — verify they fail**

```bash
/opt/homebrew/bin/mvn test -pl api -Dtest="CbrConfigTest" -q
```

Expected: compilation errors (no `cbrType` field or builder method yet).

- [ ] **Step 3: Implement CbrConfig.cbrType**

Add `cbrType` field to `CbrConfig` record (after `timing`):

```java
public record CbrConfig(
    FeatureExtractor featureExtractor,
    int topK,
    double minSimilarity,
    Map<String, Double> weights,
    String domain,
    String caseType,
    double vectorWeight,
    CbrRetrievalTiming timing,
    String cbrType)
```

In compact constructor, add:
```java
if (cbrType != null && cbrType.isBlank()) {
    throw new IllegalArgumentException("cbrType must not be blank when provided");
}
```

In `Builder`, add field `private String cbrType;` and method:
```java
public Builder cbrType(final String cbrType) {
    this.cbrType = cbrType;
    return this;
}
```

Update `build()` to pass `cbrType`:
```java
return new CbrConfig(extractor, topK, minSimilarity, weights, domain, caseType, vectorWeight, timing, cbrType);
```

- [ ] **Step 4: Run tests — verify they pass**

```bash
/opt/homebrew/bin/mvn test -pl api -Dtest="CbrConfigTest" -q
```

- [ ] **Step 5: Create CbrCaseTypeRegistration interface**

Create `api/src/main/java/io/casehub/api/model/cbr/CbrCaseTypeRegistration.java`:

```java
package io.casehub.api.model.cbr;

import io.casehub.neocortex.memory.cbr.CbrCase;

public interface CbrCaseTypeRegistration {
    String cbrType();
    Class<? extends CbrCase> caseClass();
}
```

- [ ] **Step 6: Write failing test — CbrRetrievalService generic retrieval**

In `CbrRetrievalServiceTest.java`, add:

```java
@Test
void retrieve_with_explicit_feature_vector_case_type() {
    CbrConfig config = CbrConfig.builder()
        .featureExtractor(ctx -> Map.of("f1", "v1"))
        .domain("test")
        .cbrType("feature-vector")
        .build();
    CaseDefinition def = buildDefinition(config);
    FeatureVectorCbrCase fvCase = new FeatureVectorCbrCase(
        "problem1", "solution1", "COMPLETED", 0.9, Map.of("f1", "v1"));
    cbrStore.setResult(List.of(new ScoredCbrCase<>(fvCase, 0.85)));

    List<RetrievedExperience> result =
        service.retrieve(def, buildInstance()).await().indefinitely();

    assertEquals(1, result.size());
    RetrievedExperience exp = result.get(0);
    assertEquals("problem1", exp.problem());
    assertEquals("solution1", exp.solution());
    assertTrue(exp.planTrace().isEmpty());
}

@Test
void retrieve_with_explicit_class_overload() {
    CbrConfig config = CbrConfig.builder()
        .featureExtractor(ctx -> Map.of("f1", "v1"))
        .domain("test")
        .build();
    CaseDefinition def = buildDefinition(config);
    FeatureVectorCbrCase fvCase = new FeatureVectorCbrCase(
        "problem1", "solution1", "COMPLETED", 0.8, Map.of("f1", "v1"));
    cbrStore.setResult(List.of(new ScoredCbrCase<>(fvCase, 0.75)));

    List<RetrievedExperience> result =
        service.retrieve(def, buildInstance(), FeatureVectorCbrCase.class)
            .await().indefinitely();

    assertEquals(1, result.size());
    assertTrue(result.get(0).planTrace().isEmpty());
}

@Test
void unknown_cbrType_throws() {
    CbrConfig config = CbrConfig.builder()
        .featureExtractor(ctx -> Map.of("f1", "v1"))
        .domain("test")
        .cbrType("nonexistent")
        .build();
    CaseDefinition def = buildDefinition(config);

    assertThrows(IllegalStateException.class, () ->
        service.retrieve(def, buildInstance()).await().indefinitely());
}

@Test
void plan_case_still_maps_plan_trace() {
    CbrConfig config = CbrConfig.builder()
        .featureExtractor(ctx -> Map.of("f1", "v1"))
        .domain("test")
        .cbrType("plan")
        .build();
    CaseDefinition def = buildDefinition(config);
    PlanTrace planTrace = new PlanTrace("bind1", "cap1", "worker1", "SUCCESS", 0, Map.of());
    PlanCbrCase planCase = new PlanCbrCase(
        "problem1", "solution1", "COMPLETED", 0.95,
        Map.of("f1", "v1"), List.of(planTrace));
    cbrStore.setResult(List.of(new ScoredCbrCase<>(planCase, 0.9)));

    List<RetrievedExperience> result =
        service.retrieve(def, buildInstance()).await().indefinitely();

    assertEquals(1, result.size());
    assertEquals(1, result.get(0).planTrace().size());
    assertEquals("bind1", result.get(0).planTrace().get(0).bindingName());
}
```

- [ ] **Step 7: Run tests — verify they fail**

```bash
/opt/homebrew/bin/mvn test -pl runtime -Dtest="CbrRetrievalServiceTest" -q
```

- [ ] **Step 8: Implement CbrRetrievalService generalization**

In `CbrRetrievalService.java`:

1. Add imports for `FeatureVectorCbrCase`, `TextualCbrCase`, `CbrCaseTypeRegistration`, CDI `Instance`
2. Add static type map and CDI injection:

```java
private static final Map<String, Class<? extends CbrCase>> BUILT_IN_TYPES = Map.of(
    "plan", PlanCbrCase.class,
    "feature-vector", FeatureVectorCbrCase.class,
    "textual", TextualCbrCase.class);

private final Map<String, Class<? extends CbrCase>> typeMap;
```

3. Update constructor to accept `Instance<CbrCaseTypeRegistration>`:

```java
@Inject
public CbrRetrievalService(JQEvaluator jqEvaluator, CbrCaseMemoryStore cbrStore,
                            @All Instance<CbrCaseTypeRegistration> registrations) {
    this.jqEvaluator = jqEvaluator;
    this.cbrStore = cbrStore;
    this.typeMap = buildTypeMap(registrations);
}

private static Map<String, Class<? extends CbrCase>> buildTypeMap(
        Instance<CbrCaseTypeRegistration> registrations) {
    Map<String, Class<? extends CbrCase>> map = new java.util.HashMap<>(BUILT_IN_TYPES);
    for (CbrCaseTypeRegistration reg : registrations) {
        Class<? extends CbrCase> existing = map.put(reg.cbrType(), reg.caseClass());
        if (existing != null && !BUILT_IN_TYPES.containsKey(reg.cbrType())) {
            throw new IllegalStateException(
                "Duplicate CbrCaseTypeRegistration for cbrType '" + reg.cbrType() + "'");
        }
    }
    return Map.copyOf(map);
}
```

4. Add the generic overload:

```java
public <C extends CbrCase> Uni<List<RetrievedExperience>> retrieve(
        CaseDefinition definition, CaseInstance instance, Class<C> caseClass) {
    return retrieveInternal(definition, instance, caseClass);
}
```

5. Refactor existing `retrieve()` to delegate:

```java
public Uni<List<RetrievedExperience>> retrieve(CaseDefinition definition, CaseInstance instance) {
    return Uni.createFrom().<List<RetrievedExperience>>deferred(() -> {
        CbrConfig config = definition.getCbrConfig();
        if (config == null) {
            return Uni.createFrom().item(List.of());
        }
        String cbrType = config.cbrType() != null ? config.cbrType() : "plan";
        Class<? extends CbrCase> caseClass = typeMap.get(cbrType);
        if (caseClass == null) {
            throw new IllegalStateException("Unknown cbrType: " + cbrType);
        }
        return retrieveInternal(definition, instance, caseClass);
    }).onFailure().recoverWithItem(failure -> {
        LOG.warnf(failure, "CBR retrieval failed for case definition '%s'", definition.getName());
        return List.of();
    });
}
```

6. Extract `retrieveInternal()` — same as current `retrieve()` body but using the `caseClass` parameter instead of `PlanCbrCase.class`

7. Make `mapResults` and `mapScoredCase` generic:

```java
private <C extends CbrCase> List<RetrievedExperience> mapResults(List<ScoredCbrCase<C>> scoredCases) {
    return scoredCases.stream().map(this::mapScoredCaseGeneric).toList();
}

private <C extends CbrCase> RetrievedExperience mapScoredCaseGeneric(ScoredCbrCase<C> scored) {
    CbrCase c = scored.cbrCase();
    List<ExperiencePlanStep> trace = (c instanceof PlanCbrCase plan)
        ? mapPlanTrace(plan.planTrace()) : List.of();
    return new RetrievedExperience(
        c.problem(), c.solution(), c.outcome(), c.confidence(),
        scored.score(), c.features(), trace);
}
```

Note: the test constructor `new CbrRetrievalService(jqEvaluator, cbrStore)` needs updating — add a no-registrations constructor or pass empty `Instance`. Simplest: add a package-private constructor for testing:

```java
CbrRetrievalService(JQEvaluator jqEvaluator, CbrCaseMemoryStore cbrStore) {
    this.jqEvaluator = jqEvaluator;
    this.cbrStore = cbrStore;
    this.typeMap = Map.copyOf(BUILT_IN_TYPES);
}
```

- [ ] **Step 9: Add cbrType to YAML mapper**

In `CaseDefinitionYamlMapper.java`, after the `timing` block (~line 555), add:

```java
if (cbr.getCbrType() != null) {
    cbrBuilder.cbrType(cbr.getCbrType());
}
```

If the generated `Cbr` schema class doesn't have `getCbrType()`, add `cbrType` to the YAML schema definition in the JSON schema file, then regenerate. Alternatively, read from the raw `cbrNode`:

```java
if (cbrNode != null && cbrNode.has("cbrType")) {
    cbrBuilder.cbrType(cbrNode.get("cbrType").asText());
}
```

- [ ] **Step 10: Run all tests — verify they pass**

```bash
TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn install -DskipTests -q
TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn test -pl api -Dtest="CbrConfigTest,CaseDefinitionYamlMapperCbrTest" -q
TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn test -pl runtime -Dtest="CbrRetrievalServiceTest" -q
```

- [ ] **Step 11: Build full project to check for compilation errors**

```bash
ide_build_project
```

- [ ] **Step 12: Commit**

```bash
git add -A
git commit -m "feat(#704): generalize CbrRetrievalService beyond PlanCbrCase

- CbrConfig gains cbrType field (string, nullable, defaults to plan)
- CbrCaseTypeRegistration CDI interface for extensible type mapping
- CbrRetrievalService: generic overload, generic mapping via CbrCase interface
- YAML mapper parses cbrType from cbr block

Closes #704"
```

---

### Task 2: #672 upstream — ScoredCbrCase + CbrSimilarityScorer (neocortex)

Upstream changes in the neocortex repo required before engine-side #672 work.

**Repos:** neocortex (`/Users/mdproctor/claude/casehub/neocortex`)
**Files:**
- Modify: `memory-api/src/main/java/io/casehub/neocortex/memory/cbr/ScoredCbrCase.java` — add `featureSimilarities` field
- Modify: `memory-api/src/main/java/io/casehub/neocortex/memory/cbr/CbrSimilarityScorer.java` — add `SimilarityBreakdown`, `scoreDetailed()`
- Modify: `memory-cbr-inmem/src/main/java/io/casehub/neocortex/memory/cbr/inmem/InMemoryCbrCaseMemoryStore.java` — use `scoreDetailed()`
- Modify: `memory-cbr-crossencoder/src/main/java/io/casehub/neocortex/memory/cbr/crossencoder/RerankingCbrCaseMemoryStore.java` — preserve featureSimilarities
- Modify: tests for all above
- Test: `memory-api/src/test/java/io/casehub/neocortex/memory/cbr/ScoredCbrCaseTest.java`
- Test: `memory-api/src/test/java/io/casehub/neocortex/memory/cbr/CbrSimilarityScorerTest.java`

**Interfaces:**
- Produces: `ScoredCbrCase.featureSimilarities()` — `Map<String, Double>`, never null
- Produces: `CbrSimilarityScorer.SimilarityBreakdown` — record `(double score, Map<String, Double> featureSimilarities)`
- Produces: `CbrSimilarityScorer.scoreDetailed()` — same params as `score()`, returns `SimilarityBreakdown`

- [ ] **Step 1: Create neocortex branch**

```bash
git -C /Users/mdproctor/claude/casehub/neocortex checkout -b issue-672-feature-similarity-breakdown
```

- [ ] **Step 2: Write failing test — ScoredCbrCase featureSimilarities**

In `memory-api/src/test/java/io/casehub/neocortex/memory/cbr/ScoredCbrCaseTest.java`, add:

```java
@Test
void featureSimilarities_present() {
    Map<String, Double> sims = Map.of("posture", 0.6, "size", 0.3);
    PlanCbrCase c = new PlanCbrCase("p", "s", "o", 0.5, Map.of(), List.of());
    ScoredCbrCase<PlanCbrCase> scored = new ScoredCbrCase<>(c, 0.9, false, sims);
    assertEquals(sims, scored.featureSimilarities());
}

@Test
void featureSimilarities_immutable() {
    Map<String, Double> sims = new java.util.HashMap<>();
    sims.put("a", 0.5);
    PlanCbrCase c = new PlanCbrCase("p", "s", "o", 0.5, Map.of(), List.of());
    ScoredCbrCase<PlanCbrCase> scored = new ScoredCbrCase<>(c, 0.9, false, sims);
    assertThrows(UnsupportedOperationException.class, () ->
        scored.featureSimilarities().put("b", 0.1));
}

@Test
void twoArgConstructor_emptyFeatureSimilarities() {
    PlanCbrCase c = new PlanCbrCase("p", "s", "o", 0.5, Map.of(), List.of());
    ScoredCbrCase<PlanCbrCase> scored = new ScoredCbrCase<>(c, 0.9);
    assertEquals(Map.of(), scored.featureSimilarities());
}

@Test
void withReranked_preservesFeatureSimilarities() {
    Map<String, Double> sims = Map.of("posture", 0.6);
    PlanCbrCase c = new PlanCbrCase("p", "s", "o", 0.5, Map.of(), List.of());
    ScoredCbrCase<PlanCbrCase> scored = new ScoredCbrCase<>(c, 0.9, false, sims);
    ScoredCbrCase<PlanCbrCase> reranked = scored.withReranked();
    assertTrue(reranked.reranked());
    assertEquals(sims, reranked.featureSimilarities());
}
```

- [ ] **Step 3: Run tests — verify they fail**

```bash
TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn test -pl memory-api -Dtest="ScoredCbrCaseTest" -f /Users/mdproctor/claude/casehub/neocortex/pom.xml -q
```

- [ ] **Step 4: Implement ScoredCbrCase featureSimilarities**

Update `ScoredCbrCase.java`:

```java
public record ScoredCbrCase<C extends CbrCase>(C cbrCase, double score, boolean reranked,
                                                Map<String, Double> featureSimilarities) {
    public ScoredCbrCase {
        Objects.requireNonNull(cbrCase, "cbrCase required");
        if (!(score >= -1.0 && score <= 1.0))
            throw new IllegalArgumentException("score must be in [-1,1], got: " + score);
        featureSimilarities = featureSimilarities != null ? Map.copyOf(featureSimilarities) : Map.of();
    }

    public ScoredCbrCase(C cbrCase, double score) {
        this(cbrCase, score, false, Map.of());
    }

    public ScoredCbrCase(C cbrCase, double score, boolean reranked) {
        this(cbrCase, score, reranked, Map.of());
    }

    public ScoredCbrCase<C> withReranked() {
        return new ScoredCbrCase<>(cbrCase, score, true, featureSimilarities);
    }
}
```

- [ ] **Step 5: Run ScoredCbrCase tests — verify they pass**

- [ ] **Step 6: Write failing test — CbrSimilarityScorer.scoreDetailed()**

In `CbrSimilarityScorerTest.java`, add:

```java
@Test
void scoreDetailed_returns_breakdown() {
    CbrFeatureSchema schema = CbrFeatureSchema.of("test",
        new FeatureField.Numeric("temperature", 0.0, 100.0, null),
        new FeatureField.Categorical("severity", null));

    Map<String, Object> query = Map.of("temperature", 50.0, "severity", "high");
    Map<String, Object> stored = Map.of("temperature", 60.0, "severity", "high");
    Map<String, Double> weights = Map.of("temperature", 2.0, "severity", 1.0);

    CbrSimilarityScorer.SimilarityBreakdown breakdown =
        CbrSimilarityScorer.scoreDetailed(query, stored, weights, schema, Map.of());

    assertNotNull(breakdown.featureSimilarities());
    assertEquals(2, breakdown.featureSimilarities().size());
    assertTrue(breakdown.featureSimilarities().containsKey("temperature"));
    assertTrue(breakdown.featureSimilarities().containsKey("severity"));
    assertEquals(breakdown.score(), breakdown.featureSimilarities().values().stream()
        .mapToDouble(Double::doubleValue).sum(), 0.001);
}

@Test
void scoreDetailed_matches_score() {
    CbrFeatureSchema schema = CbrFeatureSchema.of("test",
        new FeatureField.Numeric("x", 0.0, 10.0, null));
    Map<String, Object> query = Map.of("x", 5.0);
    Map<String, Object> stored = Map.of("x", 7.0);

    double oldScore = CbrSimilarityScorer.score(query, stored, Map.of(), schema);
    CbrSimilarityScorer.SimilarityBreakdown breakdown =
        CbrSimilarityScorer.scoreDetailed(query, stored, Map.of(), schema, Map.of());

    assertEquals(oldScore, breakdown.score(), 0.0001);
}
```

- [ ] **Step 7: Implement CbrSimilarityScorer.scoreDetailed()**

Add `SimilarityBreakdown` record and `scoreDetailed()` method. The existing `score()` loop body is refactored to capture per-feature contributions. The 4-arg `score()` overload delegates:

```java
public static double score(...) {
    return scoreDetailed(queryFeatures, caseFeatures, weights, schema, Map.of()).score();
}
```

The 5-arg `score()`:
```java
public static double score(..., Map<String, LocalSimilarityFunction> overrides) {
    return scoreDetailed(queryFeatures, caseFeatures, weights, schema, overrides).score();
}
```

- [ ] **Step 8: Run scorer tests — verify they pass**

- [ ] **Step 9: Update InMemoryCbrCaseMemoryStore to use scoreDetailed()**

Replace `CbrSimilarityScorer.score(...)` call with `CbrSimilarityScorer.scoreDetailed(...)`:

```java
CbrSimilarityScorer.SimilarityBreakdown breakdown = CbrSimilarityScorer.scoreDetailed(
    query.features(), stored.cbrCase().features(), query.weights(), schema, Map.of());
double featureScore = breakdown.score();

if (featureScore >= query.minSimilarity()) {
    candidates.add(new ScoredCbrCase<>((C) stored.cbrCase(), featureScore, false,
        breakdown.featureSimilarities()));
}
```

- [ ] **Step 10: Update RerankingCbrCaseMemoryStore**

Find the `new ScoredCbrCase<>` construction in `RerankingCbrCaseMemoryStore` and preserve `featureSimilarities` through reranking:

```java
new ScoredCbrCase<>(original.cbrCase(), sigmoidScore, false, original.featureSimilarities()).withReranked()
```

- [ ] **Step 11: Fix compilation across neocortex — any other ScoredCbrCase construction sites**

Build the project to find all compilation errors from the ScoredCbrCase change. Fix each — most will use the existing 2-arg or 3-arg constructors which delegate with `Map.of()`.

```bash
TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn install -DskipTests -f /Users/mdproctor/claude/casehub/neocortex/pom.xml -q
```

- [ ] **Step 12: Run all neocortex tests**

```bash
TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn test -pl memory-api,memory-cbr-inmem,memory-cbr-crossencoder -f /Users/mdproctor/claude/casehub/neocortex/pom.xml -q
```

- [ ] **Step 13: Install neocortex to local Maven repo**

```bash
TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn install -DskipTests -f /Users/mdproctor/claude/casehub/neocortex/pom.xml -q
```

- [ ] **Step 14: Commit neocortex changes**

```bash
git -C /Users/mdproctor/claude/casehub/neocortex add -A
git -C /Users/mdproctor/claude/casehub/neocortex commit -m "feat: feature-level similarity breakdown in ScoredCbrCase and CbrSimilarityScorer

- ScoredCbrCase gains Map<String, Double> featureSimilarities (never null)
- CbrSimilarityScorer.scoreDetailed() returns SimilarityBreakdown with per-feature contributions
- InMemoryCbrCaseMemoryStore uses scoreDetailed() to populate featureSimilarities
- RerankingCbrCaseMemoryStore preserves featureSimilarities through reranking
- score() delegates to scoreDetailed().score()

Refs engine#672"
```

---

### Task 3: #672 downstream — RetrievedExperience + CbrRetrievalService passthrough (engine)

Engine-side changes that consume the new neocortex API from Task 2.

**Repos:** engine
**Files:**
- Modify: `api/src/main/java/io/casehub/api/spi/routing/RetrievedExperience.java` — add `featureSimilarities` field
- Modify: `runtime/src/main/java/io/casehub/engine/internal/routing/CbrRetrievalService.java` — pass through featureSimilarities
- Modify: `api/src/test/java/io/casehub/api/spi/routing/RetrievedExperienceTest.java` — field tests
- Modify: `runtime/src/test/java/io/casehub/engine/internal/routing/CbrRetrievalServiceTest.java` — passthrough test
- Fix: all `RetrievedExperience` construction sites across engine (add 8th param)

**Interfaces:**
- Consumes: `ScoredCbrCase.featureSimilarities()` (from Task 2)
- Produces: `RetrievedExperience.featureSimilarities()` — `Map<String, Double>`, never null

- [ ] **Step 1: Write failing test — RetrievedExperience featureSimilarities**

In `RetrievedExperienceTest.java`, add:

```java
@Test
void featureSimilarities_present() {
    Map<String, Double> sims = Map.of("temperature", 0.6, "severity", 0.3);
    RetrievedExperience exp = new RetrievedExperience(
        "problem", "solution", "COMPLETED", 0.9, 0.8,
        Map.of("f1", "v1"), List.of(), sims);
    assertEquals(sims, exp.featureSimilarities());
}

@Test
void featureSimilarities_null_becomes_empty() {
    RetrievedExperience exp = new RetrievedExperience(
        "problem", "solution", "COMPLETED", 0.9, 0.8,
        Map.of(), List.of(), null);
    assertEquals(Map.of(), exp.featureSimilarities());
}
```

- [ ] **Step 2: Run tests — verify they fail**

- [ ] **Step 3: Add featureSimilarities to RetrievedExperience**

```java
public record RetrievedExperience(
    String problem, String solution, String outcome, Double confidence,
    double similarityScore, Map<String, Object> features,
    List<ExperiencePlanStep> planTrace,
    Map<String, Double> featureSimilarities) {

  public RetrievedExperience {
    // ... existing validation ...
    featureSimilarities = featureSimilarities != null ? Map.copyOf(featureSimilarities) : Map.of();
  }
}
```

- [ ] **Step 4: Fix all compilation errors**

The 7-arg `RetrievedExperience` constructor now needs an 8th param. Use `ide_build_project` to find all broken construction sites. Fix each by adding `Map.of()` (for sites not involved in CBR passthrough) or `scored.featureSimilarities()` (for CbrRetrievalService).

In `CbrRetrievalService.mapScoredCaseGeneric()`:

```java
return new RetrievedExperience(
    c.problem(), c.solution(), c.outcome(), c.confidence(),
    scored.score(), c.features(), trace,
    scored.featureSimilarities());
```

- [ ] **Step 5: Run tests — verify they pass**

```bash
TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn install -DskipTests -q
TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn test -pl api -Dtest="RetrievedExperienceTest" -q
TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn test -pl runtime -Dtest="CbrRetrievalServiceTest" -q
```

- [ ] **Step 6: Build full project**

```bash
ide_build_project
```

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add -A
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(#672): feature-level similarity breakdown in RetrievedExperience

- RetrievedExperience gains Map<String, Double> featureSimilarities
- CbrRetrievalService passes scored.featureSimilarities() through
- All construction sites updated

Closes #672"
```

---

### Task 4: #694 — ExecutionPlan core type (blocks)

The DAG plan structure — core record, factory methods, validation, tests.

**Repos:** blocks (`/Users/mdproctor/claude/casehub/blocks`)
**Files:**
- Create: `src/main/java/io/casehub/blocks/agentic/plan/ExecutionPlan.java` — core DAG type
- Create: `src/test/java/io/casehub/blocks/agentic/plan/ExecutionPlanTest.java` — comprehensive tests

**Interfaces:**
- Produces: `ExecutionPlan<T>` record — `nodes: Map<String, ExecutionNode<T>>`
- Produces: `ExecutionNode<T>` record — `id, task, dependsOn, joinType`
- Produces: `JoinType` enum — `ALL_OF, ANY_OF`
- Produces: factory methods: `singleton()`, `sequence()`, `parallel()`, `fromList()`, `sequentialMerge()`
- Produces: computed methods: `entryNodeIds()`, `exitNodeIds()`

- [ ] **Step 1: Create blocks branch**

```bash
git -C /Users/mdproctor/claude/casehub/blocks checkout -b issue-694-dag-plan-structure
```

- [ ] **Step 2: Write failing tests — ExecutionPlan**

Create `src/test/java/io/casehub/blocks/agentic/plan/ExecutionPlanTest.java`:

```java
package io.casehub.blocks.agentic.plan;

import static org.junit.jupiter.api.Assertions.*;
import io.casehub.blocks.agentic.AgentRef;
import io.casehub.blocks.agentic.decomposition.TaskNode;
import java.util.*;
import org.junit.jupiter.api.Test;

class ExecutionPlanTest {

    private static TaskNode.PlannedTask<String> task(String desc) {
        return new TaskNode.PlannedTask<>(desc, new AgentRef.ExternalAgent("agent-" + desc, s -> null), null);
    }

    @Test
    void singleton_creates_single_node_plan() {
        ExecutionPlan<String> plan = ExecutionPlan.singleton(task("A"));
        assertEquals(1, plan.nodes().size());
        assertEquals(Set.of("node-0"), plan.entryNodeIds());
        assertEquals(Set.of("node-0"), plan.exitNodeIds());
    }

    @Test
    void sequence_creates_chain() {
        ExecutionPlan<String> plan = ExecutionPlan.sequence(List.of(task("A"), task("B"), task("C")));
        assertEquals(3, plan.nodes().size());
        assertEquals(Set.of("node-0"), plan.entryNodeIds());
        assertEquals(Set.of("node-2"), plan.exitNodeIds());
        assertEquals(Set.of("node-0"), plan.nodes().get("node-1").dependsOn());
        assertEquals(Set.of("node-1"), plan.nodes().get("node-2").dependsOn());
    }

    @Test
    void parallel_creates_independent_nodes() {
        ExecutionPlan<String> plan = ExecutionPlan.parallel(List.of(task("A"), task("B"), task("C")));
        assertEquals(3, plan.nodes().size());
        assertEquals(Set.of("node-0", "node-1", "node-2"), plan.entryNodeIds());
        assertEquals(Set.of("node-0", "node-1", "node-2"), plan.exitNodeIds());
        plan.nodes().values().forEach(n -> assertTrue(n.dependsOn().isEmpty()));
    }

    @Test
    void fromList_same_as_sequence() {
        var tasks = List.of(task("A"), task("B"));
        ExecutionPlan<String> seq = ExecutionPlan.sequence(tasks);
        ExecutionPlan<String> fromList = ExecutionPlan.fromList(tasks);
        assertEquals(seq.nodes().size(), fromList.nodes().size());
    }

    @Test
    void sequentialMerge_chains_subplans() {
        ExecutionPlan<String> a = ExecutionPlan.sequence(List.of(task("A1"), task("A2")));
        ExecutionPlan<String> b = ExecutionPlan.singleton(task("B1"));
        ExecutionPlan<String> merged = ExecutionPlan.sequentialMerge(List.of(a, b));
        assertEquals(3, merged.nodes().size());
        assertEquals(1, merged.entryNodeIds().size());
        assertEquals(1, merged.exitNodeIds().size());
    }

    @Test
    void empty_nodes_rejected() {
        assertThrows(IllegalArgumentException.class, () ->
            new ExecutionPlan<>(Map.of()));
    }

    @Test
    void dangling_dependsOn_rejected() {
        var node = new ExecutionPlan.ExecutionNode<>("n1", task("A"), Set.of("nonexistent"),
            ExecutionPlan.JoinType.ALL_OF);
        assertThrows(IllegalArgumentException.class, () ->
            new ExecutionPlan<>(Map.of("n1", node)));
    }

    @Test
    void cycle_rejected() {
        var n1 = new ExecutionPlan.ExecutionNode<>("n1", task("A"), Set.of("n2"),
            ExecutionPlan.JoinType.ALL_OF);
        var n2 = new ExecutionPlan.ExecutionNode<>("n2", task("B"), Set.of("n1"),
            ExecutionPlan.JoinType.ALL_OF);
        assertThrows(IllegalArgumentException.class, () ->
            new ExecutionPlan<>(Map.of("n1", n1, "n2", n2)));
    }

    @Test
    void nodes_immutable() {
        ExecutionPlan<String> plan = ExecutionPlan.singleton(task("A"));
        assertThrows(UnsupportedOperationException.class, () ->
            plan.nodes().put("x", null));
    }

    @Test
    void joinType_defaults_to_ALL_OF() {
        var node = new ExecutionPlan.ExecutionNode<>("n1", task("A"), Set.of(), null);
        assertEquals(ExecutionPlan.JoinType.ALL_OF, node.joinType());
    }
}
```

- [ ] **Step 3: Run tests — verify they fail**

```bash
TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn test -pl . -Dtest="ExecutionPlanTest" -f /Users/mdproctor/claude/casehub/blocks/pom.xml -q
```

- [ ] **Step 4: Implement ExecutionPlan**

Create `src/main/java/io/casehub/blocks/agentic/plan/ExecutionPlan.java` with:
- `ExecutionNode<T>` inner record
- `JoinType` enum
- Compact constructor with validation (non-empty, reference integrity, cycle detection via Kahn's algorithm)
- `entryNodeIds()` and `exitNodeIds()` computed methods
- Factory methods: `singleton()`, `sequence()`, `parallel()`, `fromList()`, `sequentialMerge()`

- [ ] **Step 5: Run tests — verify they pass**

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks add -A
git -C /Users/mdproctor/claude/casehub/blocks commit -m "feat(#694): ExecutionPlan DAG type with factory methods and validation

Refs engine#694"
```

---

### Task 5: #694 — DecompositionStrategy + strategy updates + pattern builders (blocks)

Change `DecompositionStrategy` return type and update all consumers.

**Repos:** blocks
**Files:**
- Modify: `src/main/java/io/casehub/blocks/agentic/decomposition/DecompositionStrategy.java` — return `ExecutionPlan<T>`
- Modify: `src/main/java/io/casehub/blocks/agentic/decomposition/IdentityDecomposition.java` — singleton/throw
- Modify: `src/main/java/io/casehub/blocks/agentic/decomposition/StaticDecomposition.java` — forward delegate
- Modify: `src/main/java/io/casehub/blocks/agentic/decomposition/LlmDecomposition.java` — wrap as sequence
- Modify: `src/main/java/io/casehub/blocks/agentic/pattern/HtnBuilder.java` — flatten returns ExecutionPlan, topological sort for candidates
- Modify: `src/main/java/io/casehub/blocks/agentic/decomposition/DecompositionMethod.java` — if affected
- Modify: tests for all above

**Interfaces:**
- Consumes: `ExecutionPlan<T>` (from Task 4)
- Produces: `DecompositionStrategy.decompose()` returns `Uni<ExecutionPlan<T>>`

- [ ] **Step 1: Write/update tests for IdentityDecomposition**

Update `IdentityDecompositionTest.java`:

```java
@Test
void leaf_returns_singleton_plan() {
    var leaf = new TaskNode.PlannedTask<String>("task", agent, null);
    var ctx = new DecompositionContext<>("state", List.of(), 0);
    ExecutionPlan<String> plan = new IdentityDecomposition<String>()
        .decompose(leaf, ctx).await().indefinitely();
    assertEquals(1, plan.nodes().size());
}

@Test
void compound_throws() {
    var compound = new TaskNode.CompoundTask<String>("root", List.of());
    var ctx = new DecompositionContext<>("state", List.of(), 0);
    assertThrows(UnsupportedOperationException.class, () ->
        new IdentityDecomposition<String>()
            .decompose(compound, ctx).await().indefinitely());
}
```

- [ ] **Step 2: Change DecompositionStrategy return type**

```java
public interface DecompositionStrategy<T> {
    Uni<ExecutionPlan<T>> decompose(TaskNode<T> compound,
                                     DecompositionContext<T> context);
}
```

- [ ] **Step 3: Update IdentityDecomposition**

```java
@Override
public Uni<ExecutionPlan<T>> decompose(TaskNode<T> node, DecompositionContext<T> context) {
    return switch (node) {
        case TaskNode.LeafTask<T> leaf -> Uni.createFrom().item(ExecutionPlan.singleton(leaf));
        case TaskNode.CompoundTask<T> compound -> throw new UnsupportedOperationException(
            "IdentityDecomposition cannot decompose compound tasks — "
            + "it is a placeholder for non-HTN builders");
    };
}
```

- [ ] **Step 4: Update StaticDecomposition**

The key insight: `StaticDecomposition` iterates `CompoundTask.methods()`, finds the first matching guard, and delegates to `method.strategy().decompose()`. Since the delegate strategy now returns `ExecutionPlan<T>`, `StaticDecomposition` just forwards:

```java
@Override
public Uni<ExecutionPlan<T>> decompose(TaskNode<T> compound, DecompositionContext<T> context) {
    // ... same guard matching logic ...
    return matchingMethod.strategy().decompose(compound, context);
}
```

- [ ] **Step 5: Update LlmDecomposition**

Wrap the LLM-generated tasks as a sequential plan:

```java
// After generating List<TaskNode.PlannedTask<T>> tasks:
return Uni.createFrom().item(ExecutionPlan.sequence(tasks));
```

- [ ] **Step 6: Update HtnBuilder**

`flatten()` returns `ExecutionPlan<T>`. Recursive decomposition collects sub-plans and merges:

```java
private Uni<ExecutionPlan<T>> flatten(TaskNode<T> node, T state) {
    return switch (node) {
        case TaskNode.LeafTask<T> leaf -> Uni.createFrom().item(ExecutionPlan.singleton(leaf));
        case TaskNode.CompoundTask<T> compound -> {
            var matchingMethod = compound.methods().stream()
                .filter(m -> m.guard().test(state))
                .findFirst()
                .orElseThrow(() -> new IllegalStateException(
                    "No decomposition method guard matched for task: " + compound.name()));

            var ctx = new DecompositionContext<>(state, List.of(), 0);
            yield matchingMethod.strategy()
                .decompose(compound, ctx)
                .flatMap(plan -> {
                    // Plan already flattened by the strategy
                    return Uni.createFrom().item(plan);
                });
        }
    };
}
```

In `execute()`, topologically sort the plan into a flat candidate list for backward-compatible sequential execution:

```java
return flatten(rootTask, initialContext)
    .map(plan -> {
        var sortedNodes = plan.topologicalSort();
        var agents = sortedNodes.stream()
            .map(n -> new RoutingCandidate(n.task().agent(), null))
            .toList();
        // ... create local model with agents ...
    })
    .flatMap(localModel -> new OrchestratedDriver<T>().execute(localModel, initialContext));
```

Add `topologicalSort()` method to `ExecutionPlan` (Kahn's algorithm, returns `List<ExecutionNode<T>>`).

- [ ] **Step 7: Fix remaining compilation errors**

Build the blocks project and fix any remaining issues:

```bash
TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn install -DskipTests -f /Users/mdproctor/claude/casehub/blocks/pom.xml -q
```

- [ ] **Step 8: Run all tests**

```bash
TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn test -f /Users/mdproctor/claude/casehub/blocks/pom.xml -q
```

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks add -A
git -C /Users/mdproctor/claude/casehub/blocks commit -m "feat(#694): DecompositionStrategy returns ExecutionPlan, all strategies and builders updated

- DecompositionStrategy.decompose() returns Uni<ExecutionPlan<T>>
- IdentityDecomposition: singleton for leaf, throw for compound
- StaticDecomposition: forwards delegate's ExecutionPlan
- LlmDecomposition: wraps as ExecutionPlan.sequence()
- HtnBuilder.flatten() returns ExecutionPlan, topological sort for candidates
- ExecutionPlan.topologicalSort() for backward-compatible execution

Closes engine#694"
```
