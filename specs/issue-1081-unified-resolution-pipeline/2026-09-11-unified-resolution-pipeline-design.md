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

**`GuidanceStep`** — record on `ResolutionGuide`, representing one structured step from a knowledge base article. Named `GuidanceStep` (not `ResolutionStep`) to avoid collision with the existing `ResolutionStep` record in `io.casehub.neocortex.memory.cbr`, which represents a plan execution trace step with fields `(bindingName, capabilityName, workerName, stepOutcome, priority, parameters, variantId)`.

```java
public record GuidanceStep(
    String description,
    @Nullable String preconditions,
    @Nullable String expectedOutcome,
    @Nullable String automationHint
) {
    public GuidanceStep {
        Objects.requireNonNull(description, "description must not be null");
        if (description.isBlank()) throw new IllegalArgumentException("description must not be blank");
    }
}
```

**`ResolutionGuide`** gains three additions:

1. `Map<String, FeatureValue> features` field — required, non-null, immutable copy (same pattern as `ResolvedCase.features`). Overrides the `CbrCase.features()` default of `Map.of()` so that `CbrCaseStore.store()` (which reads features from `cbrCase.features()`) persists extracted features.
2. `List<GuidanceStep> steps` field — nullable, defaults to `List.of()` in compact constructor.
3. `withFeatures(Map<String, FeatureValue>)` and `withSteps(List<GuidanceStep>)` copy-with-modify methods (same pattern as `ResolvedCase.withFeatures()`).

The `solution` text field remains for prose. Both are populated — prose for humans, steps for LLMs and the engine.

Follows the same pattern as `ResolvedCase.resolutionStep` (List of plan trace steps) and `ResolvedCase.features` (Map of feature values).

**Steps are optional.** Not all resolution knowledge is step-based — declarative knowledge ("when error X, root cause is Y"), decision trees, reference knowledge, and advisory knowledge don't decompose into sequential steps. The `steps` field is nullable. Documents without steps carry prose only (`solution` text). The engine handles both: step-based documents enable LLM-guided execution; prose-only documents are presented to human analysts or used as context by agent workers.

### 4.2 Engine api (`io.casehub.api.spi.routing`)

`RetrievedExperience` is in `io.casehub.api.spi.routing`. New companion types live in the same package.

**`RetrievedExperience`** gains three fields (all backward-compatible with null/PLAN_TRACE defaults):

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `sourceType` | `ResolutionSourceType` | `PLAN_TRACE` | Discriminator: plan trace vs resolution guide |
| `documentContent` | `@Nullable String` | `null` | Prose solution from ResolutionGuide |
| `documentSteps` | `@Nullable List<DocumentStep>` | `null` | Structured steps mapped from GuidanceStep |

Backward-compatible constructor: existing N-arg constructor passes `PLAN_TRACE`, `null`, `null`.

**`ResolutionSourceType`** — enum in `io.casehub.api.spi.routing`: `PLAN_TRACE`, `RESOLUTION_GUIDE`.

**`DocumentStep`** — engine-owned record in `io.casehub.api.spi.routing`, mapped from neocortex `GuidanceStep`:

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
    String documentId,
    String problem,
    String solution,
    @Nullable List<GuidanceStepInput> steps,
    Map<String, FeatureValue> features,
    String domain,
    @Nullable Map<String, Object> metadata
) {}
```

`documentId` is required (non-null, non-blank) — it serves as the deterministic identity for deduplication. The ingestion service derives a stable `caseId` from it (e.g., `UUID.nameUUIDFromBytes(documentId.getBytes(UTF_8))`) and uses `supersede()` for all stores, ensuring idempotent ingestion on application restart.

**`GuidanceStepInput`** — record:

```java
public record GuidanceStepInput(
    String description,
    @Nullable String preconditions,
    @Nullable String expectedOutcome,
    @Nullable String automationHint
) {}
```

**`CorpusChangeEvent`** — sealed interface with distinct event types:

```java
public sealed interface CorpusChangeEvent {
    record Added(ResolutionGuideInput input) implements CorpusChangeEvent {}
    record Updated(ResolutionGuideInput input, String documentId) implements CorpusChangeEvent {}
    record Removed(String documentId) implements CorpusChangeEvent {}
}
```

`Added` and `Updated` carry the full input; `Removed` carries only the `documentId` needed for removal (the adapter may no longer have the document content). This avoids forcing adapters to construct dummy `ResolutionGuideInput` records for removal events.

**Relationship to neocortex corpus module:** `casehub-neocortex-corpus` provides low-level corpus storage infrastructure (`ZipCorpusStore`, `FlatCorpusStore`, change detection). `CorpusSourceAdapter` is a higher-level SPI — it adapts domain-specific knowledge sources into `ResolutionGuideInput` records. A `CorpusSourceAdapter` implementation MAY use the neocortex corpus module as its backing storage (e.g., reading runbooks from a zip-based corpus), but the engine SPI is intentionally source-agnostic: adapters can read from APIs, databases, file systems, or any other knowledge source.

**`NoOpCorpusSourceAdapter`** — `@DefaultBean @ApplicationScoped` in runtime, returns empty list, no change detection.

### 4.4 Engine runtime

**`ResolutionIngestionService`** — `@ApplicationScoped`. Bridges `CorpusSourceAdapter` → `CbrCaseMemoryStore`.

- Injects `Instance<CorpusSourceAdapter>` — transparent no-op when no adapter discovered
- Injects `Instance<CbrCaseMemoryStore>` — transparent no-op when no store available
- **TenancyId source:** `casehub.corpus.tenancy-ids` config property (list). At startup, iterates over configured tenancies. Single-tenant deployments configure one value; multi-tenant deployments list all. No tenancy enumeration exists at startup (no cases exist yet, `CbrCaseAdmin.discoverTenants()` throws `UnsupportedOperationException`), so explicit configuration is required.
- On `@Observes StartupEvent @Priority(300)`: for each configured tenancyId, calls `adapter.discover(tenancyId)` for initial load, subscribes to `adapter.onChange()` for incremental updates
- Maps `ResolutionGuideInput` → `ResolutionGuide` with features
- Domain resolution follows the same chain as `CbrRetrievalService.resolveDomain()`
- Error isolation per document — one failed ingestion doesn't block others

**`RetrievalFeedbackObserver`** — `@ApplicationScoped`, implements `StepOutcomeObserver`. Correlates retrieval traces with worker outcomes via the existing per-step observer SPI.

**Prerequisite fixes in `WorkflowExecutionCompletedHandler`** (child issue #5):

1. **Iteration fix:** `fireStepOutcomeObserver()` currently uses `stepOutcomeObserver.get()` (single-dispatch). Must change to iterate — `for (StepOutcomeObserver obs : stepOutcomeObserver)` — matching the `CaseOutcomeObserver` pattern in `CaseStatusChangedHandler` (line 243). Without this, adding `RetrievalFeedbackObserver` alongside `NoOpStepOutcomeObserver` causes `AmbiguousResolutionException`.
2. **DECLINED outcome fix:** `handleSemanticFailure()` (lines 510-520) currently passes `RoutingOutcome.FAILURE` for ALL semantic failures (Declined, Failed, Expired). The `WorkerOutcome.Declined` branch must pass `RoutingOutcome.DECLINED` to `fireStepOutcomeObserver()` instead of `RoutingOutcome.FAILURE`. Without this, the DECLINED-specific feedback mapping below is unreachable — the observer never sees `RoutingOutcome.DECLINED`. The `ExperienceAnalyser` already assigns distinct weights to DECLINED vs FAILURE (-0.5 vs -1.0), confirming the enum was designed for this distinction.

- Injects `Instance<CbrRetrievalTracker>` — transparent no-op when tracker absent
- Injects `EventLogRepository` — required for reading experiences from dispatch-time metadata
- `StepOutcomeEvent` carries `caseId`, `tenancyId`, `caseType`, `bindingName`, `capabilityName`, `workerName`, `outcome`, `contextSnapshot`, `executionDuration`
- Experience correlation: queries `EventLogRepository` for the most recent `WORKER_SCHEDULED` EventLog by `caseId + bindingName`, deserializes `experiences` from metadata JSON. Correlation by `caseId + bindingName` is unambiguous because the observer fires synchronously from the handler — the most recent WORKER_SCHEDULED for that combination is guaranteed to be the relevant dispatch.
- Outcome-to-relevance mapping using actual `RoutingOutcome` values:
  - `SUCCESS` → each experience: `RetrievalOutcome.RELEVANT`
  - `DECLINED` → experiences for the declined agent: `RetrievalOutcome.NOT_RELEVANT`; others: no signal
  - `FAILURE` → all experiences: `RetrievalOutcome.NOT_RELEVANT` (FAILURE covers both failed and expired workers; includes transient infrastructure failures where retrieval may have been perfectly relevant — see trade-off note below)
  - `GATE_REJECTED` → no signal (gate rejection reflects human oversight, not retrieval quality)
  - `GATE_EXPIRED` → no signal (gate timeout is not retrieval feedback)
  - `CANCELLED`, `OBSOLETE` → no signal (external cancellation and obsolescence are not retrieval feedback)
- Calls `cbrRetrievalTracker.feedback(feedbackEntries)` for each

**FAILURE → NOT_RELEVANT trade-off:** `RoutingOutcome.FAILURE` conflates logic failures (wrong approach — retrieval WAS irrelevant) with transient infrastructure failures (timeout, OOM — retrieval quality is unknown). Recording NOT_RELEVANT for transient failures creates false-negative signals. This is accepted because: (a) transient failures trigger reroutes which, on success, produce RELEVANT signals that offset the false negative; (b) the `FailureClassifier` runs AFTER the step observer fires in `handleSemanticFailure()`, so the classification is unavailable at observation time; (c) reordering the handler or duplicating classification in the observer would add complexity disproportionate to the signal quality improvement. If transient failure rates are high enough to degrade retrieval quality, a follow-up can add `FailureCategory` to `StepOutcomeEvent` and filter in the observer.

**Neocortex dependency:** `CbrRetrievalTracker` (neocortex memory-api) currently has `record()` and `findTraces()` but no feedback method. This spec requires adding `feedback(String caseId, String tenancyId, List<CbrRetrievalFeedback> entries)` to `CbrRetrievalTracker` in neocortex. `CbrRetrievalFeedback` record: `(String tracedCaseId, RetrievalOutcome outcome)`. This is a neocortex-memory-api change, tracked as a dependency of child issue #5.

**Selection feedback** — wired in the judgment completion path:

- When a judgment binding resolves with a selection, the engine records:
  - Selected candidate: `HIGHLY_RELEVANT` feedback
  - Unselected candidates with similarity ≥ configurable threshold (`casehub.cbr.selection-feedback.threshold`, default 0.5): `PARTIALLY_RELEVANT`
  - Unselected candidates below threshold: no signal
- Wired in `PlanItemCompletionApplier` (for co-located work-cloudevent path) or the equivalent judgment completion handler

## 5. CbrRetrievalService Extension

`CbrRetrievalService.retrieve()` already handles multiple CBR types via `BUILT_IN_TYPES` map. The extension is in the mapping from `ScoredCbrCase` to `RetrievedExperience`:

- `ScoredCbrCase<ResolvedCase>` → `RetrievedExperience` with `sourceType=PLAN_TRACE`, existing `planTrace` populated
- `ScoredCbrCase<ResolutionGuide>` → `RetrievedExperience` with `sourceType=RESOLUTION_GUIDE`, `documentContent` from `solution()`, `documentSteps` mapped from `steps()` (`GuidanceStep` → `DocumentStep`)

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
4. The judgment is scheduled via `JudgmentRequest` with `JudgmentPayload.BindingPayload` (the modern API — `JudgmentScheduleRequest` is `@Deprecated(forRemoval = true)`). The `BindingPayload.experiences` field carries the retrieval results. Writes to `_candidates.*` suppress `CONTEXT_CHANGED` (same pattern as `_diagnostics` writes via `engineSet()`) to prevent circular dispatch.
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

**Candidate validation:** The judgment completion path validates that `selectedCaseId` is present in `_candidates.<bindingName>`. If the selected ID is not among the presented candidates (stale selection, wrong domain, or fabricated ID), the judgment resolution is rejected with a diagnostic — it is NOT silently accepted into the feedback loop as HIGHLY_RELEVANT. This prevents feedback pollution from invalid selections.

## 7. Feedback Layers

### 7.1 Layer 1 — Retrieval relevance (per-step)

**When:** After each worker execution step completes (success or failure), via `StepOutcomeObserver` SPI.

**How:** `RetrievalFeedbackObserver.onStepOutcome(StepOutcomeEvent event)`:

1. Query `EventLogRepository` for the most recent `WORKER_SCHEDULED` EventLog by `caseId + bindingName`
2. Deserialize `experiences` from the EventLog metadata JSON
3. If empty → return (no retrieval to evaluate)
4. Map worker outcome to relevance using actual `RoutingOutcome` values (requires DECLINED handler fix from §4.4):
   - `SUCCESS` → each experience: `RetrievalOutcome.RELEVANT`
   - `DECLINED` → experiences for the declined agent: `RetrievalOutcome.NOT_RELEVANT`; others: no signal
   - `FAILURE` → all experiences: `RetrievalOutcome.NOT_RELEVANT` (includes transient failures — see §4.4 trade-off note)
   - `GATE_REJECTED`, `GATE_EXPIRED`, `CANCELLED`, `OBSOLETE` → no signal
5. Call `cbrRetrievalTracker.feedback(feedbackEntries)` for each

**Threading:** `Instance<CbrRetrievalTracker>` with `isResolvable()` guard — transparent no-op when `memory-cbr-tracking` is not on the classpath. Observer is fired from `WorkflowExecutionCompletedHandler.fireStepOutcomeObserver()` (after the iteration fix from §4.4). Exception isolation — recording failure never blocks case progression.

### 7.2 Layer 2 — CBR outcome tracking (per-case)

Already wired. `CbrCaseRetainObserver` stores `ResolvedCase` entries on case terminal state. `CbrCaseMemoryStore.recordOutcome()` updates confidence via EMA. No new code needed.

### 7.3 Layer 3 — Selection feedback (per-judgment)

**When:** After a judgment binding resolves with a `ResolutionSelection`.

**How:** In the judgment completion path (triggered by `ACTION_GATE_APPROVED` or judgment CloudEvent):

1. Read `_candidates.<bindingName>` from the case context
2. Read the resolution (selected candidate ID)
3. For each candidate:
   - Selected → `CbrRetrievalTracker.feedback(caseId, HIGHLY_RELEVANT)`
   - Unselected, similarity ≥ threshold → `PARTIALLY_RELEVANT`
   - Unselected, similarity < threshold → no signal

The threshold is configurable via `casehub.cbr.selection-feedback.threshold` (default: `0.5`). The optimal threshold is domain-dependent — high-similarity domains (e.g., many similar SOC alerts) may need a higher threshold; diverse domains may need a lower one.
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
        → deterministic caseId = UUID.nameUUIDFromBytes(input.documentId().getBytes(UTF_8))
        → ResolutionGuide(problem, solution, outcome=null, confidence=null,
                          features=input.features(), trustScore=null, producerAgentId=null)
        → guide.withSteps(input.steps())
        → CbrCaseLifecycle.supersedeAll(List.of(deterministicCaseId), tenantId, "corpus re-ingestion")
          then CbrCaseStore.store(guide, caseType, entityId, domain, tenantId, deterministicCaseId, scope)
```

**Idempotent ingestion:** The deterministic `caseId` derived from `documentId` ensures that re-ingestion on application restart supersedes the existing entry rather than creating a duplicate. `supersedeAll(List.of(caseId), ...)` is the correct API for ID-based deduplication — it's a no-op if the caseId doesn't exist (first ingestion), and marks the old entry as superseded if it does (re-ingestion). Supersession is a soft-delete: the old entry is preserved for audit but excluded from retrieval. `CbrCaseStore.store()` reads features from `guide.features()` — with the `features` field added to `ResolutionGuide` (§4.1), extracted features are persisted correctly.

### 8.2 Change detection

```java
if (adapter.supportsChangeDetection()) {
    adapter.onChange(event -> {
        switch (event) {
            case CorpusChangeEvent.Added a -> ingest(a.input());
            case CorpusChangeEvent.Updated u -> update(u.documentId(), u.input());
            case CorpusChangeEvent.Removed r -> remove(r.documentId());
        }
    });
}
```

`Updated` uses `CbrCaseLifecycle.supersedeAll()` to mark the old entry and `CbrCaseStore.store()` to create the new one. `Removed` uses `CbrCaseStore.erase()`.

### 8.3 Feature extraction

The adapter is responsible for extracting features from documents. The engine provides no default feature extraction — each corpus has different feature dimensions.

```java
public class SocRunbookAdapter implements CorpusSourceAdapter {
    @Override
    public List<ResolutionGuideInput> discover(String tenancyId) {
        return loadRunbooks().stream()
            .map(doc -> new ResolutionGuideInput(
                doc.id(),
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

Note: `OutcomeWeightingCbrCaseMemoryStore` handles null confidence correctly — when `confidence()` is null (as it is for newly ingested `ResolutionGuide` entries), it defaults to `1.0`, yielding `score * 1.0 = score` (no penalty, no boost). New documents are ranked purely by similarity until they accumulate outcome feedback. There is no cold-start penalty.

## 9.1 Guided Execution Consumption

Agent workers consume resolution guidance through `RetrievedExperience.documentSteps()` and `RetrievedExperience.documentContent()`:

- **Automated path (agent workers):** The `experiences` list on `AgentRoutingContext` already includes `RetrievedExperience` entries — no new engine mechanism is needed to deliver document data to workers. However, worker implementations must be updated to consume `RESOLUTION_GUIDE` source types: current workers handle `planTrace` and `ExperiencePlanStep` fields only. When `sourceType=RESOLUTION_GUIDE`, the worker should include `documentSteps` as structured instructions and `automationHint` fields as implementation guidance in the LLM prompt. This is a worker-side change, not an engine change, and is tracked as part of child issue #6's testing scope (integration test must verify end-to-end with a worker that handles both source types).
- **Human path:** Human analysts see `documentContent` (prose solution) in the judgment task UI. `documentSteps` are rendered as a numbered checklist.
- **Hybrid path (LLM-as-judge):** LLM judgment workers receive the same `documentSteps` as automated workers, but the output goes through judgment resolution rather than direct execution.

## 9.2 Observability

Micrometer counters for the feedback pipeline:

| Metric | Tags | Description |
|--------|------|-------------|
| `casehub.cbr.feedback.retrieval` | `outcome={RELEVANT,NOT_RELEVANT}`, `caseType` | Layer 1 retrieval feedback events |
| `casehub.cbr.feedback.selection` | `outcome={HIGHLY_RELEVANT,PARTIALLY_RELEVANT}`, `caseType` | Layer 3 selection feedback events |
| `casehub.cbr.ingestion.documents` | `adapter`, `changeType={INITIAL,ADDED,UPDATED,REMOVED}` | Document ingestion events |
| `casehub.cbr.retrieval.sourceType` | `sourceType={PLAN_TRACE,RESOLUTION_GUIDE}`, `caseType` | Retrieval results by source type |
| `casehub.cbr.selection.candidate.count` | `bindingName`, `caseType` | Number of candidates presented per selection |

Log patterns:
- `INFO "Retrieval feedback recorded: caseId=%s outcome=%s experiences=%d"` — per feedback call
- `INFO "Selection feedback recorded: caseId=%s selected=%s candidates=%d"` — per selection
- `WARN "Retrieval feedback skipped: no experiences for caseId=%s binding=%s"` — when EventLog has no experiences

## 10. Child Issues

| # | Title | Scale | Complexity | Depends on |
|---|-------|-------|------------|------------|
| 0 | CBR naming cleanup (PlanCbrCase → ResolvedCase, TextualCbrCase → ResolutionGuide) | XS | Low | — |
| 1 | GuidanceStep on ResolutionGuide (with features field) + DocumentStep mapping | S | Low | #0 |
| 2 | RetrievedExperience extension (sourceType, documentContent, documentSteps) | S | Low | #1 |
| 3 | CbrRetrievalService mixed retrieval mapping | M | Med | #2 |
| 4 | CorpusSourceAdapter SPI + ResolutionIngestionService | M | Med | #1 |
| 5 | RetrievalFeedbackObserver (Layer 1 via StepOutcomeObserver) + handler fixes (iteration + DECLINED outcome mapping) + CbrRetrievalTracker.feedback() neocortex change | M | Med | #3, neocortex change |
| 6 | JudgmentTarget candidate presentation + ResolutionSelection | L | High | #3 |
| 7 | Selection feedback (Layer 3) | S | Med | #5, #6 |
| 8 | Outcome weighting default-on + documentation | XS | Low | — |

Issue #0 is a prerequisite, filed separately. Issues #1-8 are children of engine#1081.

Critical path: #0 → #1 → #2 → #3 → #6 (candidate presentation is the most complex and novel).

Parallel work: #4 (ingestion) can proceed independently after #1. #8 can land at any time.

## 11. Testing Strategy

### 11.1 Unit tests

- `RetrievalFeedbackObserver` — mock `CbrRetrievalTracker` and `EventLogRepository`, verify feedback calls for each `RoutingOutcome` value (SUCCESS, DECLINED, FAILURE, GATE_REJECTED, GATE_EXPIRED, CANCELLED, OBSOLETE)
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
- **Neocortex version dependency.** Engine must depend on a neocortex version that includes `GuidanceStep` and `features` on `ResolutionGuide`. This is a neocortex#1081-aligned change.

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
