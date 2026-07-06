# CBR Retrieval → Routing Bridge Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #478 — CaseRetriever integration at plan creation — bridge CBR Retrieve step to ImplementationRoutingStrategy
**Issue group:** #478

**Goal:** Bridge the CBR Retrieve step to routing strategies so implementation and agent routing decisions use historical case outcomes.

**Architecture:** Push-based context enrichment. The engine retrieves similar past cases via `CbrCaseMemoryStore.retrieveSimilar()` at the top of each CONTEXT_CHANGED evaluation and threads `List<RetrievedExperience>` through `PlanExecutionContext` → `ImplementationRoutingContext` → `AgentRoutingContext`. Feature extraction is declared on `CaseDefinition` via `CbrConfig` with dual-mode `FeatureExtractor` (JQ for YAML, lambda for Java DSL). The bridge lives in `engine/runtime` which already depends on `casehub-neocortex-memory-api`.

**Tech Stack:** Java 21 (sealed interfaces, records, pattern matching), Quarkus CDI, Mutiny Uni, jackson-jq, casehub-neocortex-memory-api

## Global Constraints

- `FeatureExtractor` is `sealed` — compiler enforces exhaustiveness in `CbrRetrievalService` pattern matching
- CBR types live in `api/src/main/java/io/casehub/api/model/cbr/` — cohesive package, not mixed with `evaluator/`
- Engine-owned result types (`RetrievedExperience`, `ExperiencePlanStep`) live in `api/src/main/java/io/casehub/api/spi/routing/`
- `CbrRetrievalService` wraps the full chain with `.onFailure().recoverWithItem(List.of())` — CBR failure never blocks case progression
- JQ partial extraction: missing features are skipped (DEBUG log), not errors. Only all-null → empty list.
- `engine-api` must NOT depend on `casehub-neocortex-memory-api` — tier separation enforced
- All record constructors use defensive copies and validate invariants
- Build: `mvn install -DskipTests -q` before module-specific tests; always `TESTCONTAINERS_RYUK_DISABLED=true`

---

### Task 1: Feature extraction types + CbrConfig (api module)

**Files:**
- Create: `api/src/main/java/io/casehub/api/model/cbr/FeatureExtractor.java`
- Create: `api/src/main/java/io/casehub/api/model/cbr/JqFeatureExtractor.java`
- Create: `api/src/main/java/io/casehub/api/model/cbr/LambdaFeatureExtractor.java`
- Create: `api/src/main/java/io/casehub/api/model/cbr/CbrConfig.java`
- Create: `api/src/test/java/io/casehub/api/model/cbr/FeatureExtractorTest.java`
- Create: `api/src/test/java/io/casehub/api/model/cbr/CbrConfigBuilderTest.java`

**Interfaces:**
- Consumes: `io.casehub.api.context.CaseContext` (existing)
- Produces: `FeatureExtractor` (sealed interface), `JqFeatureExtractor`, `LambdaFeatureExtractor`, `CbrConfig` + `CbrConfig.Builder` — consumed by Tasks 2, 3, 4, 5

- [ ] **Step 1: Write FeatureExtractor tests**

```java
// api/src/test/java/io/casehub/api/model/cbr/FeatureExtractorTest.java
package io.casehub.api.model.cbr;

import static org.junit.jupiter.api.Assertions.*;
import java.util.Map;
import org.junit.jupiter.api.Test;

class FeatureExtractorTest {

    @Test void jq_type_is_jq() {
        var jq = new JqFeatureExtractor(Map.of("f1", ".x"));
        assertEquals("jq", jq.type());
    }

    @Test void jq_rejects_null_expressions() {
        assertThrows(NullPointerException.class, () -> new JqFeatureExtractor(null));
    }

    @Test void jq_rejects_empty_expressions() {
        assertThrows(IllegalArgumentException.class, () -> new JqFeatureExtractor(Map.of()));
    }

    @Test void jq_defensively_copies_map() {
        var mutable = new java.util.HashMap<String, String>();
        mutable.put("f1", ".x");
        var jq = new JqFeatureExtractor(mutable);
        mutable.put("f2", ".y");
        assertEquals(1, jq.featureExpressions().size());
    }

    @Test void lambda_type_is_lambda() {
        var lambda = new LambdaFeatureExtractor(ctx -> Map.of());
        assertEquals("lambda", lambda.type());
    }

    @Test void lambda_rejects_null_function() {
        assertThrows(NullPointerException.class, () -> new LambdaFeatureExtractor(null));
    }

    @Test void sealed_permits_only_two_subtypes() {
        assertTrue(FeatureExtractor.class.isSealed());
        var permitted = FeatureExtractor.class.getPermittedSubclasses();
        assertEquals(2, permitted.length);
    }
}
```

- [ ] **Step 2: Write CbrConfig builder tests**

```java
// api/src/test/java/io/casehub/api/model/cbr/CbrConfigBuilderTest.java
package io.casehub.api.model.cbr;

import static org.junit.jupiter.api.Assertions.*;
import java.util.Map;
import org.junit.jupiter.api.Test;

class CbrConfigBuilderTest {

    @Test void jq_mode_builds_successfully() {
        var config = CbrConfig.builder()
            .feature("f1", ".x").feature("f2", ".y")
            .topK(3).minSimilarity(0.5).weight("f1", 2.0)
            .domain("test").caseType("game").vectorWeight(0.7)
            .build();
        assertInstanceOf(JqFeatureExtractor.class, config.featureExtractor());
        assertEquals(3, config.topK());
        assertEquals(0.5, config.minSimilarity());
        assertEquals(Map.of("f1", 2.0), config.weights());
        assertEquals("test", config.domain());
        assertEquals("game", config.caseType());
        assertEquals(0.7, config.vectorWeight());
    }

    @Test void lambda_mode_builds_successfully() {
        var config = CbrConfig.builder()
            .featureExtractor(ctx -> Map.of("f1", "v1"))
            .topK(5).build();
        assertInstanceOf(LambdaFeatureExtractor.class, config.featureExtractor());
    }

    @Test void mixing_jq_then_lambda_throws() {
        var builder = CbrConfig.builder().feature("f1", ".x");
        assertThrows(IllegalStateException.class,
            () -> builder.featureExtractor(ctx -> Map.of()));
    }

    @Test void mixing_lambda_then_jq_throws() {
        var builder = CbrConfig.builder().featureExtractor(ctx -> Map.of());
        assertThrows(IllegalStateException.class,
            () -> builder.feature("f1", ".x"));
    }

    @Test void no_features_throws() {
        assertThrows(IllegalStateException.class,
            () -> CbrConfig.builder().topK(5).build());
    }

    @Test void topK_below_1_throws() {
        assertThrows(IllegalArgumentException.class,
            () -> CbrConfig.builder().feature("f1", ".x").topK(0).build());
    }

    @Test void minSimilarity_out_of_range_throws() {
        assertThrows(IllegalArgumentException.class,
            () -> CbrConfig.builder().feature("f1", ".x").minSimilarity(1.1).build());
    }

    @Test void vectorWeight_out_of_range_throws() {
        assertThrows(IllegalArgumentException.class,
            () -> CbrConfig.builder().feature("f1", ".x").vectorWeight(-0.1).build());
    }

    @Test void negative_weight_throws() {
        assertThrows(IllegalArgumentException.class,
            () -> CbrConfig.builder().feature("f1", ".x").weight("f1", -1.0).build());
    }

    @Test void blank_domain_throws() {
        assertThrows(IllegalArgumentException.class,
            () -> CbrConfig.builder().feature("f1", ".x").domain("  ").build());
    }

    @Test void null_domain_allowed() {
        var config = CbrConfig.builder().feature("f1", ".x").build();
        assertNull(config.domain());
    }

    @Test void defaults_applied() {
        var config = CbrConfig.builder().feature("f1", ".x").build();
        assertEquals(5, config.topK());
        assertEquals(0.0, config.minSimilarity());
        assertEquals(0.5, config.vectorWeight());
        assertTrue(config.weights().isEmpty());
        assertNull(config.caseType());
    }
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest="FeatureExtractorTest,CbrConfigBuilderTest" -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: Compilation failure — classes don't exist yet

- [ ] **Step 4: Implement FeatureExtractor, JqFeatureExtractor, LambdaFeatureExtractor, CbrConfig**

Create the `api/src/main/java/io/casehub/api/model/cbr/` package with the four files per the spec §1 and §2. Key implementation notes:
- `FeatureExtractor`: sealed interface permitting `JqFeatureExtractor`, `LambdaFeatureExtractor`
- `JqFeatureExtractor`: record with compact constructor validating non-null, non-empty, defensive `Map.copyOf()`
- `LambdaFeatureExtractor`: final class (not record — holds `Function` which isn't a good record component), `Objects.requireNonNull(fn)`
- `CbrConfig`: record with compact constructor validating all bounds. Builder with `LinkedHashMap` accumulators, mutual exclusivity checks, and `build()` validation

- [ ] **Step 5: Run tests to verify they pass**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest="FeatureExtractorTest,CbrConfigBuilderTest" -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: All tests PASS

- [ ] **Step 6: Commit**

```
feat(#478): add FeatureExtractor sealed hierarchy and CbrConfig

Sealed FeatureExtractor with JqFeatureExtractor (YAML) and
LambdaFeatureExtractor (Java DSL). CbrConfig carries feature
extraction + CBR query parameters with builder enforcing
mutual exclusivity between JQ and lambda modes.

Refs #478
```

---

### Task 2: Engine-owned result types + routing context changes (api module)

**Files:**
- Create: `api/src/main/java/io/casehub/api/spi/routing/RetrievedExperience.java`
- Create: `api/src/main/java/io/casehub/api/spi/routing/ExperiencePlanStep.java`
- Modify: `api/src/main/java/io/casehub/api/spi/routing/ImplementationRoutingContext.java` — add `tenancyId`, `experiences`
- Modify: `api/src/main/java/io/casehub/api/spi/routing/AgentRoutingContext.java` — add `experiences`
- Modify: `api/src/main/java/io/casehub/api/engine/PlanExecutionContext.java` — add `experiences`
- Modify: `api/src/main/java/io/casehub/api/model/CaseDefinition.java` — add `cbrConfig` field, getter, setter, builder method
- Create: `api/src/test/java/io/casehub/api/spi/routing/RetrievedExperienceTest.java`
- Create: `api/src/test/java/io/casehub/api/spi/routing/ExperiencePlanStepTest.java`

**Interfaces:**
- Consumes: `CbrConfig` from Task 1
- Produces: `RetrievedExperience`, `ExperiencePlanStep`, updated `PlanExecutionContext`, `ImplementationRoutingContext`, `AgentRoutingContext`, `CaseDefinition.getCbrConfig()` — consumed by Tasks 3, 4, 5

- [ ] **Step 1: Write RetrievedExperience and ExperiencePlanStep tests**

```java
// api/src/test/java/io/casehub/api/spi/routing/RetrievedExperienceTest.java
package io.casehub.api.spi.routing;

import static org.junit.jupiter.api.Assertions.*;
import java.util.List;
import java.util.Map;
import org.junit.jupiter.api.Test;

class RetrievedExperienceTest {

    @Test void valid_construction() {
        var step = new ExperiencePlanStep("b1", "cap1", "w1", "SUCCESS", 0, Map.of());
        var exp = new RetrievedExperience(
            "problem", "solution", "COMPLETED", 0.9, 0.85,
            Map.of("f1", "v1"), List.of(step));
        assertEquals("problem", exp.problem());
        assertEquals(0.85, exp.similarityScore());
        assertEquals(1, exp.planTrace().size());
    }

    @Test void null_problem_throws() {
        assertThrows(NullPointerException.class,
            () -> new RetrievedExperience(null, "s", "o", 0.9, 0.5, Map.of(), List.of()));
    }

    @Test void null_solution_throws() {
        assertThrows(NullPointerException.class,
            () -> new RetrievedExperience("p", null, "o", 0.9, 0.5, Map.of(), List.of()));
    }

    @Test void score_out_of_range_throws() {
        assertThrows(IllegalArgumentException.class,
            () -> new RetrievedExperience("p", "s", "o", 0.9, 1.1, Map.of(), List.of()));
    }

    @Test void defensive_copies() {
        var features = new java.util.HashMap<String, Object>();
        features.put("k", "v");
        var exp = new RetrievedExperience("p", "s", "o", null, 0.5, features, List.of());
        features.put("k2", "v2");
        assertEquals(1, exp.features().size());
    }

    @Test void null_features_defaults_to_empty() {
        var exp = new RetrievedExperience("p", "s", "o", null, 0.5, null, null);
        assertTrue(exp.features().isEmpty());
        assertTrue(exp.planTrace().isEmpty());
    }
}
```

```java
// api/src/test/java/io/casehub/api/spi/routing/ExperiencePlanStepTest.java
package io.casehub.api.spi.routing;

import static org.junit.jupiter.api.Assertions.*;
import java.util.Map;
import org.junit.jupiter.api.Test;

class ExperiencePlanStepTest {

    @Test void valid_construction() {
        var step = new ExperiencePlanStep("bind1", "cap1", "worker1", "SUCCESS", 1, Map.of("k", "v"));
        assertEquals("bind1", step.bindingName());
        assertEquals(1, step.priority());
    }

    @Test void null_bindingName_throws() {
        assertThrows(NullPointerException.class,
            () -> new ExperiencePlanStep(null, "cap", "w", "ok", 0, Map.of()));
    }

    @Test void null_capabilityName_throws() {
        assertThrows(NullPointerException.class,
            () -> new ExperiencePlanStep("b", null, "w", "ok", 0, Map.of()));
    }

    @Test void negative_priority_throws() {
        assertThrows(IllegalArgumentException.class,
            () -> new ExperiencePlanStep("b", "c", "w", "ok", -1, Map.of()));
    }

    @Test void null_parameters_defaults_to_empty() {
        var step = new ExperiencePlanStep("b", "c", "w", "ok", 0, null);
        assertTrue(step.parameters().isEmpty());
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest="RetrievedExperienceTest,ExperiencePlanStepTest" -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: Compilation failure

- [ ] **Step 3: Implement result types, update routing contexts, update CaseDefinition**

Create `RetrievedExperience.java` and `ExperiencePlanStep.java` per spec §3.

Update `PlanExecutionContext` — add `List<RetrievedExperience> experiences` as final record component.

Update `ImplementationRoutingContext` — add `String tenancyId` and `List<RetrievedExperience> experiences`.

Update `AgentRoutingContext` — add `List<RetrievedExperience> experiences` after `tenancyId`.

Update `CaseDefinition` — add `private CbrConfig cbrConfig` field, `getCbrConfig()`/`setCbrConfig()` accessors, `Builder.cbrConfig(CbrConfig)` method, wire in `build()`.

- [ ] **Step 4: Run tests to verify they pass**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest="RetrievedExperienceTest,ExperiencePlanStepTest" -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: PASS (but other api tests will fail due to record constructor changes — that's expected, fixed in Task 3)

- [ ] **Step 5: Commit**

```
feat(#478): add RetrievedExperience, ExperiencePlanStep, update routing contexts

Add tenancyId to ImplementationRoutingContext (gap fix).
Add List<RetrievedExperience> to PlanExecutionContext,
ImplementationRoutingContext, and AgentRoutingContext.
Add CbrConfig field to CaseDefinition with builder support.

Refs #478
```

---

### Task 3: Fix all broken construction sites (api, runtime, blackboard, ledger, engine-ai)

**Files:**
- Modify: ~22 test files and ~3 production files that construct `PlanExecutionContext`, `ImplementationRoutingContext`, or `AgentRoutingContext` — add `List.of()` for `experiences` and `tenancyId` where needed
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandler.java` — update `PlanExecutionContext` and `AgentRoutingContext` construction (placeholder `List.of()` for now — Task 5 wires real retrieval)
- Modify: `blackboard/src/main/java/io/casehub/blackboard/control/PlanningStrategyLoopControl.java` — update `ImplementationRoutingContext` construction with `ctx.tenancyId()` and `ctx.experiences()`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/orchestration/DefaultWorkOrchestrator.java` — update `AgentRoutingContext` construction
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/WorkflowExecutionCompletedHandler.java` — update `AgentRoutingContext` construction

**Interfaces:**
- Consumes: Updated record types from Task 2
- Produces: Compiling codebase with all tests passing — prerequisite for Tasks 4, 5

- [ ] **Step 1: Run full build to identify all broken sites**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -f /Users/mdproctor/claude/casehub/engine/pom.xml 2>&1 | grep "error:" | head -40`
Expected: Compilation errors at all construction sites

- [ ] **Step 2: Fix all production construction sites**

Use IntelliJ MCP (`ide_find_references`) to locate every `new PlanExecutionContext(`, `new ImplementationRoutingContext(`, `new AgentRoutingContext(` and update each:

Production files:
- `CaseContextChangedEventHandler.rules()` line ~221: `PlanExecutionContext` — add `List.of()` for experiences (placeholder)
- `CaseContextChangedEventHandler.publishWorkerSchedule()` line ~374: `AgentRoutingContext` — add `List.of()` for experiences (placeholder)
- `PlanningStrategyLoopControl.applyImplementationRouting()` line ~169: `ImplementationRoutingContext` — add `ctx.tenancyId()` and `ctx.experiences()`
- `DefaultWorkOrchestrator.doSubmit()`: `AgentRoutingContext` — add `List.of()` for experiences (placeholder)
- `WorkflowExecutionCompletedHandler.fireOutcomeRecorder()`: `AgentRoutingContext` — add `List.of()`

- [ ] **Step 3: Fix all test construction sites**

All mechanical — add `List.of()` for the new `experiences` parameter and `"test-tenant"` for `tenancyId` where applicable.

`PlanExecutionContext` sites (~13): `ChoreographyLoopControlTest`, `DefaultPlanningStrategyTest`, `ImplementationRoutingTest`, `PlanningStrategyContractTest`, `StageLifecycleEvaluatorTest`, `BindingGatingTest` (7 sites), `BlackboardPlanConfigurerTest`, `PlanConfigurerBlackboardTest`

`AgentRoutingContext` sites (~7): `AgentRoutingStrategyContractTest`, `LeastLoadedAgentStrategyTest`, `TrustWeightedAgentStrategyTest`, `SemanticAgentRoutingStrategyTest`

`ImplementationRoutingContext` sites (~2): `NoOpImplementationRoutingStrategyTest`, `TrustWeightedImplementationRoutingStrategyTest`

- [ ] **Step 4: Run full test suite to verify green**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: All tests PASS

- [ ] **Step 5: Commit**

```
refactor(#478): update all routing context construction sites

Mechanical: add List.of() for experiences to all PlanExecutionContext,
ImplementationRoutingContext, and AgentRoutingContext construction
sites. Add tenancyId to ImplementationRoutingContext sites.
Placeholder List.of() in CaseContextChangedEventHandler — real
retrieval wired in next task.

Refs #478
```

---

### Task 4: CbrRetrievalService (runtime module)

**Files:**
- Create: `runtime/src/main/java/io/casehub/engine/internal/routing/CbrRetrievalService.java`
- Create: `runtime/src/test/java/io/casehub/engine/internal/routing/CbrRetrievalServiceTest.java`

**Interfaces:**
- Consumes: `CbrConfig`, `FeatureExtractor` (Task 1), `RetrievedExperience`, `ExperiencePlanStep` (Task 2), `JQEvaluator` (existing), `ReactiveCbrCaseMemoryStore` (neocortex), `CaseInstance` + `CaseDefinition` (existing)
- Produces: `CbrRetrievalService.retrieve(CaseDefinition, CaseInstance) → Uni<List<RetrievedExperience>>` — consumed by Task 5

- [ ] **Step 1: Write CbrRetrievalService tests**

```java
// runtime/src/test/java/io/casehub/engine/internal/routing/CbrRetrievalServiceTest.java
package io.casehub.engine.internal.routing;

import static org.junit.jupiter.api.Assertions.*;
import io.casehub.api.model.CaseDefinition;
import io.casehub.api.model.EpisodicMemoryConfig;
import io.casehub.api.model.cbr.CbrConfig;
import io.casehub.api.spi.routing.RetrievedExperience;
import io.casehub.engine.common.internal.model.CaseInstance;
import io.casehub.engine.internal.jq.JQEvaluator;
import io.casehub.neocortex.memory.MemoryDomain;
import io.casehub.neocortex.memory.cbr.*;
import io.smallrye.mutiny.Uni;
import java.util.*;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

class CbrRetrievalServiceTest {

    // Stub implementations — no Mockito
    private RecordingCbrStore cbrStore;
    private JQEvaluator jqEvaluator;
    private CbrRetrievalService service;

    @BeforeEach void setUp() {
        jqEvaluator = new JQEvaluator();
        cbrStore = new RecordingCbrStore();
        service = new CbrRetrievalService(jqEvaluator, cbrStore);
    }

    @Test void null_cbrConfig_returns_empty() {
        var def = buildDefinition(null);
        var instance = buildInstance();
        var result = service.retrieve(def, instance).await().indefinitely();
        assertTrue(result.isEmpty());
        assertFalse(cbrStore.wasCalled());
    }

    @Test void empty_features_returns_empty() {
        // Lambda extractor that returns empty map
        var config = CbrConfig.builder()
            .featureExtractor(ctx -> Map.of())
            .domain("test").build();
        var def = buildDefinition(config);
        var result = service.retrieve(def, buildInstance()).await().indefinitely();
        assertTrue(result.isEmpty());
        assertFalse(cbrStore.wasCalled());
    }

    @Test void null_domain_no_episodic_returns_empty() {
        var config = CbrConfig.builder()
            .featureExtractor(ctx -> Map.of("f1", "v1"))
            .build(); // domain = null, no episodic config
        var def = buildDefinition(config);
        var result = service.retrieve(def, buildInstance()).await().indefinitely();
        assertTrue(result.isEmpty());
        assertFalse(cbrStore.wasCalled());
    }

    @Test void domain_falls_back_to_episodic() {
        var config = CbrConfig.builder()
            .featureExtractor(ctx -> Map.of("f1", "v1"))
            .build(); // domain = null
        var def = buildDefinition(config);
        def.setEpisodicMemoryConfig(EpisodicMemoryConfig.of("episodic-domain", ".id"));
        cbrStore.setResult(List.of());
        var result = service.retrieve(def, buildInstance()).await().indefinitely();
        assertTrue(cbrStore.wasCalled());
        assertEquals("episodic-domain", cbrStore.lastQuery().domain().name());
    }

    @Test void jq_extraction_builds_correct_query() {
        var config = CbrConfig.builder()
            .feature("posture", ".enemy.posture")
            .feature("size", ".enemy.army_size")
            .weight("posture", 2.0)
            .topK(3).minSimilarity(0.4).domain("sc2").caseType("game")
            .vectorWeight(0.6).build();
        var def = buildDefinition(config);
        var instance = buildInstanceWithContext(Map.of(
            "enemy", Map.of("posture", "aggressive", "army_size", 50)));
        cbrStore.setResult(List.of());
        service.retrieve(def, instance).await().indefinitely();

        var query = cbrStore.lastQuery();
        assertEquals("sc2", query.domain().name());
        assertEquals("game", query.caseType());
        assertEquals(3, query.topK());
        assertEquals(0.4, query.minSimilarity());
        assertEquals(0.6, query.vectorWeight());
        assertEquals("aggressive", query.features().get("posture"));
        assertEquals(50, query.features().get("size"));
        assertEquals(2.0, query.weights().get("posture"));
    }

    @Test void jq_partial_extraction_proceeds_with_available_features() {
        var config = CbrConfig.builder()
            .feature("exists", ".enemy.posture")
            .feature("missing", ".enemy.nonexistent")
            .domain("test").build();
        var def = buildDefinition(config);
        var instance = buildInstanceWithContext(Map.of(
            "enemy", Map.of("posture", "defensive")));
        cbrStore.setResult(List.of());
        service.retrieve(def, instance).await().indefinitely();

        assertTrue(cbrStore.wasCalled());
        var features = cbrStore.lastQuery().features();
        assertEquals(1, features.size());
        assertEquals("defensive", features.get("exists"));
    }

    @Test void jq_all_null_returns_empty() {
        var config = CbrConfig.builder()
            .feature("a", ".nonexistent1")
            .feature("b", ".nonexistent2")
            .domain("test").build();
        var def = buildDefinition(config);
        var instance = buildInstanceWithContext(Map.of());
        var result = service.retrieve(def, instance).await().indefinitely();
        assertTrue(result.isEmpty());
        assertFalse(cbrStore.wasCalled());
    }

    @Test void lambda_extraction_invoked() {
        var config = CbrConfig.builder()
            .featureExtractor(ctx -> Map.of("f1", "extracted"))
            .domain("test").build();
        var def = buildDefinition(config);
        cbrStore.setResult(List.of());
        service.retrieve(def, buildInstance()).await().indefinitely();

        assertTrue(cbrStore.wasCalled());
        assertEquals("extracted", cbrStore.lastQuery().features().get("f1"));
    }

    @Test void results_mapped_to_retrieved_experience() {
        var config = CbrConfig.builder()
            .featureExtractor(ctx -> Map.of("f1", "v1"))
            .domain("test").build();
        var def = buildDefinition(config);
        var planTrace = new PlanTrace("bind1", "cap1", "worker1", "SUCCESS", 0, Map.of());
        var cbrCase = new PlanCbrCase("problem1", "solution1", "COMPLETED", 0.95,
            Map.of("f1", "v1"), List.of(planTrace));
        cbrStore.setResult(List.of(new ScoredCbrCase<>(cbrCase, 0.87)));

        var result = service.retrieve(def, buildInstance()).await().indefinitely();

        assertEquals(1, result.size());
        var exp = result.get(0);
        assertEquals("problem1", exp.problem());
        assertEquals("solution1", exp.solution());
        assertEquals("COMPLETED", exp.outcome());
        assertEquals(0.95, exp.confidence());
        assertEquals(0.87, exp.similarityScore());
        assertEquals(1, exp.planTrace().size());
        assertEquals("bind1", exp.planTrace().get(0).bindingName());
    }

    @Test void store_failure_returns_empty_list() {
        var config = CbrConfig.builder()
            .featureExtractor(ctx -> Map.of("f1", "v1"))
            .domain("test").build();
        var def = buildDefinition(config);
        cbrStore.setFailure(new RuntimeException("Qdrant timeout"));

        var result = service.retrieve(def, buildInstance()).await().indefinitely();
        assertTrue(result.isEmpty()); // recovered, not propagated
    }

    @Test void caseType_defaults_to_definition_name() {
        var config = CbrConfig.builder()
            .featureExtractor(ctx -> Map.of("f1", "v1"))
            .domain("test").build(); // caseType = null
        var def = buildDefinition(config); // name = "test-case"
        cbrStore.setResult(List.of());
        service.retrieve(def, buildInstance()).await().indefinitely();
        assertEquals("test-case", cbrStore.lastQuery().caseType());
    }

    // --- helpers ---

    private CaseDefinition buildDefinition(CbrConfig config) {
        var def = CaseDefinition.builder()
            .namespace("ns").name("test-case").version("1.0.0").build();
        if (config != null) def.setCbrConfig(config);
        return def;
    }

    private CaseInstance buildInstance() {
        return buildInstanceWithContext(Map.of());
    }

    private CaseInstance buildInstanceWithContext(Map<String, Object> workingData) {
        // Build a CaseInstance with working layer data
        // Use the existing test patterns from the engine test suite
        var instance = new CaseInstance();
        instance.tenancyId = "test-tenant";
        workingData.forEach((k, v) -> instance.getCaseContext().set(k, v));
        return instance;
    }

    // Recording stub — no Mockito
    static class RecordingCbrStore implements ReactiveCbrCaseMemoryStore {
        private boolean called;
        private CbrQuery lastQuery;
        private List<ScoredCbrCase<PlanCbrCase>> result;
        private RuntimeException failure;

        void setResult(List<ScoredCbrCase<PlanCbrCase>> result) { this.result = result; }
        void setFailure(RuntimeException e) { this.failure = e; }
        boolean wasCalled() { return called; }
        CbrQuery lastQuery() { return lastQuery; }

        @Override
        @SuppressWarnings("unchecked")
        public <C extends CbrCase> Uni<List<ScoredCbrCase<C>>> retrieveSimilar(
                CbrQuery query, Class<C> caseType) {
            called = true;
            lastQuery = query;
            if (failure != null) return Uni.createFrom().failure(failure);
            return Uni.createFrom().item((List<ScoredCbrCase<C>>) (List<?>) result);
        }

        // No-op for remaining interface methods
        @Override public Uni<Void> registerSchema(CbrFeatureSchema schema) {
            return Uni.createFrom().voidItem();
        }
        @Override public Uni<String> store(CbrCase c, String ct, String eid,
                io.casehub.neocortex.memory.MemoryDomain d, String tid, String cid) {
            return Uni.createFrom().item("");
        }
        @Override public Uni<Integer> erase(io.casehub.neocortex.memory.EraseRequest r) {
            return Uni.createFrom().item(0);
        }
        @Override public Uni<Integer> eraseEntity(String eid, String tid) {
            return Uni.createFrom().item(0);
        }
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest="CbrRetrievalServiceTest" -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: Compilation failure — `CbrRetrievalService` doesn't exist

- [ ] **Step 3: Implement CbrRetrievalService**

Per spec §5 — `@ApplicationScoped`, injects `JQEvaluator` and `ReactiveCbrCaseMemoryStore`. Full implementation with:
- Sealed pattern matching for `FeatureExtractor` dispatch
- `.onFailure().recoverWithItem()` wrapping
- DEBUG logging for skipped JQ features, WARN for domain resolution failure
- `unwrap(JsonNode)` helper for JQ result conversion

- [ ] **Step 4: Run tests to verify they pass**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest="CbrRetrievalServiceTest" -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: All tests PASS

- [ ] **Step 5: Commit**

```
feat(#478): add CbrRetrievalService — CBR retrieval bridge

Evaluates FeatureExtractor (JQ or lambda) against case context,
builds CbrQuery, calls CbrCaseMemoryStore.retrieveSimilar(),
maps ScoredCbrCase<PlanCbrCase> to RetrievedExperience. Failure
recovery ensures CBR errors never block case progression.

Refs #478
```

---

### Task 5: Wire retrieval into CaseContextChangedEventHandler + YAML mapping

**Files:**
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandler.java` — inject `CbrRetrievalService`, call in `rules()`, thread through `publishByTarget`/`publishWorkerSchedule`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/orchestration/DefaultWorkOrchestrator.java` — inject `CbrRetrievalService`, retrieve before agent routing
- Modify: `api/src/main/java/io/casehub/api/model/converter/CaseDefinitionYamlMapper.java` — map `cbr:` YAML block
- Modify: `schema/src/main/resources/schema/CaseDefinition.yaml` — add `cbr` object definition
- Create: `api/src/test/java/io/casehub/api/model/converter/CaseDefinitionYamlMapperCbrTest.java`

**Interfaces:**
- Consumes: `CbrRetrievalService` (Task 4), `CbrConfig` (Task 1), updated contexts (Task 2)
- Produces: Fully wired CBR retrieval in the CONTEXT_CHANGED flow and YAML mapping

- [ ] **Step 1: Write YAML mapping tests**

```java
// api/src/test/java/io/casehub/api/model/converter/CaseDefinitionYamlMapperCbrTest.java
package io.casehub.api.model.converter;

import static org.junit.jupiter.api.Assertions.*;
import io.casehub.api.model.cbr.JqFeatureExtractor;
import java.util.Map;
import org.junit.jupiter.api.Test;

class CaseDefinitionYamlMapperCbrTest {

    @Test void cbr_block_maps_to_cbrConfig() {
        String yaml = """
            dsl: "0.1.0"
            namespace: test
            name: test-case
            version: "1.0.0"
            spec:
              cbr:
                features:
                  posture: ".enemy.posture"
                  build_order: ".enemy.build"
                weights:
                  posture: 2.0
                topK: 3
                minSimilarity: 0.4
                vectorWeight: 0.7
                domain: "sc2"
                caseType: "game"
            """;
        var def = CaseDefinitionYamlMapper.fromYaml(yaml);
        assertNotNull(def.getCbrConfig());
        var config = def.getCbrConfig();
        assertInstanceOf(JqFeatureExtractor.class, config.featureExtractor());
        var jq = (JqFeatureExtractor) config.featureExtractor();
        assertEquals(Map.of("posture", ".enemy.posture", "build_order", ".enemy.build"),
            jq.featureExpressions());
        assertEquals(3, config.topK());
        assertEquals(0.4, config.minSimilarity());
        assertEquals(0.7, config.vectorWeight());
        assertEquals("sc2", config.domain());
        assertEquals("game", config.caseType());
        assertEquals(Map.of("posture", 2.0), config.weights());
    }

    @Test void missing_cbr_block_results_in_null() {
        String yaml = """
            dsl: "0.1.0"
            namespace: test
            name: test-case
            version: "1.0.0"
            spec:
              capabilities:
                - name: cap1
            """;
        var def = CaseDefinitionYamlMapper.fromYaml(yaml);
        assertNull(def.getCbrConfig());
    }

    @Test void cbr_with_defaults_only() {
        String yaml = """
            dsl: "0.1.0"
            namespace: test
            name: test-case
            version: "1.0.0"
            spec:
              cbr:
                features:
                  f1: ".x"
            """;
        var def = CaseDefinitionYamlMapper.fromYaml(yaml);
        assertNotNull(def.getCbrConfig());
        assertEquals(5, def.getCbrConfig().topK());
        assertEquals(0.0, def.getCbrConfig().minSimilarity());
        assertEquals(0.5, def.getCbrConfig().vectorWeight());
    }
}
```

- [ ] **Step 2: Add `cbr` to JSON Schema**

Add to `schema/src/main/resources/schema/CaseDefinition.yaml` under the `spec` properties (after `planningStrategy`):

```yaml
      cbr:
        $ref: "#/$defs/Cbr"
```

And add the `Cbr` definition to `$defs`:

```yaml
  Cbr:
    type: object
    description: >
      CBR (Case-Based Reasoning) retrieval configuration. Declares how to
      extract features from the case context for similarity-based retrieval
      of past case experiences at routing time.
    required: [ features ]
    unevaluatedProperties: false
    properties:
      features:
        type: object
        description: >
          Map of feature name to JQ expression. Each expression is evaluated
          against the working layer to extract a feature value for CBR retrieval.
        additionalProperties:
          type: string
      weights:
        type: object
        description: Per-feature weight overrides for similarity scoring.
        additionalProperties:
          type: number
      topK:
        type: integer
        minimum: 1
        default: 5
        description: Maximum number of similar cases to retrieve.
      minSimilarity:
        type: number
        minimum: 0
        maximum: 1
        default: 0
        description: Minimum similarity score threshold.
      vectorWeight:
        type: number
        minimum: 0
        maximum: 1
        default: 0.5
        description: Blend factor between feature similarity and vector similarity.
      domain:
        type: string
        minLength: 1
        description: >
          MemoryDomain name for CBR retrieval. Defaults to episodicMemory.domain
          if not specified.
      caseType:
        type: string
        minLength: 1
        description: >
          CBR case type for retrieval. Defaults to the case definition name
          if not specified.
```

- [ ] **Step 3: Regenerate schema classes and implement YAML mapping**

Run: `mvn generate-sources -pl schema -q -f /Users/mdproctor/claude/casehub/engine/pom.xml`

Then update `CaseDefinitionYamlMapper` to map the `cbr` block:

```java
// In the mapping method, after existing field mapping:
if (spec.getCbr() != null) {
    var cbr = spec.getCbr();
    var builder = CbrConfig.builder();
    if (cbr.getFeatures() != null) {
        cbr.getFeatures().forEach((k, v) -> builder.feature(k, v.toString()));
    }
    if (cbr.getWeights() != null) {
        cbr.getWeights().forEach((k, v) -> builder.weight(k, ((Number) v).doubleValue()));
    }
    if (cbr.getTopK() != null) builder.topK(cbr.getTopK());
    if (cbr.getMinSimilarity() != null) builder.minSimilarity(cbr.getMinSimilarity());
    if (cbr.getVectorWeight() != null) builder.vectorWeight(cbr.getVectorWeight());
    if (cbr.getDomain() != null) builder.domain(cbr.getDomain());
    if (cbr.getCaseType() != null) builder.caseType(cbr.getCaseType());
    definition.setCbrConfig(builder.build());
}
```

- [ ] **Step 4: Wire CbrRetrievalService into CaseContextChangedEventHandler**

In `CaseContextChangedEventHandler`:
1. Inject `CbrRetrievalService cbrRetrievalService`
2. In `rules()`, before constructing `PlanExecutionContext`:
   ```java
   return cbrRetrievalService.retrieve(caseDefinition, caseInstance)
       .chain(experiences -> {
           final PlanExecutionContext planCtx = new PlanExecutionContext(
               caseInstance.getUuid(), caseDefinition, caseInstance.getCaseContext(),
               caseInstance.getState(), caseInstance.tenancyId, experiences);
           // ... rest of method, threading experiences through
       });
   ```
3. Add `List<RetrievedExperience> experiences` parameter to `publishByTarget()` and `publishWorkerSchedule()`
4. In `publishWorkerSchedule()`, pass `experiences` to `AgentRoutingContext`

In `DefaultWorkOrchestrator`:
1. Inject `CbrRetrievalService cbrRetrievalService`
2. In `doSubmit()`, retrieve before agent routing and pass to `AgentRoutingContext`

- [ ] **Step 5: Run YAML mapping tests**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest="CaseDefinitionYamlMapperCbrTest" -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: PASS

- [ ] **Step 6: Run full test suite**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: All tests PASS

- [ ] **Step 7: Commit**

```
feat(#478): wire CBR retrieval into CONTEXT_CHANGED flow + YAML mapping

CaseContextChangedEventHandler retrieves experiences once per
evaluation via CbrRetrievalService, threads through PlanExecutionContext
to implementation routing and via parameter to agent routing.
DefaultWorkOrchestrator retrieves for the synchronous path.
CaseDefinitionYamlMapper maps cbr: YAML block to CbrConfig.

Refs #478
```

---

### Task 6: End-to-end integration tests

**Files:**
- Create: `runtime/src/test/java/io/casehub/engine/internal/routing/CbrRoutingIntegrationTest.java`
- Create: `runtime/src/test/java/io/casehub/engine/internal/routing/CbrRoutingFuncDslIntegrationTest.java`

**Interfaces:**
- Consumes: Everything from Tasks 1-5
- Produces: Verified end-to-end CBR retrieval flow for both YAML and Java DSL paths

- [ ] **Step 1: Write YAML-path integration test**

A `@QuarkusTest` with:
- A YAML-defined `CaseHub` with `cbr:` config
- `InMemoryCbrCaseMemoryStore` (from `casehub-neocortex-memory-cbr-inmem`, test dependency) pre-loaded with `PlanCbrCase` entries
- Recording `ImplementationRoutingStrategy` that captures received `experiences`
- Recording `AgentRoutingStrategy` that captures received `experiences`
- Signal the case → CONTEXT_CHANGED fires → verify both strategies received non-empty experiences with correct content

- [ ] **Step 2: Write Java FuncDSL integration test**

Same structure but `CaseHub` subclass uses `CaseDefinition.builder().cbrConfig(CbrConfig.builder().featureExtractor(lambda)...)`.

- [ ] **Step 3: Write no-CBR-config test**

Verify that a case definition without `cbr:` config results in strategies receiving empty experience lists.

- [ ] **Step 4: Add test dependency for InMemoryCbrCaseMemoryStore**

Add to `runtime/pom.xml` test dependencies:
```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-neocortex-memory-cbr-inmem</artifactId>
    <version>${version.io.casehub.neocortex}</version>
    <scope>test</scope>
</dependency>
```

- [ ] **Step 5: Run integration tests**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest="CbrRoutingIntegrationTest,CbrRoutingFuncDslIntegrationTest" -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: All tests PASS

- [ ] **Step 6: Run full test suite**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: All tests PASS

- [ ] **Step 7: Commit**

```
test(#478): end-to-end CBR routing integration tests

YAML path: CaseHub with cbr: config, pre-loaded InMemoryCbrCaseMemoryStore,
recording strategies verify experiences arrive at both implementation
and agent routing.
Java DSL path: same flow with lambda feature extractor.
No-config path: verifies zero overhead when CbrConfig is absent.

Refs #478
```
