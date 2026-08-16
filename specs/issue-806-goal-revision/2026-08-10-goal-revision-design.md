# Goal Revision — Modify Goal Parameters Based on Outcomes

**Issue:** engine#806
**Epic:** engine#800 (Sub-epic C — Goal Lifecycle Management)
**Depends on:** eidos#101 (goal querying — landed), neocortex#184 (agent experience stream — landed)

## Problem

Agent goals (`AgentGoal` on `AgentDescriptor`) are static once declared. As
agents accumulate experience — successful completions and failures across their
capabilities — goal descriptions and priorities should be revisable. A SECONDARY
goal with consistently high success rates should be promoted. A PRIMARY goal
with persistent failures should be demoted. Goal descriptions should be
refinable to better capture what the agent actually accomplishes.

The existing goal lifecycle infrastructure tracks signals but never acts on
them beyond abandonment:
- `GoalOutcomeRecorder` records DECLINE signals per goal via
  `BehavioralSignalStore` using a `__goal__` sentinel capability (a
  workaround — the signal store wasn't designed for goal outcomes)
- `GoalAbandonmentEvaluator` reads DECLINE counts from the same store
- `AgentGoalCompletionMarker` writes binary completion flags to case context
- `AgentExperienceRecorder` records structured outcome events and triggers
  reflection

Meanwhile, eidos-api already provides purpose-built goal evolution SPIs —
`GoalSignalStore`, `GoalEvolution`, `GoalOutcomeCounts` — that have zero
implementations and zero engine references. These were designed for exactly
this use case.

## Design Principles

1. **Use the existing eidos SPIs.** `GoalSignalStore` records per-goal
   outcomes (SUCCESS/FAILURE) with decay support. `GoalEvolution` evaluates
   descriptor + counts and produces `GoalEvolutionResult` (Evolved/Dampened/
   Unchanged). These implement the base-state + learned-state pattern from
   the epic-800 architecture: the descriptor carries declared identity,
   `GoalSignalStore` carries accumulated learned state, `GoalEvolution`
   merges them at evaluation time.

2. **Agent-level, not per-case.** Goal revision is an agent-level concern.
   Thresholds and strategy are configured via `@ConfigProperty` defaults.
   Per the epic-800 architecture spec: "Goal formation and revision are
   agent-level concerns."

3. **Per-goal error isolation.** An invalid LLM description for one goal
   does not prevent valid priority changes for other goals.

4. **Coordinated with abandonment.** `GoalAbandonmentEvaluator` currently
   uses `BehavioralSignalStore` with a `__goal__` sentinel. It should
   evolve to use `GoalSignalStore.outcomeCounts()` — querying FAILURE
   counts from the purpose-built store instead of DECLINE counts from
   the workaround. This is a targeted evolution, not a rewrite — the
   evaluator's threshold logic is unchanged.

## Architecture

Evaluator-per-completion pattern — `GoalRevisionEvaluator` is called from
`WorkflowExecutionCompletedHandler` on every worker completion (both success
and failure paths). It accumulates per-agent outcome state internally and
evaluates revision when configurable thresholds are met. Priority adjustment
delegates to `GoalEvolution` (eidos SPI). Description refinement delegates
to `GoalRevisionStrategy` (engine SPI). Changes are applied via
`AgentRegistry.register()` with an updated `AgentDescriptor`.

```
Worker completes (success or failure)
  -> WorkflowExecutionCompletedHandler
    -> GoalOutcomeRecorder.record()           [REPLACES GoalFailureRecorder]
        records GoalOutcome.SUCCESS or FAILURE in GoalSignalStore
    -> GoalRevisionEvaluator.record()          [NEW]
        1. Resolve workerName -> agentId via CaseDefinition.agentDescriptorFor()
        2. Accumulate per-agent outcome count + importance
        3. When threshold met -> spawn virtual thread:
           a. GoalSignalStore.outcomeCounts() -> Map<String, GoalOutcomeCounts>
           b. GoalEvolution.evaluate(descriptor, counts) -> GoalEvolutionResult
           c. If Evolved: GoalRevisionStrategy.revise(context) for descriptions
           d. Merge priority changes (from GoalEvolution) + descriptions (from strategy)
           e. AgentRegistry.register(updatedDescriptor)
           f. GoalSignalStore.clear() (reset signals post-evolution)
           g. Write GOAL_REVISED EventLog
           h. If Dampened: GoalSignalStore.decay(decayFactor)
           i. If Unchanged: no action
```

### Trigger mechanism

Per-agent threshold tracking using `ConcurrentHashMap<String, RevisionState>`
keyed by `agentId|tenancyId` — same pattern as
`AgentExperienceRecorder.ReflectionState`. Counters accumulate on every worker
completion. When either `outcomeCount >= minOutcomes` or
`cumulativeImportance >= importanceThreshold`, counters reset and evaluation
runs on a virtual thread.

Thresholds are configured via `@ConfigProperty` with sensible defaults.

### v1 trigger simplification

The epic-800 architecture spec triggers goal evolution from
`@ObservesAsync ReflectionRecorded` — after reflection synthesizes insights.
This spec triggers from raw worker completion outcomes (threshold-based).
This is a deliberate v1 simplification: reflection is not always enabled,
and per-completion triggering provides more responsive goal tuning. Future
work can add a reflection-triggered evaluation path that calls the same
`GoalEvolution` SPI.

## Components

### GoalOutcomeRecorder (replaces GoalFailureRecorder)

`runtime/internal/routing/`, `@ApplicationScoped`. Replaces
`GoalOutcomeRecorder` with purpose-built `GoalSignalStore` integration.

**Changes:**
1. Replace `Instance<BehavioralSignalStore>` with `Instance<GoalSignalStore>`
2. Replace `BehavioralSignal.DECLINE` with `GoalOutcome.FAILURE`
3. Add SUCCESS recording: `WorkerOutcome.Success`/`Completed` ->
   `GoalOutcome.SUCCESS`
4. Call `GoalSignalStore.recordOutcome(agentId, tenancyId, goalName, outcome)`
5. **Null capability guard:** skip recording when `capabilityName` is null
6. Add call on success path in `WorkflowExecutionCompletedHandler` (currently
   only called on failure path)
7. Rename class and test class

`GoalAbandonmentEvaluator` evolves to use `GoalSignalStore.outcomeCounts()`
— queries `failureCount` from `GoalOutcomeCounts` instead of
`BehavioralSignalStore.count()` with `__goal__` sentinel. The threshold
logic is unchanged.

### GoalRevisionEvaluator (new)

`runtime/internal/routing/`, `@ApplicationScoped`.

**Injected dependencies:**
- `Instance<GoalSignalStore>` — query per-goal outcome counts
- `Instance<GoalEvolution>` — evaluate priority adjustment
- `Instance<AgentRegistry>` — read current descriptor, write updated one
- `CaseDefinitionRegistry` — resolve CaseDefinition, look up AgentDescriptor
- `EngineStrategyResolver` — resolve GoalRevisionStrategy by ID
- `EventLogRepository` — write GOAL_REVISED audit entries

**Method:** `record(CaseInstance, String workerName, String capabilityName,
WorkerOutcome<?>)`

**workerName -> agentId resolution:** Same path as `GoalOutcomeRecorder` —
`CaseDefinitionRegistry.getCaseDefinition()` then
`definition.agentDescriptorFor(workerName)`. Early return when no descriptor
or no goals.

**Internal state:**

```java
private static class RevisionState {
    int outcomeCount;
    double cumulativeImportance;
    Instant lastRevisionTime;
}
```

**Trigger evaluation** (inside `ConcurrentHashMap.compute()` — atomic):
1. Resolve agentId from workerName via CaseDefinitionRegistry
2. Increment outcomeCount and cumulativeImportance on RevisionState
3. If `outcomeCount >= minOutcomes` OR
   `cumulativeImportance >= importanceThreshold` -> trigger
4. Reset counters, record lastRevisionTime
5. Spawn virtual thread for evaluation

**Importance mapping:** Uses `WorkerOutcome` variant names via
`outcomeKindName()` — same pattern as `AgentExperienceRecorder`:
SUCCESS->0.3, COMPLETED->0.3, DECLINED->0.6, FAILED->0.8, EXPIRED->0.5.

**Evaluation logic** (on virtual thread):
1. `GoalSignalStore.outcomeCounts(agentId, tenancyId)` ->
   `Map<String, GoalOutcomeCounts>`
2. `AgentRegistry.findById(agentId, tenancyId)` -> current descriptor
3. `GoalEvolution.evaluate(descriptor, counts)` -> `GoalEvolutionResult`
4. Switch on result:
   - **Evolved(newGoals, promotedGoals, demotedGoals):**
     a. If `GoalRevisionStrategy` resolvable, build `GoalRevisionContext`
        with newGoals and counts, call `strategy.revise(context)` for
        description refinements
     b. Apply description changes per-goal with error isolation: validate
        against `AgentDescriptorValidator` constraints, discard invalid
        descriptions with warning
     c. Build final goals list (GoalEvolution priorities + valid LLM
        descriptions merged)
     d. `descriptor.toBuilder().goals(finalGoals).build()` ->
        `agentRegistry.register(updatedDescriptor)`
     e. `goalSignalStore.clear(agentId, tenancyId)` — reset signals
        (same pattern as DispositionSignalStore on evolution acceptance)
     f. Write GOAL_REVISED EventLog
   - **Dampened(decayFactor):**
     a. `goalSignalStore.decay(agentId, tenancyId, decayFactor)`
   - **Unchanged:** no action

**Concurrency:** Per-agent `ReentrantLock` via `ConcurrentHashMap<String,
ReentrantLock>` keyed by `agentId|tenancyId`. Acquired before step 1,
released in finally block.

**Guard rails:**
- `isResolvable()` guard on `GoalSignalStore`, `GoalEvolution`,
  `AgentRegistry` — transparent no-op when any absent
- `casehub.engine.goal.revision.enabled` master switch (default false)
- All exceptions caught and logged — never blocks case progression
- Strategy failure -> priority-only revision still applies (GoalEvolution
  result is independent of strategy)
- Per-goal error isolation for description validation

### GoalRevisionStrategy SPI (engine-api)

`io.casehub.api.spi.routing`, alongside existing routing-adjacent strategies.

```java
public interface GoalRevisionStrategy extends NamedStrategy {
    Uni<GoalRevisionProposal> revise(GoalRevisionContext context);

    @Override
    default String id() { return "llm"; }
}
```

**GoalRevisionContext** — carries ALL goals and their counts. The strategy
sees the full goal landscape.

```java
public record GoalRevisionContext(
    String agentId,
    String tenancyId,
    List<AgentGoal> goals,
    Map<String, GoalOutcomeCounts> counts,
    CaseDefinition definition
)
```

Uses `GoalOutcomeCounts` from eidos-api directly — no engine-side metrics
type needed.

**GoalRevisionProposal** — per-goal proposed description changes:

```java
public record GoalRevisionProposal(
    List<RevisedGoal> revisions,
    String rationale
) {
    public record RevisedGoal(
        String goalName,
        String revisedDescription,    // nullable - null means no change
        String revisionReason
    ) {}
}
```

No `revisedPriority` field — priority adjustment is handled by
`GoalEvolution` (eidos). The strategy only proposes description refinements.
Clean separation: eidos owns priority logic, engine owns description
refinement.

### LlmGoalRevisionStrategy (runtime)

`runtime/internal/routing/`, `@ApplicationScoped`, id=`"llm"`.

**Injected:** `Instance<ChatModelProvider>` — transparent no-op when absent.

**Prompt construction:**
- System: "You are a goal effectiveness analyst. Given an agent's goals
  and their performance metrics, evaluate whether any goal descriptions
  should be refined to better capture what the agent accomplishes. Only
  propose changes when a description is meaningfully misaligned with
  observed outcomes."
- User: all goals with names, descriptions, priorities, and effectiveness
  metrics (success rate, failure rate, total outcomes from GoalOutcomeCounts).
- Response schema (enforced via structured output):

```json
{
  "type": "object",
  "properties": {
    "revisions": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "goalName": { "type": "string" },
          "revisedDescription": { "type": ["string", "null"] },
          "revisionReason": { "type": "string" }
        },
        "required": ["goalName", "revisionReason"]
      }
    },
    "rationale": { "type": "string" }
  },
  "required": ["revisions", "rationale"]
}
```

**Error handling:**
- ChatModelProvider absent -> `Uni.createFrom().failure(new
  UnsupportedOperationException(...))`
- Invalid JSON -> `Uni.createFrom().failure(new AgentException(...))`
- Empty revisions -> valid result (no changes needed)

## Configuration

Goal revision is agent-level. Configuration uses `@ConfigProperty` with
sensible defaults.

| Property | Default | Description |
|----------|---------|-------------|
| `casehub.engine.goal.revision.enabled` | `false` | Master switch |
| `casehub.engine.goal.revision.strategy` | `"llm"` | GoalRevisionStrategy ID |
| `casehub.engine.goal.revision.min-outcomes` | `10` | Minimum outcomes before evaluation |
| `casehub.engine.goal.revision.importance-threshold` | `3.0` | Cumulative importance to trigger |

Priority thresholds (promotion rate, demotion rate) are owned by the
`GoalEvolution` implementation in eidos — not engine config. The eidos
implementation uses its own preference keys for these thresholds, following
the `DispositionPreferenceKeys` pattern.

## Cross-repo Dependencies

### eidos-api — existing SPIs, need implementations

The following SPIs exist in eidos-api but have no implementations:
- `GoalSignalStore` — needs `InMemoryGoalSignalStore` (@DefaultBean) and
  `NoOpGoalSignalStore` (for test fallback)
- `GoalEvolution` — needs `DefaultGoalEvolution` (@DefaultBean) with
  threshold-based priority adjustment

These implementations should be created in eidos-api (NoOp/in-memory
defaults, following `DispositionSignalStore` / `DispositionEvolution`
precedent) as a prerequisite for this issue. File an eidos issue.

### eidos-api — AgentGoal.toBuilder()

`GoalEvolution.evaluate()` returns `Evolved(List<AgentGoal> newGoals)` —
the evolution implementation builds the new `AgentGoal` instances. For
description merging in the engine, we need to create modified copies of
goals from the evolved list with updated descriptions. `AgentGoal` has no
`toBuilder()`. Add one (small change, follows `AgentDescriptor.toBuilder()`
pattern).

## EventLog Audit

New event type: `CaseHubEventType.GOAL_REVISED`

One EventLog entry per evaluation cycle.

**Metadata:**

| Key | Type | Description |
|-----|------|-------------|
| `agentId` | String | Which agent's goals were revised |
| `evolutionResult` | String | `EVOLVED`, `DAMPENED`, or `UNCHANGED` |
| `strategyId` | String | Which GoalRevisionStrategy was used (nullable) |
| `promotedGoals` | List | Goal names promoted SECONDARY -> PRIMARY |
| `demotedGoals` | List | Goal names demoted PRIMARY -> SECONDARY |
| `descriptionRevisions` | List | Goals with description changes |
| `descriptionRevisions[].goalName` | String | Goal name |
| `descriptionRevisions[].previousDescription` | String | Before |
| `descriptionRevisions[].newDescription` | String | After |
| `descriptionRevisions[].reason` | String | LLM rationale |
| `counts` | Object | GoalOutcomeCounts snapshot at revision time |
| `totalGoalsEvaluated` | int | Goals examined |
| `totalGoalsRevised` | int | Goals changed |

## Module Placement

| Type | Module | Rationale |
|------|--------|-----------|
| `GoalRevisionStrategy` | engine-api | Consumer-implementable NamedStrategy SPI |
| `GoalRevisionContext`, `GoalRevisionProposal` | engine-api | SPI parameter/result types |
| `GoalOutcomeRecorder` | runtime | Replaces GoalFailureRecorder |
| `GoalRevisionEvaluator` | runtime | Execution infrastructure |
| `LlmGoalRevisionStrategy` | runtime | Built-in strategy implementation |
| `GOAL_REVISED` event type | common | Event constant |
| `GoalSignalStore`, `GoalEvolution`, etc. | eidos-api | Existing (need implementations) |

## Testing

### Unit tests

1. **`GoalOutcomeRecorderTest`** (replaces GoalFailureRecorderTest):
   - SUCCESS outcome records `GoalOutcome.SUCCESS` via GoalSignalStore
   - COMPLETED outcome records `GoalOutcome.SUCCESS` via GoalSignalStore
   - DECLINED/FAILED/EXPIRED record `GoalOutcome.FAILURE`
   - Capability filtering: only goals matching capabilityName get recorded
   - Null capabilityName skips recording entirely
   - No GoalSignalStore -> no-op

2. **`GoalRevisionEvaluatorTest`** — core logic:
   - Skips when GoalSignalStore/GoalEvolution/AgentRegistry not resolvable
   - Skips when revision not enabled
   - workerName -> agentId resolution via CaseDefinition.agentDescriptorFor()
   - Early return when no descriptor or no goals
   - Accumulates outcomes, does not trigger below threshold
   - Triggers when outcomeCount >= minOutcomes
   - Triggers when cumulativeImportance >= importanceThreshold
   - Resets counters after trigger
   - Calls GoalEvolution.evaluate() with descriptor and counts
   - On Evolved: calls GoalRevisionStrategy for description refinement
   - On Evolved: applies via AgentRegistry.register() with merged goals
   - On Evolved: clears GoalSignalStore signals
   - On Dampened: calls GoalSignalStore.decay()
   - On Unchanged: no action
   - Filters strategy proposals: ignores unknown goal names
   - Per-goal error isolation: invalid LLM description discarded, other
     goals unaffected
   - Strategy failure -> priority-only revision from GoalEvolution
   - Does NOT call register when GoalEvolution returns Unchanged
   - Writes GOAL_REVISED EventLog with correct metadata
   - Per-agent lock prevents concurrent revision
   - All exceptions caught — never blocks case progression

3. **`GoalAbandonmentEvaluatorTest`** — evolution:
   - Uses GoalSignalStore.outcomeCounts() instead of BehavioralSignalStore
   - Threshold logic unchanged (count >= threshold)
   - No GoalSignalStore -> transparent no-op (same as before)

4. **`GoalRevisionProposalTest`**:
   - Record validation, null handling

5. **`LlmGoalRevisionStrategyTest`**:
   - Produces GoalRevisionProposal from structured LLM response
   - Receives all goals in context
   - No-op when ChatModelProvider absent
   - Invalid JSON -> Uni failure
   - Empty revisions when no changes needed

### Integration test

6. **`GoalRevisionIntegrationTest`** (`@QuarkusTest`):
   - Full flow: case with agent goals -> multiple worker completions ->
     threshold met -> GoalEvolution returns Evolved -> descriptions
     refined via mock ChatModelProvider -> AgentRegistry updated ->
     EventLog contains GOAL_REVISED
   - Verify promoted/demoted goals match GoalEvolution result
   - Verify description refinements applied from LlmGoalRevisionStrategy
   - Verify GoalSignalStore cleared after evolution
   - Verify Dampened result triggers decay

### eidos-api tests (cross-repo prerequisite)

7. **`InMemoryGoalSignalStoreTest`**:
   - recordOutcome increments counts
   - outcomeCounts returns correct per-goal counts
   - decay reduces counts
   - clear resets all counts

8. **`DefaultGoalEvolutionTest`**:
   - Promotes SECONDARY -> PRIMARY when success rate exceeds threshold
   - Demotes PRIMARY -> SECONDARY when failure rate exceeds threshold
   - Returns Unchanged when rates are between thresholds
   - Returns Dampened when below minimum outcomes (not ready)
   - Skips goals with zero outcomes

## Scope Boundaries

**In scope:**
- GoalOutcomeRecorder (replaces GoalFailureRecorder with GoalSignalStore)
- GoalAbandonmentEvaluator evolution (GoalSignalStore instead of
  BehavioralSignalStore + sentinel)
- GoalRevisionEvaluator with threshold-based trigger
- GoalRevisionStrategy SPI + LlmGoalRevisionStrategy (description only)
- GoalRevisionContext, GoalRevisionProposal
- Priority adjustment via GoalEvolution (eidos SPI)
- EventLog audit (GOAL_REVISED)
- Agent-level config via @ConfigProperty
- Cross-repo: GoalSignalStore + GoalEvolution implementations in eidos-api
- Cross-repo: AgentGoal.toBuilder() in eidos-api

**v1 constraints (deliberate):**
- Per-completion trigger (not reflection-triggered)
- Single strategy selection via global config
- No capability list or visibility revision
- No goal parameter rollback mechanism
- No cascading revision

**Out of scope (future work):**
- Reflection-triggered evaluation (`@ObservesAsync ReflectionRecorded`)
- GoalSignalProvider evolution to use GoalSignalStore
  (GoalAbandonmentEvaluator covers the routing-time query for now)
- Capability list revision
- Goal parameter rollback / undo
- Cross-agent goal coordination
- PreferenceStore for per-tenancy threshold overrides
- GoalSignalStore JPA implementation (eidos-runtime)

## Review Findings Addressed

| Finding | Source | Resolution |
|---------|--------|------------|
| Descriptor mutation contradicts GoalLifecycleStore | Str R1-01 | GoalSignalStore IS the learned-state store; GoalEvolution merges at evaluation time; descriptor mutation via register() is the apply step |
| GoalAbandonmentEvaluator interaction | Coh R1-05, Str R1-02, Rob R1-05 | Evolve to use GoalSignalStore; GoalEvolution.evaluate() sees full counts including failure data |
| Per-case config contradicts epic-800 | Str R1-03 | Agent-level @ConfigProperty; priority thresholds owned by GoalEvolution in eidos |
| Cumulative signal counts | Rob R1-01 | GoalSignalStore.clear() after evolution; GoalSignalStore.decay() on Dampened |
| Context/Proposal cardinality mismatch | Coh R1-03, Rob R1-06 | Context carries all goals; strategy returns subset |
| workerName -> agentId resolution | Coh R1-04 | Explicit resolution via CaseDefinition.agentDescriptorFor() |
| Cross-case config ambiguity | Rob R1-02 | Eliminated: agent-level config |
| Read-modify-write race | Rob R1-03 | Per-agent ReentrantLock; register() is the final step after all computation |
| LLM validation failure loses changes | Rob R1-04 | Per-goal error isolation; priority from GoalEvolution is independent |
| Config.enabled redundancy | Coh R1-06 | Single @ConfigProperty master switch |
| Null capability inflates counts | Rob R1-07 | GoalOutcomeRecorder skips when capabilityName null |
| bindingName unused | Coh R1-08 | Removed from signature |
| recentOutcomeDescriptions redundant | Coh R1-09 | Removed; GoalOutcomeCounts carries the data |
| SPI package doesn't exist | Str R1-05 | Placed in io.casehub.api.spi.routing |
| Importance weight inconsistency | Coh R1-07 | Uses outcomeKindName() pattern |
| Bypasses reflection pipeline | Str R1-07 | Documented as v1 simplification |
| Split state pattern | Str R1-04 | Accepted: established pattern |
| Handler god class | Str R1-06 | Acknowledged: not addressed in this issue |
