# Agent Experience Recording & Memory-Informed Dispatch

**Issues:** engine#801, engine#804
**Epic:** engine#800 (Sub-epic B — Agent Reflection & Planning)
**Depends on:** neocortex#184 (ExperienceStream), neocortex#186 (ReflectionService), engine#800 commit `a0d3acd` (ExperienceRecorder + ReflectionOrchestrator SPIs)

## Problem

The engine records worker outcomes in EventLog and CBR stores, but has no
agent-level memory — agents cannot accumulate experiences across cases or
synthesize higher-level insights from those experiences. At dispatch time,
agents receive case context and CBR-retrieved past case outcomes, but not
their own memories of prior work.

Two gaps:
1. **No experience recording or reflection trigger** — worker completions
   produce EventLog entries and CBR retain, but no agent-scoped memory
   stream. Neocortex provides the SPIs (`ExperienceRecorder`,
   `ReflectionOrchestrator`); the engine does not call them.
2. **No memory retrieval at dispatch** — `WorkerContext` carries
   `experiences` (CBR plan traces) but not agent memories. Workers
   cannot access their own past experiences or reflection insights
   when making decisions.

## Design

### Configuration Model

Two new config records on `CaseDefinition`, following the `CbrConfig` pattern.

#### ReflectionTriggerConfig (`api/model/`)

```java
public record ReflectionTriggerConfig(
    boolean enabled,
    double importanceThreshold,
    int maxUnreflectedOutcomes,
    int maxSourceMemories,
    Map<String, Double> importanceWeights
)
```

- `enabled` — master switch (default false)
- `importanceThreshold` — cumulative importance since last reflection
  that triggers synthesis. Each `ExperienceEvent.importance()` is 0.0–1.0;
  threshold 3.0 means roughly 3–6 outcomes. Range [0.0, 10.0].
- `maxUnreflectedOutcomes` — hard ceiling: reflect after this many outcomes
  regardless of importance. Safety net for low-importance streams. Min 1.
- `maxSourceMemories` — bound on raw memories passed to
  `ReflectionOrchestrator.reflect()`. Min 1.
- `importanceWeights` — per-outcome-type importance values, keyed by
  `WorkerOutcome` variant name. Each value in [0.0, 1.0]. Allows case
  definitions to customise how heavily each outcome type contributes
  toward the reflection threshold.

Defaults: `enabled=false, importanceThreshold=3.0, maxUnreflectedOutcomes=10, maxSourceMemories=50, importanceWeights={"SUCCESS": 0.3, "COMPLETED": 0.3, "DECLINED": 0.6, "FAILED": 0.8, "EXPIRED": 0.5}`.

YAML:
```yaml
spec:
  reflection:
    enabled: true
    importanceThreshold: 3.0
    maxUnreflectedOutcomes: 10
    maxSourceMemories: 50
    importanceWeights:
      SUCCESS: 0.3
      COMPLETED: 0.3
      DECLINED: 0.6
      FAILED: 0.8
      EXPIRED: 0.5
```

#### MemoryRetrievalConfig (`api/model/`)

```java
public record MemoryRetrievalConfig(
    boolean enabled,
    int maxMemories,
    Set<String> domains
)
```

- `enabled` — master switch (default false)
- `maxMemories` — maximum memories returned per dispatch. Min 1.
- `domains` — which memory domains to query (e.g. "experience",
  "reflection", "relationship"). Empty set means all domains.

Defaults: `enabled=false, maxMemories=10, domains={"experience", "reflection"}`.

YAML:
```yaml
spec:
  memoryRetrieval:
    enabled: true
    maxMemories: 10
    domains: [experience, reflection]
```

#### CaseDefinition additions

- `reflectionTrigger` (nullable `ReflectionTriggerConfig`) — builder:
  `.reflectionTrigger(ReflectionTriggerConfig)`
- `memoryRetrieval` (nullable `MemoryRetrievalConfig`) — builder:
  `.memoryRetrieval(MemoryRetrievalConfig)`

Both nullable — null means disabled (no config needed for cases that
don't use agent memory).

### AgentExperienceRecorder

**Location:** `runtime/src/main/java/io/casehub/engine/internal/memory/AgentExperienceRecorder.java`
**CDI:** `@ApplicationScoped`

Records worker outcomes as agent experience events and evaluates
reflection triggers.

#### Injection

- `Instance<ExperienceRecorder>` — neocortex SPI, `isResolvable()` guard
- `Instance<ReflectionOrchestrator>` — neocortex SPI, `isResolvable()` guard
- `CaseDefinitionRegistry` — lookup `ReflectionTriggerConfig`

Transparent no-op when neocortex-memory is not on the classpath.

#### Recording

Called from `WorkflowExecutionCompletedHandler` at the same call sites as
`personalitySignalRecorder.record()` and `goalFailureRecorder.record()`:
- Success path: `onWorkflowExecutionCompletedHandler()` after `recordSuccessOutcome()`
- Failure path: `handleSemanticFailure()`

Method signature:
```java
public void record(CaseInstance caseInstance, String workerName,
                   String capabilityName, WorkerOutcome<?> outcome,
                   String bindingName)
```

Builds an `Outcome` event:
- `agentId` = workerName
- `tenantId` = caseInstance.tenancyId
- `caseId` = caseInstance.id.toString()
- `turnId` = UUID.randomUUID().toString() (unique per execution)
- `description` = "Completed {capabilityName}" (success) or
  "Failed {capabilityName}: {reason}" (failure)
- `importance` = looked up from `ReflectionTriggerConfig.importanceWeights`
  using the outcome kind name (e.g. "SUCCESS", "DECLINED"). When config
  is null (feature disabled), falls back to default map:
  SUCCESS → 0.3, COMPLETED → 0.3, DECLINED → 0.6, FAILED → 0.8, EXPIRED → 0.5.
  (`Completed` is treated identically to `Success` by the engine — same
  default importance. The five `WorkerOutcome` variants form an exhaustive
  sealed hierarchy; all five must be handled.)
- `result` = outcome kind name (e.g. "SUCCESS", "COMPLETED", "DECLINED")
- `capability` = capabilityName
- `metadata` = `{bindingName, caseDefinitionName}`

Calls `experienceRecorder.record(outcomeEvent)`. All exceptions caught
and logged — never blocks case progression.

#### Reflection trigger evaluation

After recording, evaluates `CaseDefinition.reflectionTrigger`:

**Tracking state:** Per-agent `ConcurrentHashMap<String, ReflectionState>`
keyed by `agentId + "|" + tenantId`. `ReflectionState` is a mutable holder:
`outcomeCount`, `cumulativeImportance`, `lastReflectionTime`. JVM-local,
reset on restart.

**Restart impact:** Counter state is lost on JVM restart. Consequences:
- Partially accumulated importance/count resets to zero — the next
  reflection is **delayed** (requires a full new cycle of outcomes, not
  "one extra" reflection)
- `lastReflectionTime` resets to null — the next
  `ReflectionOrchestrator.reflect()` call passes `null` for `since`,
  causing the orchestrator to process all memories up to
  `maxSourceMemories`. This is bounded but may overlap with prior
  reflections. Acceptable: reflection is idempotent (duplicate insights
  are deduplicated at the memory store level), and `maxSourceMemories`
  caps the query cost.

**Evaluation (atomic via `ConcurrentHashMap.compute()`):**

`WorkflowExecutionCompletedHandler` runs `@RunOnVirtualThread` — the
same agent can have concurrent completions across different cases. All
counter operations must be atomic to prevent double-firing.

1. Look up `ReflectionTriggerConfig` from `CaseDefinition`
2. If null or `!enabled` → return
3. Atomically within `reflectionStates.compute(key, ...)`:
   - Increment `outcomeCount`, add importance to `cumulativeImportance`
   - If `outcomeCount >= maxUnreflectedOutcomes` OR
     `cumulativeImportance >= importanceThreshold`:
     - Capture `since = lastReflectionTime`
     - Reset `outcomeCount` and `cumulativeImportance` to zero
     - Set `lastReflectionTime` to now
     - Set `shouldReflect` flag (captured in closure-local variable)
4. Outside `compute()`, if `shouldReflect`:
   - Fire reflection on virtual thread (non-blocking):
     `reflectionOrchestrator.reflect(workerName, tenantId, since, maxSourceMemories)`

Virtual thread execution follows the `CaseOutcomeObserver` pattern —
`@RunOnVirtualThread` is not used; instead a virtual thread is spawned
directly for the reflection call. Exceptions caught and logged.

### AgentMemoryRetriever

**Location:** `runtime/src/main/java/io/casehub/engine/internal/memory/AgentMemoryRetriever.java`
**CDI:** `@ApplicationScoped`

Queries agent memories at dispatch time.

#### Injection

- `Instance<CaseMemoryStore>` — neocortex SPI, `isResolvable()` guard
- `CaseDefinitionRegistry` — lookup `MemoryRetrievalConfig`

Transparent no-op when neocortex-memory is not on the classpath.

#### Retrieval

Method signature:
```java
public List<RetrievedMemory> retrieve(String workerName, String tenantId,
                                       String capabilityName,
                                       CaseDefinition caseDefinition)
```

1. Look up `CaseDefinition.memoryRetrieval`
2. If null or `!enabled` → return `List.of()`
3. Calculate per-domain allocation:
   `perDomainLimit = max(1, config.maxMemories / domainCount)`
4. For each domain in `config.domains` (or all if empty):
   - `CaseMemoryStore.query(
       MemoryQuery.forEntity(workerName, new MemoryDomain(domain), tenantId)
           .withQuestion(capabilityName)
           .withLimit(perDomainLimit)
           .withOrder(MemoryOrder.SALIENCE))`

   Uses `MemoryQuery.forEntity()` factory which wraps `workerName` in
   `List.of(workerName)` for the `entityIds` field. The `with*` builders
   override defaults: `withLimit()` sets the record's `limit` field (not
   `topK`), `withOrder()` overrides the default `CHRONOLOGICAL` ordering.

5. Interleave results round-robin across domains to ensure domain
   diversity: take 1 from domain 0, 1 from domain 1, ..., repeat until
   `maxMemories` reached or all domain results exhausted. This preserves
   per-domain salience ordering while guaranteeing each domain has
   representation in the final result.
6. Map `Memory` → `RetrievedMemory`
7. Return

All exceptions caught and logged — returns `List.of()` on failure.
Memory retrieval failure never blocks dispatch.

### RetrievedMemory

**Location:** `api/src/main/java/io/casehub/api/model/RetrievedMemory.java`

Engine-owned read model mapping from neocortex `Memory`:

```java
public record RetrievedMemory(
    String memoryId,
    String text,
    String domain,
    Instant createdAt,
    Map<String, String> attributes
)
```

Validated: `memoryId`, `text`, `domain` non-null. `attributes` defaults
to empty map, made immutable via `Map.copyOf()`.

### WorkerContext.memories

`WorkerContext` gains an 8th field:

```java
public record WorkerContext(
    String taskDescription,
    UUID caseId,
    List<CaseChannel> channels,
    List<WorkerSummary> priorWorkers,
    PropagationContext propagationContext,
    Map<String, Object> properties,
    List<RetrievedExperience> experiences,
    List<RetrievedMemory> memories
)
```

Compact constructor: `memories = memories == null ? List.of() : List.copyOf(memories)`.
Backward-compatible 7-arg constructor passes `List.of()` for memories.
Backward-compatible 6-arg constructor unchanged (chains to 7-arg).

### Threading through the dispatch path

Unlike `experiences` (which arrive pre-populated on the `WorkerScheduleEvent`
from the routing layer), memories are retrieved at scheduling time by the
handler itself — the routing layer has no memory awareness. However,
memories follow the **same serialization path** as experiences once
retrieved: they travel through EventLog metadata, not the Quartz job
data map.

1. `WorkerScheduleEventHandler.onWorkerScheduleEventHandler()`:
   - After input projection and bridge resolution, before `buildEventLog()`
   - Calls `agentMemoryRetriever.retrieve(worker.name(), instance.tenancyId, capability.name(), caseDefinition)`
   - Passes `List<RetrievedMemory>` to `buildEventLog()` which serializes
     the full list as JSON into EventLog metadata under key `"memories"`
     (same pattern as `experiences` on line 249–251 of
     `WorkerScheduleEventHandler.buildEventLog()`)
   - Additionally stores `retrievedMemoryCount` (int) for lightweight
     audit queries without JSON deserialization (e.g. "show dispatches
     with zero memories", monitoring dashboards)

**Sizing note:** This is the same pattern used for `experiences` (line
249–251 of `WorkerScheduleEventHandler.buildEventLog()`), where each
`RetrievedExperience` carries 8 fields including `planTrace` (a
potentially unbounded list of `ExperiencePlanStep` records with 8 fields
each). `RetrievedMemory` is lighter (5 fields; `text` is typically
sentence-to-paragraph length per neocortex CaseMemoryStore contract).
With `maxMemories=10`, the serialized JSON is comparable to or smaller
than the existing experiences payload.

**Auditability rationale:** Persisting the full retrieved memory list in
EventLog metadata is intentional — in regulated environments (AML,
clinical), knowing exactly which memories informed a worker's decision
is a compliance requirement. The `retrievedMemoryCount` field is not
redundant with the serialized list: count enables O(1) filtering and
aggregation queries on EventLog metadata without deserializing the
JSON array.

2. `QuartzWorkerExecutionJob`:
   - Adds `deserializeMemories(EventLog)` method parallel to existing
     `deserializeExperiences(EventLog)` (lines 282–298)
   - Reads from `eventLog.getMetadata().get("memories")`
   - Deserializes to `List<RetrievedMemory>` using `ObjectMapper.convertValue()`
   - Constructs 8-arg `WorkerContext` with both `experiences` and
     `memories`

3. Workers access via `((WorkerRuntime) scope).context().memories()`.

### YAML mapping

`CaseDefinitionYamlMapper` additions — read from raw `JsonNode` (not
generated schema classes), following the same pattern used for
`routingSignalWeights`, `authorization`, and `quorum` (lines 721–789
of the mapper):

```java
// reflection — read from raw spec node
final JsonNode reflectionNode = specNode != null ? specNode.get("reflection") : null;
if (reflectionNode != null && reflectionNode.isObject()) {
    Map<String, Double> weights = ReflectionTriggerConfig.DEFAULT_IMPORTANCE_WEIGHTS;
    JsonNode weightsNode = reflectionNode.get("importanceWeights");
    if (weightsNode != null && weightsNode.isObject()) {
        weights = new LinkedHashMap<>();
        var it = weightsNode.fields();
        while (it.hasNext()) { var e = it.next(); weights.put(e.getKey(), e.getValue().asDouble()); }
    }
    def.setReflectionTrigger(new ReflectionTriggerConfig(
        reflectionNode.has("enabled") && reflectionNode.get("enabled").asBoolean(),
        reflectionNode.has("importanceThreshold") ? reflectionNode.get("importanceThreshold").asDouble() : 3.0,
        reflectionNode.has("maxUnreflectedOutcomes") ? reflectionNode.get("maxUnreflectedOutcomes").asInt() : 10,
        reflectionNode.has("maxSourceMemories") ? reflectionNode.get("maxSourceMemories").asInt() : 50,
        Map.copyOf(weights)));
}
```

Same pattern for `memoryRetrieval:` → `MemoryRetrievalConfig`.

Both are optional — missing block means null on `CaseDefinition`.
No generated schema classes (`io.casehub.model.*`) needed.

### No-op defaults

Both beans use `Instance<>.isResolvable()` — when neocortex is not on
the classpath, recording returns immediately and retrieval returns
`List.of()`. No `@DefaultBean` no-op implementations needed for the
neocortex SPIs — they already have their own defaults.

## Out of scope

- **LLM-backed ReflectionSynthesizer** — neocortex concern; uses
  `NoOpReflectionSynthesizer` until a real implementation ships
- **Scheduled/idle reflection triggers** — future extension; the
  `ReflectionOrchestrator.reflect()` call is the stable interface
- **Memory retrieval for humanTask routing** — future; parallel to
  `HumanTaskRoutingStrategy` CBR path
- **Agent prompt template integration** — consumer concern; workers
  access memories via `WorkerContext.memories()`
- **Sub-epic C goal lifecycle** — separate branch scope

## Test plan

| Test class | What it tests |
|---|---|
| `ReflectionTriggerConfigTest` | Validation: thresholds, bounds, defaults factory; importanceWeights immutability, unknown outcome key rejected, values clamped to [0.0, 1.0] |
| `MemoryRetrievalConfigTest` | Validation: maxMemories, domains immutability, defaults factory |
| `RetrievedMemoryTest` | Null validation, attributes immutability |
| `WorkerContextTest` | 8-arg constructor, 7-arg backward compat, memories immutability |
| `AgentExperienceRecorderTest` | Records outcome → calls ExperienceRecorder; importance from config importanceWeights per outcome type (all 5: Success, Completed, Declined, Failed, Expired); falls back to defaults when config is null; custom importanceWeights override defaults; reflection trigger fires at threshold; reflection trigger fires at count ceiling; concurrent completions for same agent do not double-fire (compute atomicity); no-op when neocortex unavailable; exceptions caught |
| `AgentMemoryRetrieverTest` | Retrieves per config; multi-domain round-robin interleaving; per-domain allocation respects maxMemories; truncation to maxMemories; returns empty when disabled; returns empty when neocortex unavailable; exceptions caught |
| `CaseDefinitionYamlMapperTest` | Parses reflection block; parses memoryRetrieval block; parses importanceWeights from reflection block; missing importanceWeights → defaults; missing blocks → null |
| Integration test | End-to-end: case with config → worker completes → experience recorded → reflection triggered; dispatch includes memories |
