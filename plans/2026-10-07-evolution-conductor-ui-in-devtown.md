# Evolution Conductor UI in Devtown — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** casehubio/engine#1180 — Surface evolution conductor UI in devtown as first consumer
**Issue group:** casehubio/engine#1180, casehubio/devtown#TBD (create before Batch 1)

**Goal:** Wire devtown as the first consumer of the evolution conductor — 5 capability areas feeding health scores, a category provider defining devtown-specific improvement categories, filtering SPIs, an API facade with enrichment, a singleton evolution case, and a 9th dashboard tab instantiating the blocks-ui evolution workbench.

**Architecture:** Devtown's `domain` module gets CDI-discovered SPI implementations (capability areas, category provider, proposal source, conflict/deny/regression SPIs). The `app` module gets the API facade (`DevtownEvolutionApi`) delegating to the engine's `EngineEvolutionApi` with view-layer enrichment, a REST resource at `/api/devtown/evolution`, a case bootstrap, and frontend tab wiring. No engine or blocks-ui production code changes.

**Tech Stack:** Java 21, Quarkus 3.39.3, casehub-engine SPIs, casehub-pages, blocks-ui evolution-workbench, Lit/TypeScript

## Global Constraints

- Devtown Quarkus version: 3.39.3 (from devtown CLAUDE.md)
- Engine SPI interfaces are in `io.casehub.api.spi.improvement` (api module)
- Engine model types are in `io.casehub.api.model.stigmergy` (api module)
- Engine `AbstractCapabilityArea` is in `io.casehub.engine.runtime.improvement.area` (runtime-core)
- Devtown capability areas go in `io.casehub.devtown.domain.evolution` (domain module)
- Devtown app-layer evolution code goes in `io.casehub.devtown.app.evolution` (app module)
- CDI discovery: capability areas are discovered via Quarkus build-time indexing — implementations need injectable constructors, no explicit scope annotation required (but `@ApplicationScoped` is good practice)
- All commits reference a devtown issue: `Refs casehubio/devtown#N`
- IntelliJ MCP is available for both engine and devtown in slot 212
- All tests follow `*Test.java` naming (never `*IT.java` — surefire, not failsafe)

---

## Batch 1: Domain SPIs — Capability Areas + Category Provider

After this batch: devtown registers 5 capability areas and a category provider with the engine's conductor. Health scores compute from devtown's PR pipeline data. `mvn test -pl domain` passes.

### Task 1: CiReliabilityCapabilityArea

**Files:**
- Create: `devtown/domain/src/main/java/io/casehub/devtown/domain/evolution/CiReliabilityCapabilityArea.java`
- Test: `devtown/domain/src/test/java/io/casehub/devtown/domain/evolution/CiReliabilityCapabilityAreaTest.java`

**Interfaces:**
- Consumes: `CapabilityArea` (engine api), `AbstractCapabilityArea` (engine runtime-core), `EventLogRepository` (engine common-core), `CaseHubEventType` (engine api)
- Produces: `CiReliabilityCapabilityArea` — CDI bean implementing `CapabilityArea` with `id()` returning `"ci-reliability"`

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.devtown.domain.evolution;

import static org.assertj.core.api.Assertions.assertThat;

import io.casehub.api.model.event.CaseHubEventType;
import io.casehub.api.model.stigmergy.CapabilityAreaAssessment;
import io.casehub.engine.common.spi.EventLogRepository;
import java.util.List;
import java.util.UUID;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.Mockito;

class CiReliabilityCapabilityAreaTest {

  private EventLogRepository eventLog;
  private CiReliabilityCapabilityArea area;
  private final UUID caseId = UUID.randomUUID();
  private final String tenancyId = "test-tenant";

  @BeforeEach
  void setUp() {
    eventLog = Mockito.mock(EventLogRepository.class);
    area = new CiReliabilityCapabilityArea(eventLog);
  }

  @Test
  void id_returns_ci_reliability() {
    assertThat(area.id()).isEqualTo("ci-reliability");
  }

  @Test
  void name_returns_CI_Reliability() {
    assertThat(area.name()).isEqualTo("CI Reliability");
  }

  @Test
  void assess_returns_neutral_when_no_events() {
    Mockito.when(eventLog.findByCaseAndTypes(
        Mockito.eq(caseId), Mockito.anyCollection(), Mockito.eq(tenancyId)))
        .thenReturn(List.of());
    CapabilityAreaAssessment result = area.assess(caseId, tenancyId);
    assertThat(result.healthScore()).isEqualTo(0.5);
  }

  @Test
  void assess_returns_high_score_when_all_builds_pass() {
    var events = TestEventFactory.createEvents(caseId, tenancyId,
        CaseHubEventType.CASE_COMPLETED, 10);
    Mockito.when(eventLog.findByCaseAndTypes(
        Mockito.eq(caseId), Mockito.anyCollection(), Mockito.eq(tenancyId)))
        .thenReturn(events);
    CapabilityAreaAssessment result = area.assess(caseId, tenancyId);
    assertThat(result.healthScore()).isGreaterThan(0.8);
  }

  @Test
  void assess_returns_low_score_when_many_failures() {
    var successes = TestEventFactory.createEvents(caseId, tenancyId,
        CaseHubEventType.CASE_COMPLETED, 2);
    var failures = TestEventFactory.createEvents(caseId, tenancyId,
        CaseHubEventType.CASE_FAULTED, 8);
    var all = new java.util.ArrayList<>(successes);
    all.addAll(failures);
    Mockito.when(eventLog.findByCaseAndTypes(
        Mockito.eq(caseId), Mockito.anyCollection(), Mockito.eq(tenancyId)))
        .thenReturn(all);
    CapabilityAreaAssessment result = area.assess(caseId, tenancyId);
    assertThat(result.healthScore()).isLessThan(0.4);
  }
}
```

- [ ] **Step 2: Create TestEventFactory helper**

```java
package io.casehub.devtown.domain.evolution;

import io.casehub.api.model.event.CaseHubEventType;
import io.casehub.engine.common.spi.EventLogRepository;
import java.time.Instant;
import java.util.ArrayList;
import java.util.List;
import java.util.UUID;
import org.mockito.Mockito;

final class TestEventFactory {

  static List<EventLogRepository.EventEntry> createEvents(
      UUID caseId, String tenancyId, CaseHubEventType type, int count) {
    List<EventLogRepository.EventEntry> events = new ArrayList<>();
    for (int i = 0; i < count; i++) {
      var entry = Mockito.mock(EventLogRepository.EventEntry.class);
      Mockito.when(entry.getEventType()).thenReturn(type);
      Mockito.when(entry.getCaseId()).thenReturn(caseId);
      Mockito.when(entry.getTimestamp()).thenReturn(Instant.now());
      events.add(entry);
    }
    return events;
  }
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl domain -Dtest=CiReliabilityCapabilityAreaTest -f /Users/mdproctor/claude/casehub/slots/212/devtown/pom.xml`
Expected: FAIL — `CiReliabilityCapabilityArea` class not found

- [ ] **Step 4: Implement CiReliabilityCapabilityArea**

```java
package io.casehub.devtown.domain.evolution;

import io.casehub.api.model.event.CaseHubEventType;
import io.casehub.api.model.stigmergy.CapabilityAreaAssessment;
import io.casehub.engine.common.spi.EventLogRepository;
import io.casehub.engine.runtime.improvement.area.AbstractCapabilityArea;
import jakarta.enterprise.context.ApplicationScoped;
import java.util.Collection;
import java.util.List;
import java.util.UUID;

@ApplicationScoped
public class CiReliabilityCapabilityArea extends AbstractCapabilityArea {

  private static final Collection<CaseHubEventType> CI_TYPES =
      List.of(CaseHubEventType.CASE_COMPLETED, CaseHubEventType.CASE_FAULTED);

  private final EventLogRepository eventLog;

  public CiReliabilityCapabilityArea(EventLogRepository eventLog) {
    this.eventLog = eventLog;
  }

  @Override
  public String id() {
    return "ci-reliability";
  }

  @Override
  public String name() {
    return "CI Reliability";
  }

  @Override
  public String description() {
    return "Build pass rate — ratio of successful CI runs to total runs";
  }

  @Override
  public CapabilityAreaAssessment assess(UUID caseId, String tenancyId) {
    var events = eventLog.findByCaseAndTypes(caseId, CI_TYPES, tenancyId);
    if (events.isEmpty()) {
      return neutralAssessment();
    }
    long successes = events.stream()
        .filter(e -> e.getEventType() == CaseHubEventType.CASE_COMPLETED)
        .count();
    double healthScore = (double) successes / events.size();
    return buildAssessment(healthScore);
  }
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl domain -Dtest=CiReliabilityCapabilityAreaTest -f /Users/mdproctor/claude/casehub/slots/212/devtown/pom.xml`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/212/devtown add domain/src/main/java/io/casehub/devtown/domain/evolution/CiReliabilityCapabilityArea.java domain/src/test/java/io/casehub/devtown/domain/evolution/CiReliabilityCapabilityAreaTest.java domain/src/test/java/io/casehub/devtown/domain/evolution/TestEventFactory.java
git -C /Users/mdproctor/claude/casehub/slots/212/devtown commit -m "feat: add CiReliabilityCapabilityArea for evolution conductor Refs casehubio/devtown#N"
```

### Task 2: ReviewQualityCapabilityArea

**Files:**
- Create: `devtown/domain/src/main/java/io/casehub/devtown/domain/evolution/ReviewQualityCapabilityArea.java`
- Test: `devtown/domain/src/test/java/io/casehub/devtown/domain/evolution/ReviewQualityCapabilityAreaTest.java`

**Interfaces:**
- Consumes: `AbstractCapabilityArea`, `EventLogRepository`, `CaseHubEventType`
- Produces: `ReviewQualityCapabilityArea` — CDI bean with `id()` returning `"review-quality"`

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.devtown.domain.evolution;

import static org.assertj.core.api.Assertions.assertThat;

import io.casehub.api.model.event.CaseHubEventType;
import io.casehub.api.model.stigmergy.CapabilityAreaAssessment;
import io.casehub.engine.common.spi.EventLogRepository;
import java.util.List;
import java.util.UUID;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.Mockito;

class ReviewQualityCapabilityAreaTest {

  private EventLogRepository eventLog;
  private ReviewQualityCapabilityArea area;
  private final UUID caseId = UUID.randomUUID();
  private final String tenancyId = "test-tenant";

  @BeforeEach
  void setUp() {
    eventLog = Mockito.mock(EventLogRepository.class);
    area = new ReviewQualityCapabilityArea(eventLog);
  }

  @Test
  void id_returns_review_quality() {
    assertThat(area.id()).isEqualTo("review-quality");
  }

  @Test
  void assess_returns_neutral_when_no_events() {
    Mockito.when(eventLog.findByCaseAndTypes(
        Mockito.eq(caseId), Mockito.anyCollection(), Mockito.eq(tenancyId)))
        .thenReturn(List.of());
    CapabilityAreaAssessment result = area.assess(caseId, tenancyId);
    assertThat(result.healthScore()).isEqualTo(0.5);
  }

  @Test
  void assess_returns_high_score_when_reviews_succeed() {
    var events = TestEventFactory.createEvents(caseId, tenancyId,
        CaseHubEventType.CASE_COMPLETED, 10);
    Mockito.when(eventLog.findByCaseAndTypes(
        Mockito.eq(caseId), Mockito.anyCollection(), Mockito.eq(tenancyId)))
        .thenReturn(events);
    CapabilityAreaAssessment result = area.assess(caseId, tenancyId);
    assertThat(result.healthScore()).isGreaterThan(0.8);
  }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl domain -Dtest=ReviewQualityCapabilityAreaTest -f /Users/mdproctor/claude/casehub/slots/212/devtown/pom.xml`
Expected: FAIL

- [ ] **Step 3: Implement ReviewQualityCapabilityArea**

Follow the same pattern as `CiReliabilityCapabilityArea` but with:
- `id()` → `"review-quality"`
- `name()` → `"Review Quality"`
- `description()` → `"Review accuracy — ratio of accepted findings to total findings produced"`
- `assess()` reads `CASE_COMPLETED` and `CASE_FAULTED` events scoped to review cases

- [ ] **Step 4: Run tests, verify pass, commit**

### Task 3: MergeQueueHealthCapabilityArea, ReviewerTrustCapabilityArea, SlaComplianceCapabilityArea

**Files:**
- Create: `devtown/domain/src/main/java/io/casehub/devtown/domain/evolution/MergeQueueHealthCapabilityArea.java`
- Create: `devtown/domain/src/main/java/io/casehub/devtown/domain/evolution/ReviewerTrustCapabilityArea.java`
- Create: `devtown/domain/src/main/java/io/casehub/devtown/domain/evolution/SlaComplianceCapabilityArea.java`
- Test: `devtown/domain/src/test/java/io/casehub/devtown/domain/evolution/MergeQueueHealthCapabilityAreaTest.java`
- Test: `devtown/domain/src/test/java/io/casehub/devtown/domain/evolution/ReviewerTrustCapabilityAreaTest.java`
- Test: `devtown/domain/src/test/java/io/casehub/devtown/domain/evolution/SlaComplianceCapabilityAreaTest.java`

**Interfaces:**
- Consumes: `AbstractCapabilityArea`, `EventLogRepository`, `CaseHubEventType`
- Produces: Three CDI beans with `id()` returning `"merge-queue-health"`, `"reviewer-trust"`, `"sla-compliance"`

Follow the same TDD pattern as Tasks 1-2. Each area:

| Area | id | CaseHubEventType scope |
|------|----|----------------------|
| MergeQueueHealth | `merge-queue-health` | CASE_COMPLETED, CASE_FAULTED (merge-related cases) |
| ReviewerTrust | `reviewer-trust` | CASE_COMPLETED, CASE_FAULTED (routing outcome events) |
| SlaCompliance | `sla-compliance` | CASE_COMPLETED, CASE_FAULTED (SLA-tracked cases) |

- [ ] **Step 1: Write failing tests for all three** (one test class each)
- [ ] **Step 2: Implement all three**
- [ ] **Step 3: Run all tests, verify pass**
- [ ] **Step 4: Commit**

### Task 4: DevtownCategoryProvider

**Files:**
- Create: `devtown/domain/src/main/java/io/casehub/devtown/domain/evolution/DevtownCategoryProvider.java`
- Test: `devtown/domain/src/test/java/io/casehub/devtown/domain/evolution/DevtownCategoryProviderTest.java`

**Interfaces:**
- Consumes: `ImprovementCategoryProvider` (engine api), `CategoryDescriptor`, `StageDescriptor` (engine api model)
- Produces: `DevtownCategoryProvider` — CDI bean contributing 5 categories and 5 stages for domain `"devtown"`

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.devtown.domain.evolution;

import static org.assertj.core.api.Assertions.assertThat;

import io.casehub.api.model.stigmergy.CategoryDescriptor;
import io.casehub.api.model.stigmergy.StageDescriptor;
import java.util.List;
import org.junit.jupiter.api.Test;

class DevtownCategoryProviderTest {

  private final DevtownCategoryProvider provider = new DevtownCategoryProvider();

  @Test
  void domainId_returns_devtown() {
    assertThat(provider.domainId()).isEqualTo("devtown");
  }

  @Test
  void provides_five_categories() {
    List<CategoryDescriptor> cats = provider.categories();
    assertThat(cats).hasSize(5);
    assertThat(cats).extracting(CategoryDescriptor::id)
        .containsExactlyInAnyOrder(
            "reviewer-calibration", "routing-adjustment",
            "sla-tuning", "gate-tightening", "queue-optimization");
    assertThat(cats).allMatch(c -> "devtown".equals(c.domainId()));
  }

  @Test
  void provides_five_stages_in_order() {
    List<StageDescriptor> stages = provider.stages();
    assertThat(stages).hasSize(5);
    assertThat(stages).extracting(StageDescriptor::id)
        .containsExactly("analyze", "propose", "review", "apply", "observe");
    assertThat(stages).extracting(StageDescriptor::ordinal)
        .containsExactly(0, 1, 2, 3, 4);
  }

  @Test
  void propose_and_review_are_gate_checkpoints() {
    List<StageDescriptor> stages = provider.stages();
    assertThat(stages.stream().filter(StageDescriptor::gateCheckpoint)
        .map(StageDescriptor::id).toList())
        .containsExactlyInAnyOrder("propose", "review");
  }
}
```

- [ ] **Step 2: Run test to verify it fails**
- [ ] **Step 3: Implement DevtownCategoryProvider** (as shown in the design spec §2)
- [ ] **Step 4: Run tests, verify pass, commit**

---

## Batch 2: Filtering SPIs — Proposal Source, Conflict, Deny, Regression

After this batch: devtown has a complete filtering pipeline — proposals generated from health scores, conflicts detected per-category, structural invariants protected, regression evaluated. `mvn test -pl domain` passes.

### Task 5: DevtownConflictStrategy

**Files:**
- Create: `devtown/domain/src/main/java/io/casehub/devtown/domain/evolution/DevtownConflictStrategy.java`
- Test: `devtown/domain/src/test/java/io/casehub/devtown/domain/evolution/DevtownConflictStrategyTest.java`

**Interfaces:**
- Consumes: `ConflictStrategy` (engine api), `ImprovementRequest` (engine api model)
- Produces: `DevtownConflictStrategy` — CDI bean with `domainId()` returning `"devtown"`, same-category conflict detection

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.devtown.domain.evolution;

import static org.assertj.core.api.Assertions.assertThat;

import io.casehub.api.model.stigmergy.ImprovementRequest;
import io.casehub.api.spi.improvement.ConflictStrategy.ConflictResult;
import java.util.Map;
import java.util.UUID;
import org.junit.jupiter.api.Test;

class DevtownConflictStrategyTest {

  private final DevtownConflictStrategy strategy = new DevtownConflictStrategy();

  @Test
  void domainId_returns_devtown() {
    assertThat(strategy.domainId()).isEqualTo("devtown");
  }

  @Test
  void clear_when_no_active_improvements() {
    var request = new ImprovementRequest(
        "recalibrate", "reviewer-calibration", "reviewer:agent-sec",
        "devtown", 3, Map.of());
    var result = strategy.check(request, Map.of(), 5);
    assertThat(result).isInstanceOf(ConflictResult.Clear.class);
  }

  @Test
  void conflicting_when_same_category_active() {
    var active = new ImprovementRequest(
        "recalibrate-existing", "reviewer-calibration", "reviewer:agent-perf",
        "devtown", 2, Map.of());
    var request = new ImprovementRequest(
        "recalibrate-new", "reviewer-calibration", "reviewer:agent-sec",
        "devtown", 3, Map.of());
    UUID activeId = UUID.randomUUID();
    var result = strategy.check(request, Map.of(activeId, active), 5);
    assertThat(result).isInstanceOf(ConflictResult.Conflicting.class);
  }

  @Test
  void clear_when_different_category_active() {
    var active = new ImprovementRequest(
        "tune-sla", "sla-tuning", "sla:security-review",
        "devtown", 2, Map.of());
    var request = new ImprovementRequest(
        "recalibrate", "reviewer-calibration", "reviewer:agent-sec",
        "devtown", 3, Map.of());
    var result = strategy.check(request, Map.of(UUID.randomUUID(), active), 5);
    assertThat(result).isInstanceOf(ConflictResult.Clear.class);
  }
}
```

- [ ] **Step 2: Run test to verify it fails**
- [ ] **Step 3: Implement DevtownConflictStrategy** (as shown in design spec §7)
- [ ] **Step 4: Run tests, verify pass, commit**

### Task 6: DevtownDenyPatternProvider + DevtownRegressionEvaluator

**Files:**
- Create: `devtown/domain/src/main/java/io/casehub/devtown/domain/evolution/DevtownDenyPatternProvider.java`
- Create: `devtown/domain/src/main/java/io/casehub/devtown/domain/evolution/DevtownRegressionEvaluator.java`
- Test: `devtown/domain/src/test/java/io/casehub/devtown/domain/evolution/DevtownDenyPatternProviderTest.java`
- Test: `devtown/domain/src/test/java/io/casehub/devtown/domain/evolution/DevtownRegressionEvaluatorTest.java`

**Interfaces:**
- Consumes: `DenyPatternProvider`, `RegressionEvaluator` (engine api), `ImprovementRequest`, `HealthScoreSnapshot`, `RegressionVerdict` (engine api model), `ImprovementConfig` (engine api model)
- Produces: `DevtownDenyPatternProvider` with `domainId()` `"devtown"`, `DevtownRegressionEvaluator` with `evaluatorId()` `"devtown-health-delta"` and `domainId()` `"devtown"`

- [ ] **Step 1: Write failing tests for both**
- [ ] **Step 2: Implement both** (as shown in design spec §7 and §8)
- [ ] **Step 3: Run tests, verify pass, commit**

### Task 7: DevtownProposalSource

**Files:**
- Create: `devtown/domain/src/main/java/io/casehub/devtown/domain/evolution/DevtownProposalSource.java`
- Test: `devtown/domain/src/test/java/io/casehub/devtown/domain/evolution/DevtownProposalSourceTest.java`

**Interfaces:**
- Consumes: `ImprovementProposalSource` (engine api), `ImprovementRequest`, `ImprovementConfig` (engine api model), `CiReliabilityCapabilityArea`, `ReviewQualityCapabilityArea`, `MergeQueueHealthCapabilityArea`, `ReviewerTrustCapabilityArea`, `SlaComplianceCapabilityArea` (from Task 1-3)
- Produces: `DevtownProposalSource` — CDI bean with `sourceId()` `"devtown-assessment"` and `domainId()` `"devtown"`. Generates proposals when capability area health scores drop below thresholds.

- [ ] **Step 1: Write the failing test**

Test scenarios: no proposals when all areas healthy (score > 0.6), reviewer-calibration proposal when reviewer trust drops below 0.6, sla-tuning proposal when SLA compliance drops, multiple proposals when multiple areas degrade.

- [ ] **Step 2: Run test to verify it fails**
- [ ] **Step 3: Implement DevtownProposalSource** (as shown in design spec §6)
- [ ] **Step 4: Run tests, verify pass, commit**

---

## Batch 3: API + Bootstrap — Backend Wiring

After this batch: devtown has a REST API at `/api/devtown/evolution` serving enriched evolution state. A singleton evolution case auto-bootstraps at startup. `mvn test -pl app` passes.

### Task 8: Case Template + Bootstrap + Resolver

**Files:**
- Create: `devtown/app/src/main/resources/templates/devtown-evolution.yaml`
- Create: `devtown/app/src/main/java/io/casehub/devtown/app/evolution/DevtownEvolutionBootstrap.java`
- Create: `devtown/app/src/main/java/io/casehub/devtown/app/evolution/DevtownEvolutionCaseResolver.java`
- Test: `devtown/app/src/test/java/io/casehub/devtown/app/evolution/DevtownEvolutionBootstrapTest.java`
- Test: `devtown/app/src/test/java/io/casehub/devtown/app/evolution/DevtownEvolutionCaseResolverTest.java`

**Interfaces:**
- Consumes: `CaseInstanceRepository` (engine common-core SPI), `CaseInstance` (engine api model), `StartupEvent` (Quarkus)
- Produces: `DevtownEvolutionBootstrap` — creates singleton case on startup. `DevtownEvolutionCaseResolver` — resolves the singleton by template ID.

- [ ] **Step 1: Create case template YAML** (as shown in design spec §5)
- [ ] **Step 2: Write failing tests for bootstrap and resolver**
- [ ] **Step 3: Implement bootstrap and resolver** (as shown in design spec §5)
- [ ] **Step 4: Run tests, verify pass, commit**

### Task 9: DevtownEvolutionApi Facade + REST Resource

**Files:**
- Create: `devtown/app/src/main/java/io/casehub/devtown/app/evolution/DevtownEvolutionApi.java`
- Create: `devtown/app/src/main/java/io/casehub/devtown/app/evolution/DevtownEvolutionEnricher.java`
- Create: `devtown/app/src/main/java/io/casehub/devtown/app/evolution/DevtownEvolutionStateSnapshot.java`
- Create: `devtown/app/src/main/java/io/casehub/devtown/app/evolution/DevtownImprovementStreamView.java`
- Create: `devtown/app/src/main/java/io/casehub/devtown/app/evolution/DevtownEvolutionResource.java`
- Create: `devtown/app/src/main/java/io/casehub/devtown/app/evolution/GateResolutionRequest.java`
- Test: `devtown/app/src/test/java/io/casehub/devtown/app/evolution/DevtownEvolutionApiTest.java`
- Test: `devtown/app/src/test/java/io/casehub/devtown/app/evolution/DevtownEvolutionEnricherTest.java`

**Interfaces:**
- Consumes: `EngineEvolutionApi` (engine api SPI), `DevtownEvolutionCaseResolver` (from Task 8), `GovernanceQueryService` (devtown app)
- Produces: REST resource at `/api/devtown/evolution` with endpoints: `GET /state`, `GET /inbox`, `POST /inbox/{entryId}/resolve`, `GET /deny-patterns`, `POST /deny-patterns`, `DELETE /deny-patterns/{pattern}`, `GET /watch-patterns`, `POST /watch-patterns`, `DELETE /watch-patterns/{patternId}`, `GET /gate-policy`, `PUT /gate-policy`, `GET /streams`, `POST /category/{category}/pause`, `POST /category/{category}/unpause`, `POST /circuit-breaker/reset`

- [ ] **Step 1: Create view records** (DevtownEvolutionStateSnapshot, DevtownImprovementStreamView, GateResolutionRequest)
- [ ] **Step 2: Write failing tests for enricher** (stream targets enriched with devtown context)
- [ ] **Step 3: Implement enricher**
- [ ] **Step 4: Write failing tests for API facade** (delegates to engine API, enriches queries)
- [ ] **Step 5: Implement API facade**
- [ ] **Step 6: Implement REST resource** (JAX-RS endpoints delegating to facade)
- [ ] **Step 7: Run all tests, verify pass, commit**

---

## Batch 4: Frontend — Evolution Tab

After this batch: devtown's dashboard has a 9th "Evolution" tab rendering the blocks-ui evolution workbench against the `/api/devtown/evolution` endpoint. The full feedback loop is visible.

### Task 10: Frontend Evolution Tab Integration

**Files:**
- Create: `devtown/app/src/main/webui/src/views/evolution.ts`
- Modify: `devtown/app/src/main/webui/src/index.ts`

**Interfaces:**
- Consumes: `@casehubio/blocks-ui-evolution-workbench` (blocks-ui package), `registerPanel`, `hostPanel` (casehub-pages)
- Produces: "Evolution" tab in devtown dashboard

- [ ] **Step 1: Create evolution view**

```typescript
// devtown/app/src/main/webui/src/views/evolution.ts
import { hostPanel } from "@casehubio/pages-ui";

export const evolutionView = hostPanel("evolution-workbench", {
  endpoint: "/api/devtown/evolution",
});
```

- [ ] **Step 2: Register the panel and add the tab in index.ts**

Add to imports:
```typescript
import "@casehubio/blocks-ui-evolution-workbench";
import { evolutionView } from "./views/evolution";
```

Add panel registration:
```typescript
registerPanel("evolution-workbench", "blocks-evolution-workbench");
```

Add tab between "System" and "Definitions":
```typescript
["Evolution", evolutionView],
```

- [ ] **Step 3: Build frontend and verify**

Run: `yarn --cwd /Users/mdproctor/claude/casehub/slots/212/devtown/app/src/main/webui build`
Expected: Build succeeds with no TypeScript errors

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/212/devtown add app/src/main/webui/src/views/evolution.ts app/src/main/webui/src/index.ts
git -C /Users/mdproctor/claude/casehub/slots/212/devtown commit -m "feat: add Evolution tab to devtown dashboard Refs casehubio/devtown#N"
```

---

## References

- [2026-10-07-evolution-conductor-ui-in-devtown-design.md] — design spec this plan implements
- [engine/api/src/main/java/io/casehub/api/spi/improvement/CapabilityArea.java:21] — SPI interface
- [engine/runtime-core/src/main/java/io/casehub/engine/runtime/improvement/area/AbstractCapabilityArea.java:23] — base class
- [engine/runtime-core/src/main/java/io/casehub/engine/runtime/improvement/area/StabilityCapabilityArea.java:25] — reference implementation
- [engine/runtime-core/src/main/java/io/casehub/engine/runtime/improvement/CapabilityAreaBootstrap.java:20] — CDI discovery pattern
- [engine/runtime-core/src/main/java/io/casehub/engine/runtime/improvement/EvolutionBootstrap.java:24] — SPI bootstrap pattern
- [devtown/app/src/main/webui/src/index.ts] — dashboard tab registration
- [blocks-ui/components/evolution-workbench/src/evolution-workbench.ts] — workbench component
- [blocks-ui/components/evolution-config/src/types.ts] — TypeScript types
- [GitHub casehubio/engine#1180] — focal issue
- [GitHub casehubio/engine#1149] — parent epic
- Decisions D1–D7 in `decisions.md`
