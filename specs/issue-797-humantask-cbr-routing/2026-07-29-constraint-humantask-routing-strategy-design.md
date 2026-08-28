# ConstraintHumanTaskRoutingStrategy Design

Refs: engine#755 (parent: engine#797)

## Context

engine#741 shipped the `HumanTaskRoutingStrategy` SPI. engine#754 added the
CBR-based implementation scoring candidates by historical plan trace data.

This spec covers a constraint-based strategy that uses declarative rules for
candidate filtering and scoring, with both context-driven conditions and
workload/fairness balancing.

## Two constraint types

Context constraints and workload constraints have genuinely different evaluation
semantics. Context constraints are global (evaluated once against the case state,
effect applies to named groups/users). Workload constraints are per-candidate
(evaluated for each candidate user against their operational state). Unifying them
into a single evaluation model adds artificial complexity.

### ContextConstraint

Global condition evaluated against the case context (working layer). When the
condition is true, the effect applies to named groups or users.

**Java DSL:**
```java
ContextConstraint.builder()
    .when(ctx -> ctx.layer(WORKING).getDouble("transaction.amount") > 10000)
    .preferUsers(Set.of("senior-reviewer-a", "senior-reviewer-b"))
    .weight(0.8)
    .build()
```

**YAML:**
```yaml
humanTaskConstraints:
  context:
    - when: ".transaction.amount > 10000"
      preferGroups: ["senior-reviewers"]
      weight: 0.8
    - when: ".risk.level == \"high\""
      excludeUsers: ["junior-analysts"]
```

**Record fields:**
- `condition` — `ExpressionEvaluator` (JQ, MVEL, or Lambda)
- `effect` — sealed: `Prefer(Set<String> groups, Set<String> users)` |
  `Exclude(Set<String> groups, Set<String> users)`
- `weight` — `double`, 0.0–1.0 (scoring magnitude for `Prefer`; ignored for `Exclude`).
  Values outside 0.0–1.0 are rejected with `IllegalArgumentException` in the compact
  constructor.

**Effect semantics:**
- `Prefer` — additive score boost. Each matching user gets `+weight` added to their
  score. Multiple constraints stack.
- `Exclude` — hard filter. Matching users are removed from candidates before scoring.
  Irreversible within this evaluation cycle.

**Condition evaluation:** the strategy uses
`ExpressionEngineRegistry.evaluate(constraint.condition(), context.caseContext())`
where `caseContext` is the `CaseContext` from `HumanTaskRoutingContext`. This is
polymorphic — JQ, Lambda, MVEL, and any future expression language are dispatched
automatically by the registry. Non-boolean or evaluation failure → condition
treated as false (logged, not thrown).

**Location:** `api/src/main/java/io/casehub/api/model/routing/ContextConstraint.java`

### WorkloadConstraint

Per-candidate thresholds applied using data from `WorkloadDataProvider`.

**Java DSL:**
```java
WorkloadConstraint.builder()
    .maxActiveTaskCount(5)
    .loadBalanceWeight(0.3)
    .build()
```

**YAML:**
```yaml
humanTaskConstraints:
  workload:
    maxActiveTaskCount: 5
    loadBalanceWeight: 0.3
```

**Record fields:**
- `maxActiveTaskCount` — `Integer`, nullable. Exclude users with active task count
  above this threshold. Hard filter. Negative values rejected with
  `IllegalArgumentException` in the compact constructor.
- `loadBalanceWeight` — `Double`, nullable, 0.0–1.0. Scoring weight for load
  balancing. Lower active count → higher score, normalised across candidates.
  Score formula: `weight * (1.0 - (userCount / maxCountAmongCandidates))`.
  When `maxCountAmongCandidates == 0` (all candidates equally idle), workload scoring
  is skipped — no differentiation is needed. Values outside 0.0–1.0 are rejected
  with `IllegalArgumentException` in the compact constructor (consistent with
  `ContextConstraint.weight`).

**Location:** `api/src/main/java/io/casehub/api/model/routing/WorkloadConstraint.java`

## WorkloadDataProvider SPI

Provides operational workload snapshots for candidate users.

```java
public interface WorkloadDataProvider extends NamedStrategy {
    Map<String, WorkloadSnapshot> getWorkload(Set<String> userIds, String tenancyId);
}

public record WorkloadSnapshot(int activeTaskCount) {}
```

**Location:** `api/src/main/java/io/casehub/api/spi/routing/WorkloadDataProvider.java`
and `api/src/main/java/io/casehub/api/spi/routing/WorkloadSnapshot.java`

**Default:** `NoOpWorkloadDataProvider` (`@DefaultBean @ApplicationScoped @Unremovable`,
id `"default"`) in `runtime/internal/routing/`. Returns empty map. When active,
workload constraints degrade gracefully — `maxActiveTaskCount` excludes nobody,
`loadBalanceWeight` produces no scores.

**Real implementations** (follow-on, not #755 scope):
- `casehub-engine-actor-state` — backed by `WorkerExecutionManager.getActiveCaseIds()`
- `casehub-work-engine-adapter` — backed by WorkItem query

**EngineStrategyResolver:** already discovers `NamedStrategy` beans via the
`registerRemainingStrategies` catch-all. No resolver changes needed.

## ConstraintHumanTaskRoutingStrategy

**Location:** `runtime/src/main/java/io/casehub/engine/internal/routing/ConstraintHumanTaskRoutingStrategy.java`

**CDI:** `@ApplicationScoped @Unremovable`, id `"constraint"`.

**Dependencies:** `ExpressionEngineRegistry` for polymorphic condition evaluation,
`@Inject WorkloadDataProvider` (CDI resolves `@DefaultBean NoOpWorkloadDataProvider`
when no real implementation is present — the provider is always injected, never absent).

### Algorithm

1. Read constraints from `context.caseDefinition()`:
   `humanTaskContextConstraints` and `humanTaskWorkloadConstraint`.
2. Copy `candidates.users()` to a mutable `eligibleUsers` set.
3. **Context constraints (global):** for each `ContextConstraint` on the
   case definition:
   a. Evaluate `condition` using
      `expressionEngineRegistry.evaluate(condition, context.caseContext())`.
   b. If false, skip.
   c. If `Exclude` effect: remove named users from `eligibleUsers`. Group-based
      exclusion deferred to engine#757 (no-op until group membership resolution).
   d. If `Prefer` effect: accumulate `+weight` for each named user present in
      `eligibleUsers`. Group-based preference deferred to engine#757.
4. If `eligibleUsers` is empty after exclusions → return
   `Escalated("all candidates excluded by context constraints")`.
5. **Workload constraints (per-candidate):** if `WorkloadConstraint` is configured
   AND `WorkloadDataProvider` returns non-empty data:
   a. Query `provider.getWorkload(eligibleUsers, tenancyId)`.
   b. If `maxActiveTaskCount` set: exclude users above threshold from `eligibleUsers`.
   c. If `eligibleUsers` empty after workload exclusion → return
      `Escalated("all candidates excluded by workload constraints")`.
   d. If `loadBalanceWeight` set AND `maxCountAmongCandidates > 0`: compute normalised
      load-balance scores for remaining candidates.
      Formula: `weight * (1.0 - (count / maxCount))`.
      Users without workload data get score 0.0 (no boost, not excluded).
      When `maxCountAmongCandidates == 0`, skip — all candidates equally idle.
6. **Combine scores:** context preference scores + workload balance scores
   (additive). Filter combined scores to only include keys present in
   `eligibleUsers` — a Prefer may have scored a user who was subsequently
   removed by an Exclude or workload threshold.
7. If no changes were made (no exclusions from `candidates.users()` AND combined
   scores map is empty) → return `Unchanged`.
8. Return `Enriched(candidates.groups(), eligibleUsers, combinedScores)`.

Step 7 ensures that exclusion-only scenarios (no Prefer, no workload scoring)
return `Enriched` with the reduced `eligibleUsers` set rather than `Unchanged`,
which would cause the handler to restore the original candidates.

**Key differences from CbrHumanTaskRoutingStrategy:**
- CAN filter candidates (Exclude, maxActiveTaskCount)
- CAN escalate (all candidates excluded)
- Modifies `candidateUsers` in the Enriched result (excluded users removed)
- Does not use `ExperienceAnalyser` — independent of CBR (single-strategy selection
  model means one or the other, not both; see §Future work for composition)

### Expression evaluation

The strategy injects `ExpressionEngineRegistry` (api module) for all condition
evaluation. This is polymorphic — JQ, Lambda, MVEL, and any future expression
language are dispatched automatically:

```java
boolean match = expressionEngineRegistry.evaluate(constraint.condition(), context.caseContext());
```

The `CaseContext` overload handles JQ (extracts working layer as `JsonNode`
internally), Lambda (passes `CaseContext` directly), and MVEL. No `instanceof`
dispatch in the strategy. Consistent with `ExpressionSetStrategy` and
`CaseContextChangedEventHandler` which both use `ExpressionEngineRegistry`.

### HumanTaskRoutingContext change

```java
public record HumanTaskRoutingContext(
    UUID caseId,
    String bindingName,
    String tenancyId,
    CaseContext caseContext,
    CaseDefinition caseDefinition,
    List<RetrievedExperience> experiences) {}
```

Changes from current:
- `JsonNode caseContext` → `CaseContext caseContext` — strategies that need the JSON
  form call `caseContext.layer(ContextLayer.WORKING).asJsonNode()`. The
  `ExpressionEngineRegistry.evaluate(evaluator, CaseContext)` overload requires
  `CaseContext`, not `JsonNode`.
- Added `CaseDefinition caseDefinition` — gives any strategy access to definition-level
  configuration (constraints, routing config, etc.) without strategy-specific context
  fields. The handler already has `CaseDefinition` — threading it costs nothing.

This is a breaking change to the record (parameter change + addition). All construction
sites update. Pre-release platform — the cost is trivial.

## Handler changes

### Escalated result handling

The current `CaseContextChangedEventHandler.publishHumanTaskSchedule()` switch on
`HumanTaskRoutingResult.Escalated` logs a warning but continues to publish
`HumanTaskScheduleEvent` with the original candidates — the escalation has no
effect. This is a bug: the task dispatches to the full candidate pool as if no
constraints existed.

**Fix:** The `Escalated` branch returns early without publishing the event,
leaving the PlanItem in PENDING state. This is consistent with how the handler
returns early on bridge validation failure (line ~553). A warning log is retained
for observability.

```java
case HumanTaskRoutingResult.Escalated e -> {
    LOG.warnf(
        "HumanTask routing escalated for caseId=%s binding=%s: %s — PlanItem stays PENDING",
        caseInstance.getUuid(), binding.getName(), e.reason());
    return;
}
```

This handler change is a deliverable of engine#755, not a follow-on.

## CaseDefinition changes

`CaseDefinition` gains:
- `List<ContextConstraint> humanTaskContextConstraints` (empty by default)
- `WorkloadConstraint humanTaskWorkloadConstraint` (nullable)

**Scope:** constraints are case-level, consistent with `humanTaskRouting` (also
case-level). All human task bindings in a case share the same constraints.
Binding-level constraints (per-binding overrides) are out of scope for #755 and
can be added in a follow-on if needed — the constraint model and algorithm are
compatible with binding-level scoping.

**Builder:**
```java
.humanTaskContextConstraint(ContextConstraint.builder()
    .when(".transaction.amount > 10000")
    .preferGroups(Set.of("senior-reviewers"))
    .weight(0.8)
    .build())
.humanTaskWorkloadConstraint(WorkloadConstraint.builder()
    .maxActiveTaskCount(5)
    .loadBalanceWeight(0.3)
    .build())
```

**YAML mapper:** `humanTaskConstraints:` block in `CaseDefinitionYamlMapper` with
`context:` array and `workload:` object sub-blocks. The `when:` field in each
context constraint is converted to an `ExpressionEvaluator` at parse time using
`ExpressionEngineRegistry.create(expression, "jq")`, following the
`ExpressionSetStrategy` precedent. JQ is the only YAML expression language for
#755 scope — Lambda is available via the Java DSL only.

**Registration-time validation:** When `CaseDefinition` is registered, if any
context constraint has group-based effects (`preferGroups` / `excludeGroups`),
a warning is logged: "Group-based constraint effects configured but not evaluable
until engine#757 — group membership resolution required." The constraint is stored
(data model complete); evaluation skips group effects.

## Activation

```yaml
humanTaskRouting: "constraint"
humanTaskConstraints:
  context:
    - when: ".transaction.amount > 10000"
      preferGroups: ["senior-reviewers"]
      weight: 0.8
  workload:
    maxActiveTaskCount: 5
    loadBalanceWeight: 0.3
```

`EngineStrategyResolver` resolves by id `"constraint"`.

## Group membership limitation

`Exclude(groups)` and `Prefer(groups)` need to know which candidate users belong
to which groups. The strategy receives `HumanTaskCandidates(groups, users)` — flat
sets, no membership mapping.

For #755: group-based effects are stored on the constraint but have no effect at
evaluation time. A warning is logged at CaseDefinition registration time when
group-based effects are configured. `excludeUsers`/`preferUsers` work immediately.
Group evaluation is deferred to engine#757 (group scoring via group membership
resolution). The constraint model is complete; the group evaluation is deferred.

## Tests

### Unit tests — ContextConstraint

`api/src/test/java/io/casehub/api/model/routing/ContextConstraintTest.java`

| Test | Assertion |
|------|-----------|
| `preferUsersEffect` | Builder creates Prefer with users |
| `preferGroupsEffect` | Builder creates Prefer with groups |
| `excludeUsersEffect` | Builder creates Exclude with users |
| `excludeGroupsEffect` | Builder creates Exclude with groups |
| `weightOutOfRangeRejected` | Weight outside 0.0–1.0 → IllegalArgumentException |
| `conditionRequired` | Null condition rejected |
| `effectRequired` | No effect set rejected |

### Unit tests — WorkloadConstraint

`api/src/test/java/io/casehub/api/model/routing/WorkloadConstraintTest.java`

| Test | Assertion |
|------|-----------|
| `maxActiveTaskCountOnly` | Builds with threshold, null weight |
| `loadBalanceWeightOnly` | Builds with weight, null threshold |
| `bothSet` | Both fields populated |
| `loadBalanceWeightOutOfRangeRejected` | Weight outside 0.0–1.0 → IllegalArgumentException |
| `maxActiveTaskCountNegativeRejected` | Negative threshold → IllegalArgumentException |
| `emptyConstraintRejected` | Neither field set → rejected |

### Unit tests — ConstraintHumanTaskRoutingStrategy

`runtime/src/test/java/io/casehub/engine/internal/routing/ConstraintHumanTaskRoutingStrategyTest.java`

| Test | Assertion |
|------|-----------|
| `idIsConstraint` | `id()` returns `"constraint"` |
| `noConstraintsReturnsUnchanged` | No constraints configured → Unchanged |
| `preferUsersBoostsScores` | Matching Prefer adds weight to named users |
| `excludeUsersRemovesCandidates` | Matching Exclude removes named users |
| `excludeOnlyReturnsEnrichedNotUnchanged` | Exclude with no Prefer/workload → Enriched with reduced users, empty scores |
| `falseConditionSkipped` | Condition evaluates false → no effect |
| `multipleConstraintsStack` | Multiple Prefer weights are additive |
| `allExcludedEscalates` | All users excluded → Escalated |
| `workloadExcludesAboveThreshold` | Users above maxActiveTaskCount removed |
| `workloadLoadBalanceScoring` | Lower load → higher score |
| `workloadAllIdleSkipsScoring` | All candidates have 0 tasks → no workload scores (avoids 0/0) |
| `workloadAllExcludedEscalates` | All users above threshold → Escalated |
| `noWorkloadProviderSkipsWorkload` | No provider → workload constraints have no effect |
| `preferThenExcludeCleansScores` | Prefer scores user, subsequent Exclude removes user — scores filtered to eligibleUsers only |
| `combinedContextAndWorkload` | Both constraint types combine scores additively |
| `groupEffectsDeferredNoOp` | preferGroups/excludeGroups stored but no effect (pending #757) |
| `lambdaConditionEvaluated` | Lambda ExpressionEvaluator works via ExpressionEngineRegistry |

### Handler tests — Escalated result

`runtime/src/test/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandlerRoutingTest.java` — extend:

| Test | Assertion |
|------|-----------|
| `humanTaskEscalatedReturnsEarly` | Escalated result → no HumanTaskScheduleEvent published, PlanItem stays PENDING |

### YAML mapper tests

`api/src/test/java/io/casehub/api/model/converter/CaseDefinitionYamlMapperTest.java` — extend:

| Test | Assertion |
|------|-----------|
| `humanTaskConstraints_contextParsed` | YAML context constraints parsed with baked-in ExpressionEvaluator |
| `humanTaskConstraints_workloadParsed` | YAML workload constraint parsed |

## Downstream impact

- **HumanTaskRoutingContext** changes: `JsonNode caseContext` → `CaseContext caseContext`,
  added `CaseDefinition caseDefinition`. All construction sites in
  `CaseContextChangedEventHandler.publishHumanTaskSchedule()` update. Pre-release
  breaking change.
- **CaseContextChangedEventHandler** handler fix: `Escalated` branch returns early
  without publishing event (deliverable of #755).
- **CaseDefinition** gains two fields — additive, no existing code breaks.
- **CaseDefinitionYamlMapper** gains `humanTaskConstraints:` parsing with
  `ExpressionEngineRegistry.create()` for baked-in evaluators.
- **EngineStrategyResolver** — no changes (auto-discovers `NamedStrategy` beans).
- **WorkloadDataProvider** SPI + no-op default — additive.

## Future work

- engine#757: Group membership resolution — enables group-based Prefer/Exclude effects
- Follow-on: Composite strategy combining CBR scoring with constraint filtering
  (implementable as a standalone strategy bean that delegates to both — no SPI changes)
- Follow-on: Binding-level constraint overrides (per-binding scoping beyond case-level)
- Follow-on: Real `WorkloadDataProvider` implementation (actor-state or work adapter)
- Follow-on: MVEL expression evaluator support in the constraint evaluation path
