# Unified Resolution Pipeline — Design Spec

> **Issue:** casehubio/engine#1081
> **Status:** Draft
> **Scale:** XL / Complexity: High
> **Prerequisite:** CBR naming cleanup (separate XS issue — PlanCbrCase → ResolvedCase, TextualCbrCase → ResolutionGuide in engine imports)
> **Blocked by:** neocortex#304 (CLOSED — extracted hortora/engine capabilities into neocortex)

---

## 1. Problem

CaseHub's CBR retrieval and agent routing work well for the fully automated path: retrieve similar past cases, score agents by historical success, dispatch the best-fit agent. But real-world case management has four execution quadrants:

| Select | Execute | Example |
|--------|---------|---------|
| Agent | Agent | CBR-routed automated investigation |
| Agent | Human | Agent picks the runbook, analyst follows it manually |
| Human | Agent | Analyst picks from ranked candidates, engine dispatches |
| Human | Human | Analyst picks a resolution procedure and follows it |

Today the engine only serves the Agent→Agent quadrant. The human alternative is missing at every stage:

1. **No document retrieval** — knowledge base articles (runbooks, SOPs, investigation procedures) can't be retrieved alongside plan traces. `ResolutionGuide` exists in neocortex but the engine doesn't ingest or present textual cases.
2. **No candidate presentation** — when CBR retrieves 5 similar cases, there's no mechanism to present them to a human for selection. The agent always selects.
3. **No retrieval feedback** — the engine stores retrieval data in EventLog metadata but never evaluates whether the retrieved cases were relevant. No feedback loop closes.
4. **No selection feedback** — when a human does make a selection (via JudgmentTarget), the selection isn't fed back to improve future retrieval.
5. **Outcome weighting disabled** — neocortex's `OutcomeWeightingCbrCaseMemoryStore` decorator is config-gated and disabled by default. Cases with higher outcome confidence don't rank higher.

## 2. Vision

Every resolution — automated, human, or hybrid — follows the same pipeline:

```
Situation → Retrieve → Rank → Select → Execute → Feedback
```

The variable is who selects and who executes. The fully automated path (current behavior) remains the default. Human-in-the-loop adds alternatives at each stage, not gates.

### 2.1 Why CBR for documents (not RAG)

Neocortex has both RAG (`rag-*` modules) and CBR (`memory-*` modules). RAG is designed for chunk-level retrieval — it splits documents into chunks, embeds them, and retrieves relevant chunks. CBR is designed for whole-case retrieval — it stores complete cases with features and retrieves similar ones.

Resolution guidance is whole-document retrieval: a runbook is a case (problem description + solution steps), not a bag of chunks. CBR's `CbrCaseMemoryStore` already supports:
- Dense vector search (semantic matching via problem() field embedding)
- SPLADE sparse embeddings
- BM25 server-side inference
- Feature vector similarity (structural matching)
- Hybrid fusion across all four legs

This makes CBR functionally equivalent to RAG for whole-document semantic retrieval, while additionally supporting feature-based structural matching that RAG lacks. The mixed ranked list of plan traces and documents is natural in CBR — both are cases with problem/solution structure. RAG would require a separate retrieval pipeline and a merge step.

The trade-off: CBR's feature extraction assumes the adapter can identify discriminating features. For documents where features aren't obvious, the dense vector (semantic) leg dominates and feature weights should be low.

## 3. Architecture

The unified pipeline is not a new module. It's distributed across existing engine components, with each piece landing in its natural home:

| Pipeline stage | Component | Change type |
|----------------|-----------|-------------|
| **Ingest** | `ResolutionIngestionService` (new, runtime) | New service bridging corpus SPI to CBR storage |
| **Retrieve** | `CbrRetrievalService` (runtime) | Extended for mixed results (ResolvedCase + ResolutionGuide) |
| **Rank** | Neocortex decorators (classpath-activated) | Outcome weighting enabled by default |
| **Select (auto)** | `AgentRoutingStrategy` (runtime) | No change — existing CBR signal provider |
| **Select (human)** | `JudgmentTarget` binding (YAML) | Payload enriched with ranked candidate list |
| **Execute** | `WorkerScheduleEventHandler` → scheduler | No change |
| **Feedback L1** | `RetrievalFeedbackObserver` (new, runtime) | StepOutcomeObserver: correlates retrieval trace with worker outcome |
| **Feedback L2** | `CbrCaseRetainObserver` (runtime) | Already wired — no change |
| **Feedback L3** | Judgment completion path (runtime) | Selection recorded as feedback signal |

### 3.1 Data Flow

```
                    ┌─────────────────────────────────────────────────┐
                    │                 Ingestion                       │
                    │  CorpusSourceAdapter → ResolutionIngestionService│
                    │       → CbrCaseMemoryStore.store(ResolutionGuide)│
                    └──────────────────────┬──────────────────────────┘
                                           │
                    ┌──────────────────────▼──────────────────────────┐
                    │                 Retrieval                       │
                    │  CbrRetrievalService.retrieve()                 │
                    │    → CbrCaseMemoryStore.retrieveSimilar()       │
                    │    → Mixed List<RetrievedExperience>            │
                    │       (PLAN_TRACE + RESOLUTION_GUIDE)           │
                    └──────────────────────┬──────────────────────────┘
                                           │
                          ┌────────────────┴────────────────┐
                          │                                 │
                 ┌────────▼────────┐               ┌───────▼────────┐
                 │  Automated path │               │  Human path    │
                 │  AgentRouting   │               │  JudgmentTarget│
                 │  Strategy       │               │  binding       │
                 │  selects agent  │               │  presents      │
                 └────────┬────────┘               │  candidates    │
                          │                        └───────┬────────┘
                          │                                │
                          │                     human/LLM selects
                          │                                │
                 ┌────────▼────────────────────────────────▼────────┐
                 │                   Dispatch                       │
                 │  WorkerScheduleEventHandler → scheduler          │
                 └────────────────────┬────────────────────────────┘
                                      │
                 ┌────────────────────▼────────────────────────────┐
                 │                  Feedback                       │
                 │  L1: RetrievalFeedbackEvaluator (per-step)      │
                 │  L2: CbrCaseRetainObserver (per-case, existing) │
                 │  L3: Selection feedback (per-judgment)          │
                 └─────────────────────────────────────────────────┘
```

## 4. New Types

### 4.1 Neocortex memory-api

**`ResolutionStep`** — record on `ResolutionGuide`, representing one structured step from a knowledge base article.

```java
public record ResolutionStep(
    String description,
    @Nullable String preconditions,
    @Nullable String expectedOutcome,
    @Nullable String automationHint
) {
    public ResolutionStep {
        Objects.requireNonNull(description, "description must not be null");
        if (description.isBlank()) throw new IllegalArgumentException("description must not be blank");
    }
}
```

**`ResolutionGuide`** gains `List<ResolutionStep> steps` (nullable, defaults to `List.of()` in compact constructor) and a `withSteps(List<ResolutionStep>)` copy-with-modify method (same pattern as `ResolvedCase.withFeatures()`). The `solution` text field remains for prose. Both are populated — prose for humans, steps for LLMs and the engine.

Follows the same pattern as `ResolvedCase.resolutionStep` (List of plan trace steps).

**Steps are optional.** Not all resolution knowledge is step-based — declarative knowledge ("when error X, root cause is Y"), decision trees, reference knowledge, and advisory knowledge don't decompose into sequential steps. The `steps` field is nullable. Documents without steps carry prose only (`solution` text). The engine handles both: step-based documents enable LLM-guided execution; prose-only documents are presented to human analysts or used as context by agent workers.

### 4.2 Engine api (`io.casehub.api.model`)

**`RetrievedExperience`** gains three fields (all backward-compatible with null/PLAN_TRACE defaults):

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `sourceType` | `ResolutionSourceType` | `PLAN_TRACE` | Discriminator: plan trace vs resolution guide |
| `documentContent` | `@Nullable String` | `null` | Prose solution from ResolutionGuide |
| `documentSteps` | `@Nullable List<DocumentStep>` | `null` | Structured steps mapped from ResolutionStep |

Backward-compatible constructor: existing N-arg constructor passes `PLAN_TRACE`, `null`, `null`.

**`ResolutionSourceType`** — enum: `PLAN_TRACE`, `RESOLUTION_GUIDE`.

**`DocumentStep`** — engine-owned record mapped from neocortex `ResolutionStep`:

```java
public record DocumentStep(
    String description,
    @Nullable String preconditions,
    @Nullable String expectedOutcome,
    @Nullable String automationHint
) {}
```

### 4.3 Engine api SPI (`io.casehub.api.spi`)

**`CorpusSourceAdapter`** — SPI for knowledge base document ingestion. Extends `NamedStrategy`.

```java
public interface CorpusSourceAdapter extends NamedStrategy {
    List<ResolutionGuideInput> discover(String tenancyId);
    default void onChange(Consumer<CorpusChangeEvent> listener) {}
    default boolean supportsChangeDetection() { return false; }
}
```

**`ResolutionGuideInput`** — record for ingestion input:

```java
public record ResolutionGuideInput(
    String problem,
    String solution,
    @Nullable List<ResolutionStepInput> steps,
    Map<String, FeatureValue> features,
    String domain,
    @Nullable Map<String, Object> metadata
) {}
```

**`ResolutionStepInput`** — record:

```java
public record ResolutionStepInput(
    String description,
    @Nullable String preconditions,
    @Nullable String expectedOutcome,
    @Nullable String automationHint
) {}
```

**`CorpusChangeEvent`** — record: `changeType` (ADDED, UPDATED, REMOVED), `input` (ResolutionGuideInput), `documentId` (String).

**`NoOpCorpusSourceAdapter`** — `@DefaultBean @ApplicationScoped` in runtime, returns empty list, no change detection.

### 4.4 Engine runtime

**`ResolutionIngestionService`** — `@ApplicationScoped`. Bridges `CorpusSourceAdapter` → `CbrCaseMemoryStore`.

- Injects `Instance<CorpusSourceAdapter>` — transparent no-op when no adapter discovered
- Injects `Instance<CbrCaseMemoryStore>` — transparent no-op when no store available
- On `@Observes StartupEvent @Priority(300)`: calls `adapter.discover(tenancyId)` for initial load, subscribes to `adapter.onChange()` for incremental updates
- Maps `ResolutionGuideInput` → `ResolutionGuide` with features
- Domain resolution follows the same chain as `CbrRetrievalService.resolveDomain()`
- Error isolation per document — one failed ingestion doesn't block others

**`RetrievalFeedbackObserver`** — `@ApplicationScoped`, implements `StepOutcomeObserver`. Correlates retrieval traces with worker outcomes via the existing per-step observer SPI (same pattern as `CaseOutcomeObserver` for case-level CBR retain).

- Injects `Instance<CbrRetrievalTracker>` — transparent no-op when tracker absent
- `StepOutcomeEvent` carries `caseId`, `tenancyId`, `bindingName`, `workerName`, `outcome`, `contextSnapshot`
- On SUCCESS/COMPLETED: all retrieved cases scored as `RELEVANT`
- On DECLINED/FAILED: retrieved cases that informed the failed agent scored as `NOT_RELEVANT`; others unscored (avoids noise)
- On EXPIRED: all retrieved cases scored as `PARTIALLY_RELEVANT` (timeout doesn't imply irrelevance)
- Reads `experiences` from EventLog metadata (already stored at dispatch time by `WorkerScheduleEventHandler`)

**Neocortex dependency:** `CbrRetrievalTracker` (neocortex memory-api) currently has `record()` and `findTraces()` but no feedback method. This spec requires adding `feedback(String caseId, String tenancyId, List<CbrRetrievalFeedback> entries)` to `CbrRetrievalTracker` in neocortex. `CbrRetrievalFeedback` record: `(String tracedCaseId, RetrievalOutcome outcome)`. This is a neocortex-memory-api change, tracked as a dependency of child issue #5.

**Selection feedback** — wired in the judgment completion path:

- When a judgment binding resolves with a selection, the engine records:
  - Selected candidate: `HIGHLY_RELEVANT` feedback
  - Unselected candidates with similarity ≥ 0.5: `PARTIALLY_RELEVANT`
  - Unselected candidates below 0.5: no signal
- Wired in `PlanItemCompletionApplier` (for co-located work-cloudevent path) or the equivalent judgment completion handler

## 5. CbrRetrievalService Extension

`CbrRetrievalService.retrieve()` already handles multiple CBR types via `BUILT_IN_TYPES` map. The extension is in the mapping from `ScoredCbrCase` to `RetrievedExperience`:

- `ScoredCbrCase<ResolvedCase>` → `RetrievedExperience` with `sourceType=PLAN_TRACE`, existing `planSteps` populated
- `ScoredCbrCase<ResolutionGuide>` → `RetrievedExperience` with `sourceType=RESOLUTION_GUIDE`, `documentContent` from `solution()`, `documentSteps` mapped from `steps()`

Cross-type retrieval via `CaseTypeScope.AllInDomain()` already returns mixed results. The engine just needs to map both types.

`retrieveForSelection()` also returns mixed results — consumers group by `sourceType` and `caseType` for presentation.

## 6. Candidate Presentation via JudgmentTarget

### 6.1 Flow

For human-in-the-loop resolution selection, the case definition declares two bindings:

1. **Judgment binding** — fires when the situation is detected but no resolution is selected yet. Presents ranked CBR candidates to a human (or LLM, or A2A agent) for selection.
2. **Capability binding** — fires when a resolution has been selected. Dispatches the selected resolution.

### 6.2 Candidate context path

When `CaseContextChangedEventHandler` evaluates a judgment binding that targets a capability with a `cbr:` block:

1. `CaseContextChangedEventHandler.publishByTarget()` detects that the binding is a `JudgmentTarget` AND the case definition has a `CbrConfig`. This is the trigger for candidate population — judgment bindings without a `CbrConfig` on the case definition skip this path entirely.
2. CBR retrieval runs via `CbrRetrievalService.retrieve()` (same call as the existing capability dispatch path)
3. Candidate **summaries** are written to `_candidates.<bindingName>` in the working layer as a JSON array of `{caseId, sourceType, similarity, caseType, problem, confidence, stepCount}`. Full document content is NOT written to the context — it's retrieved on demand via `CbrCaseMemoryStore` when the selected candidate is dispatched. This prevents context bloat for large knowledge bases.
4. The judgment payload includes the candidate summaries from this context path. Writes to `_candidates.*` suppress `CONTEXT_CHANGED` (same pattern as `_diagnostics` writes via `engineSet()`) to prevent circular dispatch.
5. The human/LLM resolution is written to the output path specified by the binding's `producedKeys`

### 6.3 YAML example

```yaml
dsl: "0.1.0"
namespace: soc
name: alert-triage-with-hitl
version: "1.0.0"

spec:
  cbr:
    domain: "soc-incidents"
    crossType: true
    features:
      severity: ".alert.severity"
      category: ".alert.category"
    weights:
      category: 2.0

  capabilities:
    - name: investigate
      inputProjection: "{alert: .alert, resolution: .selectedResolution}"
      outputProjection: "{verdict: .verdict}"

  bindings:
    - name: select-resolution
      judgment:
        caller:
          human:
            title: "Select investigation approach"
            candidateGroups: [soc-analysts]
            outcomes: [approve, reject, escalate]
        resolutionType: io.casehub.api.model.ResolutionSelection
      on: ".alert != null and .selectedResolution == null"
      producedKeys: [selectedResolution]

    - name: investigate-alert
      capability: investigate
      on: ".selectedResolution != null and .verdict == null"
      producedKeys: [verdict]

  completion:
    success:
      allOf: [investigated]
    failure:
      anyOf: [triage-failed]

  goals:
    - name: investigated
      condition: ".verdict != null"
    - name: triage-failed
      condition: "._diagnostics.\"investigate-alert\".status == \"REROUTES_EXHAUSTED\""

workers:
  - name: investigation-agent
    capabilities: [investigate]
    agent:
      model: anthropic
      modelName: claude-sonnet-4-20250514
      systemPrompt: |
        You are a SOC investigator. The selected resolution approach
        is provided in the input. Follow the steps and adapt to the
        current context.
```

### 6.4 Automated path

For the automated path, the judgment binding is simply absent. The capability binding fires directly, CBR informs routing via the existing signal provider, and the agent selects autonomously. No YAML ceremony for the default path.

### 6.5 ResolutionSelection type

Engine-owned type for judgment resolution:

```java
public record ResolutionSelection(
    String selectedCaseId,
    ResolutionSourceType sourceType,
    @Nullable String rationale
) {}
```

The judgment's `resolutionType` is `ResolutionSelection`. The human/LLM picks a candidate by `selectedCaseId` and optionally provides a rationale.

## 7. Feedback Layers

### 7.1 Layer 1 — Retrieval relevance (per-step)

**When:** After each worker execution step completes (success or failure), via `StepOutcomeObserver` SPI.

**How:** `RetrievalFeedbackObserver.onStepOutcome(StepOutcomeEvent event)`:

1. Read `experiences` from EventLog metadata (stored at dispatch time)
2. If empty → return (no retrieval to evaluate)
3. Map worker outcome to relevance:
   - `Success` → each experience: `RetrievalOutcome.RELEVANT`
   - `Completed` → each experience: `RetrievalOutcome.RELEVANT`
   - `Declined`/`Failed` → experiences for the declined/failed agent: `RetrievalOutcome.NOT_RELEVANT`; others: no signal
   - `Expired` → each experience: `RetrievalOutcome.PARTIALLY_RELEVANT`
4. Call `cbrRetrievalTracker.feedback(feedbackEntries)` for each

**Threading:** `Instance<CbrRetrievalTracker>` with `isResolvable()` guard — transparent no-op when `memory-cbr-tracking` is not on the classpath. Observer is fired from `WorkflowExecutionCompletedHandler.fireStepOutcomeObserver()` with `isUnsatisfied()` guard — same pattern as the existing `StepOutcomeObserver`.

### 7.2 Layer 2 — CBR outcome tracking (per-case)

Already wired. `CbrCaseRetainObserver` stores `ResolvedCase` entries on case terminal state. `CbrCaseMemoryStore.recordOutcome()` updates confidence via EMA. No new code needed.

### 7.3 Layer 3 — Selection feedback (per-judgment)

**When:** After a judgment binding resolves with a `ResolutionSelection`.

**How:** In the judgment completion path (triggered by `ACTION_GATE_APPROVED` or judgment CloudEvent):

1. Read `_candidates.<bindingName>` from the case context
2. Read the resolution (selected candidate ID)
3. For each candidate:
   - Selected → `CbrRetrievalTracker.feedback(caseId, HIGHLY_RELEVANT)`
   - Unselected, similarity ≥ 0.5 → `PARTIALLY_RELEVANT`
   - Unselected, similarity < 0.5 → no signal
4. Write `RESOLUTION_SELECTED` EventLog entry with metadata: `selectedCaseId`, `selectedSourceType`, `candidateCount`, `rationale`

**EventLog metadata schema for `RESOLUTION_SELECTED`:**
```json
{
  "selectedCaseId": "uuid",
  "selectedSourceType": "PLAN_TRACE|RESOLUTION_GUIDE",
  "candidateCount": 5,
  "rationale": "Matches the credential harvesting pattern from last month",
  "bindingName": "select-resolution"
}
```

## 8. Document Ingestion

### 8.1 Ingestion pipeline

```
CorpusSourceAdapter.discover(tenancyId)
  → List<ResolutionGuideInput>
  → ResolutionIngestionService.ingest(inputs)
     → for each input:
        → ResolutionGuide(problem, solution, outcome=null, confidence=null,
                          trustScore=null, producerAgentId=null)
        → ResolutionGuide.withSteps(input.steps())
        → CbrCaseMemoryStore.store(guide, domain, tenancyId, features, scope)
```

### 8.2 Change detection

```java
if (adapter.supportsChangeDetection()) {
    adapter.onChange(event -> {
        switch (event.changeType()) {
            case ADDED -> ingest(event.input(), event.documentId());
            case UPDATED -> update(event.documentId(), event.input());
            case REMOVED -> remove(event.documentId());
        }
    });
}
```

`UPDATED` uses `CbrCaseMemoryStore.supersede()` to mark the old entry and store the new one. `REMOVED` uses `CbrCaseMemoryStore.erase()`.

### 8.3 Feature extraction

The adapter is responsible for extracting features from documents. The engine provides no default feature extraction — each corpus has different feature dimensions.

```java
public class SocRunbookAdapter implements CorpusSourceAdapter {
    @Override
    public List<ResolutionGuideInput> discover(String tenancyId) {
        return loadRunbooks().stream()
            .map(doc -> new ResolutionGuideInput(
                doc.title(),
                doc.content(),
                parseSteps(doc.content()),
                Map.of("category", FeatureValue.string(doc.category()),
                       "severity", FeatureValue.string(doc.applicableSeverity())),
                "soc-incidents",
                Map.of("sourceFile", doc.path())))
            .toList();
    }
}
```

## 9. Outcome Weighting Default

Single property change in engine's default `application.properties`:

```properties
casehub.cbr.outcome-weighting.enabled=true
```

The `OutcomeWeightingCbrCaseMemoryStore` decorator (neocortex `memory` module) applies a score multiplier: `score * (1 - α + α * confidence)` where `α = 0.3` (configurable via `casehub.cbr.outcome-weighting.influence`). Cases with higher outcome confidence rank higher.

Document in `cbr-playbook-guide.md` under a new "Outcome Weighting" section.

## 10. Child Issues

| # | Title | Scale | Complexity | Depends on |
|---|-------|-------|------------|------------|
| 0 | CBR naming cleanup (PlanCbrCase → ResolvedCase, TextualCbrCase → ResolutionGuide) | XS | Low | — |
| 1 | ResolutionStep on ResolutionGuide + DocumentStep mapping | S | Low | #0 |
| 2 | RetrievedExperience extension (sourceType, documentContent, documentSteps) | S | Low | #1 |
| 3 | CbrRetrievalService mixed retrieval mapping | M | Med | #2 |
| 4 | CorpusSourceAdapter SPI + ResolutionIngestionService | M | Med | #1 |
| 5 | RetrievalFeedbackObserver (Layer 1 via StepOutcomeObserver) + CbrRetrievalTracker.feedback() neocortex change | M | Med | #3, neocortex change |
| 6 | JudgmentTarget candidate presentation + ResolutionSelection | L | High | #3 |
| 7 | Selection feedback (Layer 3) | S | Med | #5, #6 |
| 8 | Outcome weighting default-on + documentation | XS | Low | — |

Issue #0 is a prerequisite, filed separately. Issues #1-8 are children of engine#1081.

Critical path: #0 → #1 → #2 → #3 → #6 (candidate presentation is the most complex and novel).

Parallel work: #4 (ingestion) can proceed independently after #1. #8 can land at any time.

## 11. Testing Strategy

### 11.1 Unit tests

- `RetrievalFeedbackObserver` — mock `CbrRetrievalTracker`, verify feedback calls for each outcome type (Success, Declined, Failed, Expired)
- `ResolutionIngestionService` — mock `CorpusSourceAdapter` and `CbrCaseMemoryStore`, verify store calls and error isolation
- `CbrRetrievalService` mixed mapping — verify `ResolutionGuide` → `RetrievedExperience` with `sourceType=RESOLUTION_GUIDE`
- `ResolutionSelection` — round-trip Jackson serialization

### 11.2 Integration tests

- End-to-end judgment candidate flow: case starts → CBR retrieves mixed results → judgment binding fires with candidates → human selects → capability binding dispatches
- Feedback loop: case completes → `RetrievalFeedbackEvaluator` records relevance → verify tracker received correct feedback
- Ingestion: adapter produces documents → `ResolutionIngestionService` stores → CBR retrieves them

### 11.3 Test infrastructure

- `casehub-persistence-memory` for in-memory engine state
- `casehub-neocortex-memory-cbr-inmem` for in-memory CBR store
- Mock `CorpusSourceAdapter` in test classes
- No Testcontainers — all tests run with in-memory stores

## 12. Migration and Backward Compatibility

- **No breaking changes.** All new fields on `RetrievedExperience` have null/default values. Existing case definitions work unchanged.
- **Outcome weighting flip:** Only affects deployments that upgrade AND have CBR active. The effect is a ranking improvement, not a behavioral change.
- **No database migration.** No Flyway scripts. No schema changes to engine tables.
- **Neocortex version dependency.** Engine must depend on a neocortex version that includes `ResolutionStep` on `ResolutionGuide`. This is a neocortex#1081-aligned change.

## References

- [engine#1081](https://github.com/casehubio/engine/issues/1081) — Epic issue with vision and comments
- [neocortex#304](https://github.com/casehubio/neocortex/issues/304) — Resolved blocker: hortora/engine capabilities extracted to neocortex
- `engine/runtime/src/main/java/io/casehub/engine/internal/routing/CbrRetrievalService.java` — Current CBR retrieval bridge
- `engine/runtime/src/main/java/io/casehub/engine/internal/memory/CbrCaseRetainObserver.java` — Current CBR retain observer
- `neocortex/memory-api/src/main/java/io/casehub/neocortex/memory/cbr/ResolutionGuide.java` — Textual CBR case type
- `neocortex/memory-api/src/main/java/io/casehub/neocortex/memory/cbr/ResolvedCase.java` — Plan CBR case type
- `neocortex/rag-tracking/` — RAG retrieval tracking module
- `neocortex/memory-cbr-tracking/` — CBR retrieval tracking module
- `neocortex/corpus-api/` — Corpus ingestion SPIs
- `engine/docs/guides/cbr-playbook-guide.md` — Current CBR documentation
- Engine CLAUDE.md sections: CBR Retrieval Bridge, JudgmentTarget, Unified Judgment Scheduling, Worker Outcome Handling, CaseOutcomeObserver SPI
