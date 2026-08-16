# Goal Formation — Agents Discover New Goals from Experience

**Issue:** engine#805, engine#808 (merged — mechanism + trigger)
**Epic:** engine#800 (Sub-epic C — Goal Lifecycle Management)
**Depends on:** engine#801 (reflection orchestration — landed), neocortex#186 (reflective diary — landed)

## Problem

Agent goals (`AgentGoal` on `AgentDescriptor`) are static once declared.
The engine can revise existing goals (#806 — priority adjustment,
description refinement) and abandon infeasible ones (#807 — threshold-
based abandonment), but no mechanism exists for agents to *discover new
goals* from accumulated experience.

The reflection pipeline already exists: `AgentExperienceRecorder` triggers
`ReflectionOrchestrator.reflect()` when outcome thresholds are met,
producing `List<String>` insights. These insights capture higher-order
patterns from accumulated memories — exactly the kind of signal that
should surface new goal candidates.

No project in the landscape analysis has agents that form new goals from
experience. Goals are either pre-declared or implicit from survival
pressure. This is the most underexplored area in the field.

## Design Principles

1. **Post-reflection pipeline.** Goal formation fires after reflection
   produces insights, on the same virtual thread. The insights are the
   direct input — no separate trigger mechanism.

2. **Rich LLM context.** The LLM sees reflection insights, current goals,
   AND recent memories from `CaseMemoryStore`. Richer context produces
   more relevant goals and prevents duplicates at generation time.

3. **Config-based approval gate.** `auto-approve=true` for dev/testing
   registers goals immediately. `auto-approve=false` logs `GOAL_PROPOSED`
   but does not register — production safety without committing to an
   approval UX.

4. **Guard rails over gatekeeping.** Rate limiting (per-reflection cap +
   cooldown), capacity enforcement (max 10 goals), and structural
   validation (AgentGoal constraints) prevent proliferation without
   blocking legitimate formation.

## Architecture

Inline evaluator pattern — `GoalFormationEvaluator` is called from
`AgentExperienceRecorder` after `reflect()` returns insights. Same
pattern as `GoalRevisionEvaluator` and `PersonalitySignalRecorder`.

```
Worker completes (success or failure)
  → AgentExperienceRecorder.record()                    [existing]
    → evaluateReflectionTrigger()                        [existing]
      → ReflectionOrchestrator.reflect()                 [existing]
        → List<String> insights
      → GoalFormationEvaluator.evaluate()                [NEW]
          1. Guard: enabled check
          2. Guard: GoalSignalStore/AgentRegistry resolvable
          3. Cooldown check (per-agent, configurable)
          4. AgentRegistry.findById() → current descriptor
          5. Capacity check (10 - existing goals count)
          6. Retrieve recent memories via AgentMemoryRetriever
          7. Build GoalFormationContext
          8. GoalFormationStrategy.propose(context) → GoalFormationProposal
          9. Validate each ProposedGoal against AgentGoal constraints
         10. Cap at maxNewGoalsPerReflection
         11. If auto-approve: merge into descriptor, AgentRegistry.register()
         12. If not auto-approve: write GOAL_PROPOSED EventLog only
         13. Write GOAL_FORMED audit EventLog (when registered)
         14. Update cooldown timestamp
```

### Integration point

`AgentExperienceRecorder.evaluateReflectionTrigger()` currently:
1. Checks thresholds
2. Spawns virtual thread calling `reflectionOrchestrator.get().reflect()`

After this change, the virtual thread additionally calls:
```java
List<String> insights = reflectionOrchestrator.get().reflect(...);
goalFormationEvaluator.evaluate(workerName, caseInstance, insights);
```

The evaluator uses `Instance<>` injection with `isResolvable()` — transparent
no-op when dependencies are absent.

## Components

### GoalFormationStrategy SPI (engine-api)

`io.casehub.api.spi.routing`, alongside `GoalRevisionStrategy`.

```java
public interface GoalFormationStrategy extends NamedStrategy {
    Uni<GoalFormationProposal> propose(GoalFormationContext context);

    @Override
    default String id() { return "llm"; }
}
```

### GoalFormationContext (engine-api)

```java
public record GoalFormationContext(
    String agentId,
    String tenancyId,
    List<String> reflectionInsights,
    List<AgentGoal> existingGoals,
    List<RetrievedMemory> recentMemories,
    int remainingCapacity,
    CaseDefinition definition
)
```

`remainingCapacity` = `10 - existingGoals.size()`. Passed explicitly so
the strategy can use it in prompt construction without knowing the max.

`recentMemories` uses the engine-owned `RetrievedMemory` type (already
used by `AgentMemoryRetriever` and `WorkerContext.memories`). Retrieved
via `AgentMemoryRetriever`-style queries: per-domain with the agent's
tenancy, capped at a configurable max.

### GoalFormationProposal (engine-api)

```java
public record GoalFormationProposal(
    List<ProposedGoal> goals,
    String rationale
) {
    public GoalFormationProposal {
        Objects.requireNonNull(goals);
        goals = List.copyOf(goals);
    }

    public record ProposedGoal(
        String name,
        String description,
        GoalPriority suggestedPriority,
        String formationReason
    ) {
        public ProposedGoal {
            Objects.requireNonNull(name);
            Objects.requireNonNull(description);
            Objects.requireNonNull(formationReason);
        }
    }
}
```

`suggestedPriority` is nullable — defaults to `SECONDARY` when null.
New goals should start low-priority and earn promotion via
`GoalRevisionEvaluator`.

No `capabilities` on `ProposedGoal` — new goals start with an empty
capability list. Capability association is a future concern (goal
refinement over time).

### GoalFormationEvaluator (runtime)

`runtime/internal/routing/`, `@ApplicationScoped`.

**Injected dependencies:**
- `Instance<AgentRegistry>` — read current descriptor, write updated one
- `Instance<CaseMemoryStore>` — retrieve recent memories for LLM context
- `CaseDefinitionRegistry` — resolve CaseDefinition, look up AgentDescriptor
- `EngineStrategyResolver` — resolve GoalFormationStrategy by ID
- `EventLogRepository` — write GOAL_FORMED/GOAL_PROPOSED audit entries

**Method:** `evaluate(String workerName, CaseInstance caseInstance, List<String> insights)`

**Internal state:**
```java
private final ConcurrentHashMap<String, Instant> lastFormationTime = new ConcurrentHashMap<>();
```

**Guard rails (in order):**
1. `enabled` config check (master switch, default `false`)
2. `Instance<AgentRegistry>.isResolvable()` — no registry, no formation
3. Resolve `AgentDescriptor` via `CaseDefinition.agentDescriptorFor(workerName)`
4. Cooldown: `now - lastFormationTime[agentKey] < cooldown` → skip
5. Capacity: `10 - descriptor.goals().size() <= 0` → skip (full)
6. Empty insights → skip

**Validation of ProposedGoals:**
- Name: non-null, ≤100 chars, not duplicate of existing goal names
- Description: non-null, ≤500 chars
- Priority: default to `SECONDARY` if null
- Visibility: always `PUBLIC` (new goals are public)
- Capabilities: empty list (v1)
- Per-goal error isolation: invalid proposal logged and skipped, valid
  proposals still registered

**Registration (auto-approve=true):**
```java
List<AgentGoal> merged = new ArrayList<>(descriptor.goals());
for (ProposedGoal proposed : validatedGoals) {
    merged.add(new AgentGoal(
        proposed.name(), proposed.description(),
        proposed.suggestedPriority() != null ? proposed.suggestedPriority() : GoalPriority.SECONDARY,
        Visibility.PUBLIC, List.of()));
}
AgentDescriptor updated = descriptor.toBuilder().goals(merged).build();
agentRegistry.get().register(updated);
```

`AgentDescriptor.Builder.build()` validates max 10 goals, no duplicate
names — final safety net.

### LlmGoalFormationStrategy (runtime)

`runtime/internal/routing/`, `@ApplicationScoped`, id=`"llm"`.

**Injected:** `Instance<ChatModelProvider>` — transparent no-op when absent.

**Prompt construction:**
- System: "You are a goal discovery analyst. Given an agent's recent
  reflection insights, its current goals, and relevant memories, identify
  new goals the agent should pursue. Only propose goals that represent
  genuinely new objectives — not refinements of existing goals. Each goal
  must be specific, actionable, and distinct from existing goals."
- User: all reflection insights, current goals with descriptions and
  priorities, recent memories (text summaries), remaining capacity.
- Response schema (structured output):

```json
{
  "type": "object",
  "properties": {
    "goals": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "name": { "type": "string" },
          "description": { "type": "string" },
          "suggestedPriority": { "type": ["string", "null"], "enum": ["PRIMARY", "SECONDARY", null] },
          "formationReason": { "type": "string" }
        },
        "required": ["name", "description", "formationReason"]
      }
    },
    "rationale": { "type": "string" }
  },
  "required": ["goals", "rationale"]
}
```

**Error handling:**
- ChatModelProvider absent → `Uni.createFrom().failure(new UnsupportedOperationException(...))`
- Invalid JSON → `Uni.createFrom().failure(new AgentException(...))`
- Empty goals array → valid result (no new goals needed)

### Memory Retrieval

`GoalFormationEvaluator` retrieves recent memories using the same pattern
as `AgentMemoryRetriever`: queries `CaseMemoryStore` per domain with the
agent's tenancy. Uses `Instance<CaseMemoryStore>` with `isResolvable()`
guard — when no memory store is available, formation proceeds with empty
memories (insights + goals only).

Configurable max memories: `casehub.engine.goal.formation.max-memories`
(default 20).

## Configuration

All agent-level via `@ConfigProperty`.

| Property | Default | Description |
|----------|---------|-------------|
| `casehub.engine.goal.formation.enabled` | `false` | Master switch |
| `casehub.engine.goal.formation.auto-approve` | `true` | Register immediately or log GOAL_PROPOSED only |
| `casehub.engine.goal.formation.strategy` | `"llm"` | GoalFormationStrategy ID |
| `casehub.engine.goal.formation.max-new-per-reflection` | `2` | Cap per reflection cycle |
| `casehub.engine.goal.formation.cooldown-minutes` | `60` | Min time between formation attempts per agent |
| `casehub.engine.goal.formation.max-memories` | `20` | Max memories to retrieve for LLM context |

## EventLog Audit

Two event types:

### GOAL_FORMED (registered)

Written when `auto-approve=true` and goals are registered.

| Key | Type | Description |
|-----|------|-------------|
| `agentId` | String | Which agent's goals were updated |
| `formedGoals` | List | New goal names |
| `formedGoals[].name` | String | Goal name |
| `formedGoals[].description` | String | Goal description |
| `formedGoals[].priority` | String | Assigned priority |
| `formedGoals[].reason` | String | Formation reason from LLM |
| `previousGoalCount` | int | Goals before formation |
| `newGoalCount` | int | Goals after formation |
| `strategyId` | String | Which strategy generated them |
| `insightCount` | int | Number of reflection insights used |
| `memoryCount` | int | Number of memories retrieved |

### GOAL_PROPOSED (not registered)

Written when `auto-approve=false`. Same metadata as GOAL_FORMED plus
`approved: false`.

## Module Placement

| Type | Module | Rationale |
|------|--------|-----------|
| `GoalFormationStrategy` | engine-api | Consumer-implementable NamedStrategy SPI |
| `GoalFormationContext`, `GoalFormationProposal` | engine-api | SPI parameter/result types |
| `GoalFormationEvaluator` | runtime | Execution infrastructure |
| `LlmGoalFormationStrategy` | runtime | Built-in strategy implementation |
| `GOAL_FORMED`, `GOAL_PROPOSED` event types | engine-api | Event constants |

## Testing

### Unit tests

1. **`GoalFormationEvaluatorTest`** — core logic:
   - Skips when not enabled
   - Skips when AgentRegistry not resolvable
   - Skips when no descriptor or no insights
   - Skips during cooldown period
   - Skips when capacity full (10 existing goals)
   - Calls GoalFormationStrategy.propose() with correct context
   - Validates proposed goals: name length, description length, duplicates
   - Per-goal error isolation: invalid proposal skipped, valid ones registered
   - Caps at maxNewGoalsPerReflection
   - auto-approve=true: registers via AgentRegistry
   - auto-approve=false: writes GOAL_PROPOSED, does NOT register
   - Updates cooldown timestamp after formation
   - Exception isolation: never blocks case progression
   - Writes GOAL_FORMED EventLog with correct metadata

2. **`GoalFormationProposalTest`** — record validation, null handling

3. **`LlmGoalFormationStrategyTest`**:
   - Produces proposal from structured LLM response
   - No-op when ChatModelProvider absent
   - Invalid JSON → Uni failure
   - Empty goals array when no new goals needed
   - Includes existing goals and memories in prompt

### Integration test

4. **`GoalFormationIntegrationTest`** (`@QuarkusTest`):
   - Full flow: case with agent goals → multiple worker completions →
     reflection fires → insights returned → formation evaluates →
     new goal registered on AgentDescriptor
   - Verify new goal appears in AgentRegistry
   - Verify GOAL_FORMED EventLog with correct metadata
   - Verify cooldown prevents immediate re-formation
   - Verify capacity enforcement (nearly full descriptor)

## Scope Boundaries

**In scope:**
- GoalFormationEvaluator with post-reflection trigger
- GoalFormationStrategy SPI + LlmGoalFormationStrategy
- GoalFormationContext, GoalFormationProposal, ProposedGoal
- Config-based approval gate (auto-approve flag)
- Rate limiting (per-reflection cap + cooldown)
- AgentGoal constraint validation
- Memory retrieval for LLM context
- EventLog audit (GOAL_FORMED, GOAL_PROPOSED)
- AgentExperienceRecorder integration (call after reflect())

**v1 constraints (deliberate):**
- New goals always start with empty capabilities list
- New goals always PUBLIC visibility
- No built-in approval workflow (config gate only)
- Single-node cooldown state (ConcurrentHashMap)
- No goal deduplication beyond name matching

**Out of scope (future work):**
- Human approval workflow for GOAL_PROPOSED
- Capability association on newly formed goals
- Cross-agent goal coordination
- Goal deduplication via semantic similarity
- Per-tenancy configuration overrides
- Persistent cooldown state (survives restart)
