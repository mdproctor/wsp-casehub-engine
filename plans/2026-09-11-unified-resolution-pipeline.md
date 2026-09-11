# Unified Resolution Pipeline Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #1081 — Unified resolution pipeline — CBR, documents, human-in-the-loop, and retrieval feedback
**Issue group:** #1081

**Goal:** Unify the resolution pipeline across automated, human, and hybrid paths: mixed retrieval (plan traces + documents), candidate presentation via JudgmentTarget, three-layer retrieval feedback, and document ingestion.

**Architecture:** Distributed composition across existing engine components. No new module. Each piece lands in its natural home: CbrRetrievalService (retrieval), RetrievalFeedbackObserver (feedback), JudgmentTarget enrichment (candidate presentation), ResolutionIngestionService (ingestion).

**Tech Stack:** Java 21, Quarkus 3.32.2, casehub-neocortex memory-api (ResolutionGuide, CbrCaseMemoryStore, CbrRetrievalTracker)

## Global Constraints

- Pre-release platform — breaking changes are acceptable when the design is right
- All new types use records (immutable)
- `@DefaultBean @ApplicationScoped` for no-op SPI defaults per protocol PP-20260514
- `Instance<T>` with `isResolvable()` guard for optional dependencies
- Backward-compatible constructors on RetrievedExperience
- No database migrations (Hibernate drop-and-create)
- `quarkus.arc.selected-alternatives` activation for in-memory test stores
- IntelliJ MCP mandatory for all code navigation and editing
- `RoutingOutcome` enum values: SUCCESS, FAILURE, GATE_REJECTED, GATE_EXPIRED, DECLINED, CANCELLED, OBSOLETE

## Neocortex Prerequisites

These neocortex changes must land before Batches 2-5. Create issues in casehubio/neocortex:

1. **GuidanceStep record + ResolutionGuide extension** — Add `GuidanceStep` record, `features` field, `steps` field, `withFeatures()`, and `withSteps()` to `ResolutionGuide` in memory-api
2. **CbrRetrievalTracker.feedback()** — Add `feedback(String caseId, String tenancyId, List<CbrRetrievalFeedback> entries)` method and `CbrRetrievalFeedback` record to `CbrRetrievalTracker` in memory-api

---

## Batch 1: Foundation Types [engine API types + naming cleanup]

### Task 1: CBR naming cleanup + new API types

**Files:**
- Modify: `runtime/src/main/java/io/casehub/engine/internal/routing/CbrRetrievalService.java` (BUILT_IN_TYPES imports)
- Modify: `runtime/src/main/java/io/casehub/engine/internal/memory/CbrCaseRetainObserver.java` (PlanCbrCase → ResolvedCase)
- Create: `api/src/main/java/io/casehub/api/spi/routing/ResolutionSourceType.java`
- Create: `api/src/main/java/io/casehub/api/spi/routing/DocumentStep.java`
- Create: `api/src/main/java/io/casehub/api/model/ResolutionSelection.java`
- Test: `api/src/test/java/io/casehub/api/spi/routing/DocumentStepTest.java`
- Test: `api/src/test/java/io/casehub/api/model/ResolutionSelectionTest.java`

**Interfaces:**
- Produces: `ResolutionSourceType` enum (PLAN_TRACE, RESOLUTION_GUIDE) — used by Task 2 (RetrievedExperience extension)
- Produces: `DocumentStep` record (description, preconditions, expectedOutcome, automationHint) — used by Task 2
- Produces: `ResolutionSelection` record (selectedCaseId, sourceType, rationale) — used by Task 7

- [ ] **Step 1: Write test for DocumentStep**

```java
package io.casehub.api.spi.routing;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.*;

class DocumentStepTest {
    @Test
    void constructsWithAllFields() {
        var step = new DocumentStep("Isolate mailbox", "Access to Exchange admin", "Mailbox isolated", "exchange:disable-mailbox");
        assertThat(step.description()).isEqualTo("Isolate mailbox");
        assertThat(step.preconditions()).isEqualTo("Access to Exchange admin");
        assertThat(step.expectedOutcome()).isEqualTo("Mailbox isolated");
        assertThat(step.automationHint()).isEqualTo("exchange:disable-mailbox");
    }

    @Test
    void nullableFieldsDefaultToNull() {
        var step = new DocumentStep("Reset credentials", null, null, null);
        assertThat(step.description()).isEqualTo("Reset credentials");
        assertThat(step.preconditions()).isNull();
    }
}
```

- [ ] **Step 2: Run test — verify it fails (class not defined)**

Run: `mvn test -pl api -Dtest=DocumentStepTest -Dsurefire.failIfNoSpecifiedTests=false -q`
Expected: compilation failure

- [ ] **Step 3: Create ResolutionSourceType enum**

```java
package io.casehub.api.spi.routing;

public enum ResolutionSourceType {
    PLAN_TRACE,
    RESOLUTION_GUIDE
}
```

- [ ] **Step 4: Create DocumentStep record**

```java
package io.casehub.api.spi.routing;

import jakarta.annotation.Nullable;

public record DocumentStep(
    String description,
    @Nullable String preconditions,
    @Nullable String expectedOutcome,
    @Nullable String automationHint
) {}
```

- [ ] **Step 5: Create ResolutionSelection record**

```java
package io.casehub.api.model;

import io.casehub.api.spi.routing.ResolutionSourceType;
import jakarta.annotation.Nullable;

public record ResolutionSelection(
    String selectedCaseId,
    ResolutionSourceType sourceType,
    @Nullable String rationale
) {
    public ResolutionSelection {
        java.util.Objects.requireNonNull(selectedCaseId, "selectedCaseId must not be null");
        java.util.Objects.requireNonNull(sourceType, "sourceType must not be null");
    }
}
```

- [ ] **Step 6: Write test for ResolutionSelection**

```java
package io.casehub.api.model;

import io.casehub.api.spi.routing.ResolutionSourceType;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.*;

class ResolutionSelectionTest {
    @Test
    void constructsWithRequiredFields() {
        var sel = new ResolutionSelection("case-123", ResolutionSourceType.PLAN_TRACE, null);
        assertThat(sel.selectedCaseId()).isEqualTo("case-123");
        assertThat(sel.sourceType()).isEqualTo(ResolutionSourceType.PLAN_TRACE);
        assertThat(sel.rationale()).isNull();
    }

    @Test
    void rejectsNullCaseId() {
        assertThatThrownBy(() -> new ResolutionSelection(null, ResolutionSourceType.PLAN_TRACE, null))
            .isInstanceOf(NullPointerException.class);
    }
}
```

- [ ] **Step 7: Run tests — verify they pass**

Run: `mvn test -pl api -Dtest="DocumentStepTest,ResolutionSelectionTest" -q`
Expected: PASS

- [ ] **Step 8: Update CBR naming in CbrRetrievalService**

Use `ide_search_text` to find all `PlanCbrCase` and `TextualCbrCase` references in the engine. Update imports:
- `import io.casehub.neocortex.memory.cbr.PlanCbrCase` → `import io.casehub.neocortex.memory.cbr.ResolvedCase`
- `import io.casehub.neocortex.memory.cbr.TextualCbrCase` → `import io.casehub.neocortex.memory.cbr.ResolutionGuide`
- Update `BUILT_IN_TYPES` map entries: `"plan", PlanCbrCase.class` → `"plan", ResolvedCase.class` and `"textual", TextualCbrCase.class` → `"textual", ResolutionGuide.class`
- Update all references in CbrCaseRetainObserver

- [ ] **Step 9: Run full test suite to verify naming cleanup**

Run: `mvn test -pl runtime -q`
Expected: PASS (naming cleanup is mechanical)

- [ ] **Step 10: Commit**

```bash
git add api/src runtime/src
git commit -m "$(cat <<'EOF'
feat(#1081): foundation types — ResolutionSourceType, DocumentStep, ResolutionSelection, CBR naming cleanup

Add engine API types for the unified resolution pipeline:
- ResolutionSourceType enum (PLAN_TRACE, RESOLUTION_GUIDE)
- DocumentStep record (mapped from neocortex GuidanceStep)
- ResolutionSelection record (judgment resolution type)
- Update PlanCbrCase→ResolvedCase, TextualCbrCase→ResolutionGuide imports

Refs #1081

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

### Task 2: RetrievedExperience extension

**Files:**
- Modify: `api/src/main/java/io/casehub/api/spi/routing/RetrievedExperience.java`
- Test: `api/src/test/java/io/casehub/api/spi/routing/RetrievedExperienceTest.java` (extend existing)

**Interfaces:**
- Consumes: `ResolutionSourceType`, `DocumentStep` (from Task 1)
- Produces: Extended `RetrievedExperience` with `sourceType()`, `documentContent()`, `documentSteps()` — used by Tasks 3-8

- [ ] **Step 1: Read current RetrievedExperience to understand field count**

Use `ide_file_structure` on `RetrievedExperience.java` to see current record components.

- [ ] **Step 2: Write test for new fields**

```java
@Test
void documentFieldsPopulatedForResolutionGuide() {
    var steps = List.of(new DocumentStep("Isolate", null, null, null));
    var exp = new RetrievedExperience(
        /* existing fields... */
        ResolutionSourceType.RESOLUTION_GUIDE, "Full prose solution", steps);
    assertThat(exp.sourceType()).isEqualTo(ResolutionSourceType.RESOLUTION_GUIDE);
    assertThat(exp.documentContent()).isEqualTo("Full prose solution");
    assertThat(exp.documentSteps()).containsExactly(steps.get(0));
}

@Test
void backwardCompatConstructorDefaultsToPlanTrace() {
    var exp = /* use existing N-arg constructor */;
    assertThat(exp.sourceType()).isEqualTo(ResolutionSourceType.PLAN_TRACE);
    assertThat(exp.documentContent()).isNull();
    assertThat(exp.documentSteps()).isNull();
}
```

- [ ] **Step 3: Run test — verify it fails**

- [ ] **Step 4: Add three fields to RetrievedExperience record**

Add `sourceType` (ResolutionSourceType, default PLAN_TRACE), `documentContent` (@Nullable String), `documentSteps` (@Nullable List<DocumentStep>). Add backward-compatible constructor that passes PLAN_TRACE, null, null for these fields.

- [ ] **Step 5: Run tests — verify they pass**

Run: `mvn test -pl api -q`
Expected: PASS

- [ ] **Step 6: Verify no downstream compilation breaks**

Run: `mvn compile -pl runtime,planning -q`
Expected: PASS (backward-compat constructor ensures no breaks)

- [ ] **Step 7: Commit**

```bash
git add api/src
git commit -m "$(cat <<'EOF'
feat(#1081): extend RetrievedExperience with sourceType, documentContent, documentSteps

Three new fields for mixed retrieval results:
- sourceType (ResolutionSourceType, defaults to PLAN_TRACE)
- documentContent (nullable String, prose from ResolutionGuide)
- documentSteps (nullable List<DocumentStep>, structured steps)
Backward-compatible constructor passes defaults for all three.

Refs #1081

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Batch 2: Mixed Retrieval [CbrRetrievalService cross-type fix + mapping]

**Neocortex prerequisite:** ResolutionGuide must have `features` field and `withFeatures()` method.

### Task 3: CbrRetrievalService cross-type fix + mixed mapping

**Files:**
- Modify: `runtime/src/main/java/io/casehub/engine/internal/routing/CbrRetrievalService.java`
- Test: `runtime/src/test/java/io/casehub/engine/internal/routing/CbrRetrievalServiceTest.java` (extend)

**Interfaces:**
- Consumes: `RetrievedExperience` extension (from Task 2), `ResolutionSourceType`, `DocumentStep`
- Produces: Mixed `List<RetrievedExperience>` with both PLAN_TRACE and RESOLUTION_GUIDE entries — consumed by all downstream tasks

- [ ] **Step 1: Read CbrRetrievalService.retrieve() to understand class parameter issue**

Use `ide_file_structure` on CbrRetrievalService.java. Locate `retrieve()`, `mapScoredCase()`, and `retrieveForSelection()`.

- [ ] **Step 2: Write test for cross-type retrieval returning mixed results**

```java
@Test
void crossTypeRetrievalReturnsMixedResults() {
    // Store a ResolvedCase and a ResolutionGuide in the same domain
    var planCase = new ResolvedCase("plan problem", "plan solution", "WIN", Confidence.unknown(0.9),
        Map.of("severity", FeatureValue.string("HIGH")), List.of(), null, null);
    var guideCase = new ResolutionGuide("guide problem", "guide solution", null, null,
        Map.of("severity", FeatureValue.string("HIGH")), List.of(), null, null);

    cbrStore.store(planCase, "plan", "entity-1", "soc-domain", "tenant-1",
        UUID.randomUUID().toString(), Path.of("test"));
    cbrStore.store(guideCase, "textual", "entity-2", "soc-domain", "tenant-1",
        UUID.randomUUID().toString(), Path.of("test"));

    // Build definition with crossType: true
    var definition = buildDefinitionWithCrossTypeCbr("soc-domain");
    var instance = buildCaseInstance("tenant-1");

    var results = cbrRetrievalService.retrieve(definition, instance);

    assertThat(results).hasSize(2);
    assertThat(results.stream().map(RetrievedExperience::sourceType).toList())
        .containsExactlyInAnyOrder(ResolutionSourceType.PLAN_TRACE, ResolutionSourceType.RESOLUTION_GUIDE);
}

@Test
void resolutionGuideResultCarriesDocumentFields() {
    // Store a ResolutionGuide with steps
    var steps = List.of(new GuidanceStep("Step 1", null, null, null));
    var guide = new ResolutionGuide("problem", "prose solution", null, null,
        Map.of(), steps, null, null);

    cbrStore.store(guide, "textual", "entity-1", "soc-domain", "tenant-1",
        UUID.randomUUID().toString(), Path.of("test"));

    var definition = buildDefinitionWithCrossTypeCbr("soc-domain");
    var instance = buildCaseInstance("tenant-1");

    var results = cbrRetrievalService.retrieve(definition, instance);
    var docResult = results.stream()
        .filter(r -> r.sourceType() == ResolutionSourceType.RESOLUTION_GUIDE)
        .findFirst().orElseThrow();

    assertThat(docResult.documentContent()).isEqualTo("prose solution");
    assertThat(docResult.documentSteps()).hasSize(1);
    assertThat(docResult.documentSteps().get(0).description()).isEqualTo("Step 1");
}
```

- [ ] **Step 3: Run tests — verify they fail**

- [ ] **Step 4: Fix cross-type class parameter**

In `retrieve()`, when `config.crossType()` is true, pass `CbrCase.class` instead of the `cbrType`-resolved class:

```java
if (config.crossType()) {
    return retrieveInternal(definition, instance, CbrCase.class);
}
```

Same fix for `retrieveForSelection()`.

- [ ] **Step 5: Extend mapScoredCase() for ResolutionGuide**

Add a branch for `ResolutionGuide` alongside the existing `ResolvedCase` branch:

```java
if (scored.cbrCase() instanceof ResolutionGuide guide) {
    var docSteps = guide.steps() != null
        ? guide.steps().stream()
            .map(s -> new DocumentStep(s.description(), s.preconditions(), s.expectedOutcome(), s.automationHint()))
            .toList()
        : null;
    return new RetrievedExperience(
        /* existing scoring fields */,
        ResolutionSourceType.RESOLUTION_GUIDE, guide.solution(), docSteps);
}
```

- [ ] **Step 6: Run tests — verify they pass**

Run: `mvn test -pl runtime -Dtest=CbrRetrievalServiceTest -q`
Expected: PASS

- [ ] **Step 7: Run full runtime test suite**

Run: `mvn test -pl runtime -q`
Expected: PASS

- [ ] **Step 8: Commit**

```bash
git add runtime/src
git commit -m "$(cat <<'EOF'
feat(#1081): CbrRetrievalService cross-type fix + mixed retrieval mapping

Fix cross-type retrieval to pass CbrCase.class (not cbrType-resolved class)
when crossType=true. Add ResolutionGuide → RetrievedExperience mapping with
sourceType=RESOLUTION_GUIDE, documentContent from solution(), documentSteps
mapped from GuidanceStep.

Refs #1081

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Batch 3: Document Ingestion [CorpusSourceAdapter SPI + ingestion service]

**Neocortex prerequisite:** GuidanceStep on ResolutionGuide.

### Task 4: CorpusSourceAdapter SPI + ResolutionIngestionService

**Files:**
- Create: `api/src/main/java/io/casehub/api/spi/CorpusSourceAdapter.java`
- Create: `api/src/main/java/io/casehub/api/spi/ResolutionGuideInput.java`
- Create: `api/src/main/java/io/casehub/api/spi/GuidanceStepInput.java`
- Create: `api/src/main/java/io/casehub/api/spi/CorpusChangeEvent.java`
- Create: `runtime/src/main/java/io/casehub/engine/internal/worker/NoOpCorpusSourceAdapter.java`
- Create: `runtime/src/main/java/io/casehub/engine/internal/routing/ResolutionIngestionService.java`
- Test: `runtime/src/test/java/io/casehub/engine/internal/routing/ResolutionIngestionServiceTest.java`

**Interfaces:**
- Consumes: `CbrCaseMemoryStore` (neocortex), `ResolutionGuide` (neocortex)
- Produces: `CorpusSourceAdapter` SPI — consumers implement for their source
- Produces: `ResolutionIngestionService` — bridges adapter to CBR store

- [ ] **Step 1: Write test for ResolutionIngestionService**

Test should verify: discover() calls store(), idempotent re-ingestion via supersede, error isolation per document, no-op when no adapter.

```java
@Test
void ingestsDiscoveredDocuments() {
    var adapter = new TestAdapter(List.of(
        new ResolutionGuideInput("doc-1", "Problem A", "Solution A", null,
            Map.of("cat", FeatureValue.string("phishing")), "soc-domain", null)));

    var service = new ResolutionIngestionService(adapter, cbrStore);
    service.ingest("tenant-1");

    var results = cbrStore.retrieveSimilar(query("soc-domain", "tenant-1"), ResolutionGuide.class);
    assertThat(results).hasSize(1);
    assertThat(results.get(0).cbrCase().problem()).isEqualTo("Problem A");
}

@Test
void idempotentOnRestart() {
    var adapter = new TestAdapter(List.of(
        new ResolutionGuideInput("doc-1", "Problem A", "Solution A", null,
            Map.of(), "soc-domain", null)));

    var service = new ResolutionIngestionService(adapter, cbrStore);
    service.ingest("tenant-1");
    service.ingest("tenant-1"); // second ingestion

    var results = cbrStore.retrieveSimilar(query("soc-domain", "tenant-1"), ResolutionGuide.class);
    assertThat(results).hasSize(1); // not duplicated
}

@Test
void errorIsolationPerDocument() {
    // First doc will fail (null problem), second should still succeed
    var adapter = new TestAdapter(List.of(
        new ResolutionGuideInput("bad-1", null, "sol", null, Map.of(), "d", null),
        new ResolutionGuideInput("good-1", "Problem", "Sol", null, Map.of(), "d", null)));

    var service = new ResolutionIngestionService(adapter, cbrStore);
    service.ingest("tenant-1");

    var results = cbrStore.retrieveSimilar(query("d", "tenant-1"), ResolutionGuide.class);
    assertThat(results).hasSize(1);
}
```

- [ ] **Step 2: Run tests — verify they fail**

- [ ] **Step 3: Create SPI types in engine-api**

Create `CorpusSourceAdapter`, `ResolutionGuideInput`, `GuidanceStepInput`, `CorpusChangeEvent` (sealed interface) as specified in the spec §4.3.

- [ ] **Step 4: Create NoOpCorpusSourceAdapter**

```java
@DefaultBean
@ApplicationScoped
public class NoOpCorpusSourceAdapter implements CorpusSourceAdapter {
    @Override public String id() { return "noop"; }
    @Override public List<ResolutionGuideInput> discover(String tenancyId) { return List.of(); }
}
```

- [ ] **Step 5: Implement ResolutionIngestionService**

```java
@ApplicationScoped
public class ResolutionIngestionService {
    @Inject Instance<CorpusSourceAdapter> adapter;
    @Inject Instance<CbrCaseMemoryStore> cbrStore;

    public void ingest(String tenancyId) {
        if (!adapter.isResolvable() || !cbrStore.isResolvable()) return;
        for (var input : adapter.get().discover(tenancyId)) {
            try {
                var caseId = UUID.nameUUIDFromBytes(input.documentId().getBytes(UTF_8)).toString();
                var guide = new ResolutionGuide(input.problem(), input.solution(),
                    null, null, input.features(), mapSteps(input.steps()), null, null);
                cbrStore.get().supersedeAll(List.of(caseId), tenancyId, "corpus re-ingestion");
                cbrStore.get().store(guide, "textual", input.documentId(),
                    input.domain(), tenancyId, caseId, Path.of("corpus"));
            } catch (Exception e) {
                log.warn("Ingestion failed for document {}: {}", input.documentId(), e.getMessage());
            }
        }
    }
}
```

- [ ] **Step 6: Run tests — verify they pass**

Run: `mvn test -pl runtime -Dtest=ResolutionIngestionServiceTest -q`

- [ ] **Step 7: Commit**

```bash
git add api/src runtime/src
git commit -m "$(cat <<'EOF'
feat(#1081): CorpusSourceAdapter SPI + ResolutionIngestionService

New SPI for knowledge base document ingestion (CorpusSourceAdapter).
ResolutionIngestionService bridges adapter to CbrCaseMemoryStore with
deterministic caseId for idempotent re-ingestion and per-document error
isolation.

Refs #1081

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Batch 4: Retrieval Feedback [handler fixes + observer]

**Neocortex prerequisite:** CbrRetrievalTracker.feedback() method.

### Task 5: Handler fixes + RetrievalFeedbackObserver

**Files:**
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/WorkflowExecutionCompletedHandler.java` (iteration fix + DECLINED outcome fix)
- Create: `runtime/src/main/java/io/casehub/engine/internal/routing/RetrievalFeedbackObserver.java`
- Test: `runtime/src/test/java/io/casehub/engine/internal/routing/RetrievalFeedbackObserverTest.java`

**Interfaces:**
- Consumes: `StepOutcomeObserver` SPI, `CbrRetrievalTracker` (neocortex), `EventLogRepository`
- Produces: Retrieval feedback per worker completion step

- [ ] **Step 1: Fix StepOutcomeObserver iteration in WorkflowExecutionCompletedHandler**

Use `ide_find_references` on `fireStepOutcomeObserver` to locate the single-dispatch call. Change from `stepOutcomeObserver.get()` to iterating:

```java
for (StepOutcomeObserver observer : stepOutcomeObserver) {
    try {
        observer.onStepOutcome(event);
    } catch (Exception e) {
        log.warn("StepOutcomeObserver failed: {}", e.getMessage());
    }
}
```

- [ ] **Step 2: Fix DECLINED outcome mapping in handleSemanticFailure()**

In the `WorkerOutcome.Declined` branch, pass `RoutingOutcome.DECLINED` instead of `RoutingOutcome.FAILURE`:

```java
case WorkerOutcome.Declined d -> fireStepOutcomeObserver(event, RoutingOutcome.DECLINED);
```

- [ ] **Step 3: Write test for RetrievalFeedbackObserver**

```java
@Test
void successOutcomeRecordsRelevantFeedback() {
    var experiences = List.of(mockExperience("case-1", 0.8));
    setupEventLogWithExperiences(experiences, "binding-1");

    var event = new StepOutcomeEvent(caseId, "tenant-1", "caseType",
        "binding-1", "cap-1", "worker-1",
        RoutingOutcome.SUCCESS, Map.of(), Duration.ofSeconds(5));

    observer.onStepOutcome(event);

    verify(cbrTracker).feedback(eq(caseId.toString()), eq("tenant-1"),
        argThat(entries -> entries.size() == 1
            && entries.get(0).outcome() == RetrievalOutcome.RELEVANT));
}

@Test
void declinedOutcomeRecordsNotRelevantForDeclinedAgent() {
    var experiences = List.of(mockExperience("case-1", 0.8));
    setupEventLogWithExperiences(experiences, "binding-1");

    var event = new StepOutcomeEvent(caseId, "tenant-1", "caseType",
        "binding-1", "cap-1", "worker-1",
        RoutingOutcome.DECLINED, Map.of(), Duration.ofSeconds(5));

    observer.onStepOutcome(event);

    verify(cbrTracker).feedback(eq(caseId.toString()), eq("tenant-1"),
        argThat(entries -> entries.get(0).outcome() == RetrievalOutcome.NOT_RELEVANT));
}

@Test
void noOpWhenTrackerAbsent() {
    // Instance<CbrRetrievalTracker> not resolvable
    var observer = new RetrievalFeedbackObserver(
        Instance.empty(), eventLogRepo);
    // Should not throw
    observer.onStepOutcome(event);
}
```

- [ ] **Step 4: Run tests — verify they fail**

- [ ] **Step 5: Implement RetrievalFeedbackObserver**

```java
@ApplicationScoped
public class RetrievalFeedbackObserver implements StepOutcomeObserver {
    @Inject Instance<CbrRetrievalTracker> cbrTracker;
    @Inject EventLogRepository eventLogRepo;

    @Override
    public void onStepOutcome(StepOutcomeEvent event) {
        if (!cbrTracker.isResolvable()) return;
        var experiences = loadExperiencesFromEventLog(event.caseId(), event.bindingName(), event.tenancyId());
        if (experiences.isEmpty()) return;
        var feedbackEntries = mapOutcomeToFeedback(event.outcome(), experiences);
        if (!feedbackEntries.isEmpty()) {
            try {
                cbrTracker.get().feedback(event.caseId().toString(), event.tenancyId(), feedbackEntries);
            } catch (Exception e) {
                log.warn("Retrieval feedback recording failed for case {}: {}", event.caseId(), e.getMessage());
            }
        }
    }
}
```

- [ ] **Step 6: Run tests — verify they pass**

- [ ] **Step 7: Run full runtime test suite**

Run: `mvn test -pl runtime -q`

- [ ] **Step 8: Commit**

```bash
git add runtime/src
git commit -m "$(cat <<'EOF'
feat(#1081): RetrievalFeedbackObserver + handler fixes

Layer 1 retrieval feedback via StepOutcomeObserver:
- Fix fireStepOutcomeObserver() iteration (single → multi-observer)
- Fix DECLINED outcome mapping (FAILURE → DECLINED)
- RetrievalFeedbackObserver correlates retrieval trace with outcome
- Transparent no-op when CbrRetrievalTracker absent

Refs #1081

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Batch 5: Candidate Presentation + Selection Feedback + Polish

### Task 6: JudgmentTarget candidate enrichment

**Files:**
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandler.java`
- Test: `runtime/src/test/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandlerTest.java` (extend)

**Interfaces:**
- Consumes: `CbrRetrievalService.retrieve()` (mixed results from Task 3), `JudgmentRequest` / `JudgmentPayload.BindingPayload`
- Produces: `_candidates.<bindingName>` context path with candidate summaries

- [ ] **Step 1: Read CaseContextChangedEventHandler.publishByTarget() for JudgmentTarget path**

Use `ide_file_structure` and `ide_find_references` on `publishJudgmentSchedule` to understand the current judgment dispatch path.

- [ ] **Step 2: Write test for candidate population on judgment binding with CbrConfig**

```java
@Test
void judgmentBindingWithCbrConfigPopulatesCandidates() {
    // Setup: case definition with cbr config + judgment binding
    // Trigger: CONTEXT_CHANGED
    // Assert: _candidates.<bindingName> written to context as JSON array
    // Assert: judgment payload carries experiences
}

@Test
void judgmentBindingWithoutCbrConfigSkipsCandidatePopulation() {
    // Setup: case definition without cbr config + judgment binding
    // Assert: no _candidates path written
}

@Test
void candidateSummariesExcludeFullDocumentContent() {
    // Assert: _candidates entries contain caseId, sourceType, similarity, problem, confidence, stepCount
    // Assert: _candidates entries do NOT contain solution prose or full steps
}
```

- [ ] **Step 3: Run tests — verify they fail**

- [ ] **Step 4: Add candidate population to judgment dispatch path**

In `publishJudgmentSchedule()` (or the equivalent method), before scheduling the judgment:

1. Check if `definition.getCbrConfig() != null`
2. If yes, call `cbrRetrievalService.retrieve(definition, instance)`
3. Write candidate summaries to `_candidates.<bindingName>` via `engineSet()` (suppresses CONTEXT_CHANGED)
4. Include experiences in the `JudgmentPayload.BindingPayload`

- [ ] **Step 5: Add ResolutionSelection validation in judgment completion**

In the judgment completion path, validate `selectedCaseId` against `_candidates.<bindingName>`. Reject invalid selections.

- [ ] **Step 6: Run tests — verify they pass**

- [ ] **Step 7: Commit**

```bash
git add runtime/src
git commit -m "$(cat <<'EOF'
feat(#1081): JudgmentTarget candidate enrichment + ResolutionSelection validation

When a judgment binding fires on a case with CbrConfig, CBR retrieval
populates _candidates.<bindingName> with ranked summaries (via engineSet
to suppress CONTEXT_CHANGED). Judgment completion validates selectedCaseId
against presented candidates.

Refs #1081

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

### Task 7: Selection feedback + outcome weighting + documentation

**Files:**
- Modify: judgment completion handler (for selection feedback)
- Modify: `runtime/src/main/resources/application.properties` (outcome weighting)
- Modify: `docs/guides/cbr-playbook-guide.md` (documentation)
- Create: `runtime/src/main/java/io/casehub/engine/internal/routing/SelectionFeedbackRecorder.java`
- Test: `runtime/src/test/java/io/casehub/engine/internal/routing/SelectionFeedbackRecorderTest.java`

**Interfaces:**
- Consumes: `CbrRetrievalTracker.feedback()`, `_candidates.<bindingName>` context path, `ResolutionSelection`
- Produces: Layer 3 selection feedback signals

- [ ] **Step 1: Write test for selection feedback**

```java
@Test
void selectedCandidateRecordsHighlyRelevant() {
    var candidates = List.of(
        candidateSummary("case-1", 0.9),
        candidateSummary("case-2", 0.7),
        candidateSummary("case-3", 0.3));

    recorder.recordSelectionFeedback(caseId, "tenant-1", "binding-1",
        new ResolutionSelection("case-1", ResolutionSourceType.PLAN_TRACE, null),
        candidates);

    verify(cbrTracker).feedback(eq(caseId.toString()), eq("tenant-1"),
        argThat(entries -> {
            assertThat(entries).hasSize(2); // case-1 HIGHLY_RELEVANT, case-2 PARTIALLY_RELEVANT (≥0.5)
            return true;
        }));
}

@Test
void unselectedBelowThresholdGetsNoSignal() {
    var candidates = List.of(
        candidateSummary("case-1", 0.9),
        candidateSummary("case-3", 0.3));

    recorder.recordSelectionFeedback(caseId, "tenant-1", "binding-1",
        new ResolutionSelection("case-1", ResolutionSourceType.PLAN_TRACE, null),
        candidates);

    // case-3 (similarity 0.3 < threshold 0.5) should NOT appear in feedback
    verify(cbrTracker).feedback(any(), any(),
        argThat(entries -> entries.size() == 1));
}
```

- [ ] **Step 2: Run tests — verify they fail**

- [ ] **Step 3: Implement SelectionFeedbackRecorder**

- [ ] **Step 4: Wire SelectionFeedbackRecorder into judgment completion path**

- [ ] **Step 5: Run tests — verify they pass**

- [ ] **Step 6: Enable outcome weighting by default**

Add to `runtime/src/main/resources/application.properties`:
```properties
casehub.cbr.outcome-weighting.enabled=true
```

- [ ] **Step 7: Update cbr-playbook-guide.md**

Add "Outcome Weighting" section documenting the default-on behavior, the influence parameter, and the null-confidence handling.

- [ ] **Step 8: Run full test suite**

Run: `mvn test -pl runtime -q`

- [ ] **Step 9: Commit**

```bash
git add runtime/src docs/guides
git commit -m "$(cat <<'EOF'
feat(#1081): selection feedback (Layer 3) + outcome weighting default-on

Layer 3 selection feedback records HIGHLY_RELEVANT for selected candidate,
PARTIALLY_RELEVANT for unselected above configurable threshold
(casehub.cbr.selection-feedback.threshold, default 0.5).

Enable casehub.cbr.outcome-weighting.enabled=true by default.
Update cbr-playbook-guide.md with outcome weighting documentation.

Closes #1081

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## References

- [2026-09-11-unified-resolution-pipeline-design.md] — design spec this plan implements
- [engine/runtime/src/main/java/io/casehub/engine/internal/routing/CbrRetrievalService.java] — current CBR retrieval bridge
- [engine/runtime/src/main/java/io/casehub/engine/internal/engine/handler/WorkflowExecutionCompletedHandler.java] — completion handler with StepOutcomeObserver
- [engine/runtime/src/main/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandler.java] — dispatch handler with judgment path
- [neocortex/memory-api/src/main/java/io/casehub/neocortex/memory/cbr/ResolutionGuide.java] — textual CBR case type
- [engine/docs/guides/cbr-playbook-guide.md] — current CBR documentation
- [GitHub #1081] — epic issue
