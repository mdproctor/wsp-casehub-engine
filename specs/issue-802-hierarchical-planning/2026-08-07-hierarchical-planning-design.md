# Hierarchical Planning — Goal Decomposition Design

**Issue:** engine#802
**Epic:** engine#800 (Sub-epic B — Agent Reflection & Planning)
**Depends on:** engine#801 (reflection orchestration — closed), eidos-api `AgentGoal` record, blocks#60 Phase 5 (HTN decomposition SPI — landed)

## Problem

The `DecompositionStrategy` SPI exists in engine-api with a full type hierarchy
(`TaskNode`, `CompoundTask`, `LeafTask`, `DecompositionMethod`, `DecompositionContext`),
`EngineStrategyResolver` wires it, and `CaseDefinition.decompositionStrategy` selects
it by ID — but nothing in the engine calls it. No production implementation exists.

Agents have goals (`AgentGoal` on `AgentDescriptor`) and the engine tracks goal
lifecycle (abandonment, failure recording, completion marking), but there is no
mechanism to decompose a high-level goal into an ordered sequence of capability
invocations. Agents either execute as single-shot workers or rely on manual
binding ordering via trigger conditions.

## Design

### Architecture

Pre-dispatch goal decomposition at case start. An LLM-backed
`DecompositionStrategy` takes agent goals and available capabilities, produces
an ordered plan, and materializes it as compound PlanItems. The existing
planning/dispatch loop then executes sub-steps in order.

```
Case starts
  → GoalDecomposer collects active AgentGoals per agent
  → Creates CompoundTask per goal
  → Calls DecompositionStrategy.decompose(compoundTask, context)
  → LlmDecompositionStrategy prompts ChatModel with goal + context + capabilities
  → Returns DagPlan<LeafTask<JsonNode>> (ordered sub-steps, each referencing a capability)
  → GoalDecomposer materializes DagPlan as PlanItemDefinitions:
      - Compound for the goal (parent)
      - Primitive per sub-step (children, ordered)
      - Each child targets a capability binding on the case definition
  → CasePlanModel.addCompound() registers the compound
  → PlanItemStore persists PlanItems
  → Existing dispatch loop (CompoundStrategyDispatcher + CHOREOGRAPHED) handles ordering
```

### What the LLM decides vs what's static

The LLM's job is narrow: given a goal description and available capabilities,
determine the **ordering and dependencies** between capabilities. It does NOT:

- Invent new capabilities (must reference existing ones on the case definition)
- Select workers (routing strategies handle that)
- Set input projections (bindings handle that)
- Define trigger conditions (sub-steps dispatch in compound order)

Validation: `GoalDecomposer` rejects any sub-step referencing a capability not
declared on the case definition. Missing capability → logged warning, step skipped.

## Components

### GoalStep (engine-api, `io.casehub.engine.plan`)

Record implementing `TaskNode.LeafTask<JsonNode>` — represents a single
decomposed sub-step with a capability reference.

```java
public record GoalStep(
    UUID id,
    String description,
    String capabilityName,
    Instant createdAt
) implements TaskNode.LeafTask<JsonNode> {
    // TaskDescriptor: executor() returns null (assigned at dispatch)
    // status() returns PENDING
    // snapshot() returns TaskSnapshot with description + capabilityName
}
```

Per protocol PP-20260727-5267d2: plan-definition type (consumers inspect it) →
belongs in engine-api.

### GoalDecomposer (runtime/internal/planning/, @ApplicationScoped)

Orchestrates goal-to-plan decomposition at case start.

**Injected dependencies:**
- `Instance<DecompositionStrategy<JsonNode>>` — resolved via `EngineStrategyResolver` from `CaseDefinition.decompositionStrategy`
- `GoalAbandonmentEvaluator` — filters abandoned goals
- `CaseDefinitionRegistry` — looks up agent descriptors
- `EventLogRepository` — records `GOAL_DECOMPOSED` audit entries

**Method:** `decompose(CaseInstance, CaseDefinition, MutableCaseContext, CasePlanModel)`

**Logic:**
1. If `definition.getDecompositionStrategy()` is null → return (no decomposition configured)
2. Resolve `DecompositionStrategy` via `EngineStrategyResolver`
3. For each worker on the definition with an `AgentDescriptor`:
   a. Get active goals via `GoalAbandonmentEvaluator.activeGoals(descriptor)`
   b. For each active goal:
      - Build `CompoundTask(goal.name(), singletonMethod(strategy))`
      - Build `DecompositionContext` with case context snapshot (`JsonNode`) and depth 0
      - Call `strategy.decompose(compoundTask, context).await().indefinitely()`
      - Validate: filter steps with unknown capability names, log warnings
      - If valid steps remain:
        - Build `PlanItemDefinition.Compound` (name = goal name, dispatch = CHOREOGRAPHED, completion = All)
        - Build `PlanItemDefinition.Primitive` per sub-step, scoped to the compound
        - Call `casePlanModel.addCompound(compound)`
        - Create and save PlanItems via PlanItemStore
        - Write EventLog entry (`GOAL_DECOMPOSED`) with plan structure metadata
4. Error isolation: exceptions per goal are caught and logged — one failing
   goal does not prevent decomposition of others or case start

### LlmDecompositionStrategy (runtime/internal/planning/, @ApplicationScoped, id = "llm")

LLM-backed `DecompositionStrategy<JsonNode>` implementation.

**Injected dependencies:**
- `Instance<ChatModelProvider>` — transparent no-op when absent

**Prompt construction:**
- System: "You are a planning assistant. Given a goal and available capabilities, produce an ordered plan of sub-steps. Each step must reference exactly one capability by name."
- User: goal description, goal priority, list of available capability names with descriptions, case context snapshot (truncated to fit context window)
- Response schema (enforced via `Agent.responseSchema()`):

```json
{
  "type": "object",
  "properties": {
    "steps": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "id": { "type": "string" },
          "description": { "type": "string" },
          "capabilityName": { "type": "string" },
          "dependsOn": { "type": "array", "items": { "type": "string" } }
        },
        "required": ["id", "description", "capabilityName"]
      }
    }
  },
  "required": ["steps"]
}
```

**Plan construction:**
- Parse structured response
- Create `GoalStep` per step (UUID generated, `createdAt` = now)
- Build `DagNode<GoalStep>` per step with `dependsOn` edges
- Construct `DagPlan.fromNodes(nodes)` — validates no cycles, no dangling refs
- Return `Uni.createFrom().item(plan)`

**Error handling:**
- `ChatModelProvider` absent → `Uni.createFrom().failure(new UnsupportedOperationException(...))`
- LLM returns invalid structure → `Uni.createFrom().failure(new AgentException(...))`
- LLM returns empty steps → return empty `DagPlan` (GoalDecomposer handles this)

### CasePlanModel extension (planning module)

`DefaultCasePlanModel` gains:

```java
void addCompound(String name, PlanItemDefinition.Compound compound)
```

Adds a compound definition with its children to the model at runtime. The
compound's `scopedBindings` map links child binding names to
`Participation.PARTICIPANT`. Duplicate compound name → `IllegalArgumentException`.

`CompoundLifecycleEvaluator` evaluates entry conditions on the next planning
cycle. `CompoundCompletionEvaluator` propagates completion up the parent chain.
Both already handle compounds generically — no changes needed to these evaluators.

## Integration Points

### Case start handler chain

`CaseStartedEventHandler` gains a call to `goalDecomposer.decompose()` after
initialization but before publishing `CONTEXT_CHANGED`:

```
CaseStartedEventHandler.onCaseStarted()
  → openChannel()
  → fire CaseLifecycleEvent
  → goalDecomposer.decompose(instance, definition, context, casePlanModel)  ← NEW
  → publish CONTEXT_CHANGED (existing)
```

Decomposition is synchronous (runs on virtual thread via `@RunOnVirtualThread`).
The LLM call blocks but this is acceptable — case start is already a heavy
operation. Decomposition happens once per case, not per binding fire.

### EventLog audit

New event type constant in `EventBusAddresses` (or `CaseHubEventType`):

```java
public static final String GOAL_DECOMPOSED = "casehub.goal.decomposed";
```

EventLog metadata for `GOAL_DECOMPOSED`:
- `goalName` — which goal was decomposed
- `agentId` — which agent's goal
- `strategyId` — which DecompositionStrategy was used (e.g., "llm")
- `planStructure` — the full DagPlan as JSON (nodes + edges)
- `stepCount` — number of sub-steps produced
- `skippedSteps` — any steps filtered due to unknown capabilities

### Recovery

On JVM restart, `WorkerRecoveryCoordinator` already recovers PlanItems from
`PlanItemStore`. Since decomposed sub-steps ARE PlanItems (with compound
parent references), recovery is automatic.

For `CasePlanModel` reconstruction: `DefaultCasePlanModel` must be rebuilt
from persisted PlanItems on recovery. The compound structure (parent/child
relationships) is recoverable from PlanItem metadata. The DagPlan structure
in EventLog metadata allows ordering reconstruction without re-calling the LLM.

## YAML Configuration

Existing `decompositionStrategy:` field on `CaseDefinition` selects the strategy.
No new YAML fields needed.

```yaml
spec:
  decompositionStrategy: llm

  capabilities:
    - name: data-gathering
      inputProjection: ".sources"
    - name: analysis
      inputProjection: ".rawData"
    - name: reporting
      inputProjection: ".analysis"

  workers:
    - name: research-agent
      capabilities: [data-gathering, analysis, reporting]
      agent:
        systemPrompt: "You are a research analyst..."
        model: { provider: anthropic, name: claude-sonnet-5 }
      agentDescriptor:
        agentId: research-agent-01
        goals:
          - name: comprehensive-analysis
            description: >
              Gather data from all sources, analyse patterns,
              and produce a structured report
            priority: PRIMARY
            visibility: PUBLIC
            capabilities: [data-gathering, analysis, reporting]

  bindings:
    - name: gather
      capability: data-gathering
      on: { contextChange: ".sources != null" }
    - name: analyse
      capability: analysis
      on: { contextChange: ".rawData != null" }
    - name: report
      capability: reporting
      on: { contextChange: ".analysis != null" }
```

When this case starts with `decompositionStrategy: llm`:
1. `GoalDecomposer` finds `research-agent` has goal `comprehensive-analysis`
2. LLM produces: `gather → analyse → report` (sequential)
3. Creates `Compound("comprehensive-analysis", CHOREOGRAPHED, All)` scoping bindings `gather`, `analyse`, `report`
4. Compound dispatch ensures `gather` fires first, then `analyse` after gather completes, then `report`

### Binding trigger conditions with decomposition

When decomposition is active, the compound's scoped bindings gate dispatch —
bindings only fire when their owning compound activates them. The binding's own
trigger condition (`on: contextChange:`) still applies as an additional gate.
Both must be satisfied: compound says "it's your turn" AND trigger condition is met.

## Module Placement

| Type | Module | Rationale (PP-20260727-5267d2) |
|------|--------|-------------------------------|
| `GoalStep` | engine-api | Plan-definition type — consumers inspect decomposed plans |
| `GoalDecomposer` | runtime | Execution infrastructure — internal orchestration |
| `LlmDecompositionStrategy` | runtime | Execution infrastructure — LLM integration |
| `CasePlanModel.addCompound()` | planning | Plan model mutation — internal to planning module |
| `GOAL_DECOMPOSED` event type | common | Event constant — shared across modules |

## Testing

### Unit tests

1. **`GoalDecomposerTest`** — core logic:
   - Decomposes goals into compound PlanItemDefinitions
   - Skips when no decompositionStrategy configured
   - Skips agents without AgentDescriptor or goals
   - Filters abandoned goals
   - Rejects sub-steps with unknown capability names
   - Rejects non-linear plans (parallel branches) with warning
   - Overlapping compound scopes: higher-priority goal wins contested bindings
   - Idempotency: skips decomposition when PlanItems already exist
   - Timeout: per-goal timeout triggers graceful degradation
   - Error isolation: LLM failure doesn't prevent case start
   - Empty plan → no compound created (check before DagPlan construction)

2. **`LlmDecompositionStrategyTest`** — LLM interaction:
   - Produces DagPlan from structured response
   - Sequential ordering (A → B → C)
   - Parallel-capable ordering (A → [B, C] → D)
   - Invalid JSON → Uni failure
   - Non-existent capabilities filtered out
   - No-op when ChatModelProvider absent

3. **`GoalStepTest`** — record validation:
   - TaskDescriptor contract (id, description, status = PENDING)
   - Null capability name rejected

4. **`GoalDecompositionRecoveryTest`**:
   - CasePlanModel compounds rebuilt from PlanItem metadata after restart
   - Idempotency: second decompose() call is no-op when PlanItems exist
   - Compound ordering preserved after reconstruction

### Integration test

5. **`GoalDecompositionIntegrationTest`** (`@QuarkusTest`):
   - Full flow: case with goals + LLM strategy → start → PlanItems in order → compound completion
   - Mock `ChatModelProvider` with canned JSON
   - EventLog contains `GOAL_DECOMPOSED` with plan structure

## Scope Boundaries

**In scope:**
- GoalDecomposer orchestration at case start
- LlmDecompositionStrategy (single LLM-backed implementation)
- GoalStep record type
- CasePlanModel runtime modification
- EventLog audit trail
- Recovery via existing PlanItem infrastructure

**v1 limitations (deliberate, not gaps):**
- Plans must be linear chains (sequential only). Parallel branches in
  decomposed plans are rejected at validation time. Parallel decomposition
  requires dispatch infrastructure beyond CHOREOGRAPHED — tracked as
  future work.
- Single decomposition strategy per case definition. Per-agent or per-goal
  strategy override is not supported in v1.

**Out of scope (future work):**
- Parallel branch dispatch in decomposed plans (requires DagDriver integration or new dispatch mode)
- Re-decomposition on context change (#803 — plan adaptation)
- Static/YAML-declared decomposition strategy (add when needed)
- GOAP (Goal-Oriented Action Planning) strategy
- Multi-agent coordinated planning
- Plan visualization or inspection API
- Per-agent or per-goal decomposition strategy override
