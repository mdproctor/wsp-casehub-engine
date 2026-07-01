# Milestone, Goal, and Stage — Full Conceptual Alignment

**Issue:** [#581](https://github.com/casehubio/engine/issues/581)  
**Epic:** [#84](https://github.com/casehubio/engine/issues/84)  
**Date:** 2026-06-28

## S1: Summary

Four remaining gaps prevent closing epic #84:

1. `Goal.terminal` exists in the model and YAML schema but creates conceptual overlap with Milestone — the epic's conclusion is to remove it
2. `Milestone` has no `parentStageId` — Stage tracks contained milestones but Milestone has no back-pointer
3. `CasePlanModel` milestone lifecycle tracking uses `Boolean` instead of `MilestoneLifecycleStatus` — explicitly marked as interim in the Javadoc
4. No CMMN 1.1 alignment documentation

## S2: 1. Goal.terminal — remove the field

### S2.1: Problem

`Goal.terminal` defaults to `false` in both the Java model (`Goal.java:64`) and the YAML schema (`CaseDefinition.yaml:429-432`). The YAML mapper (`CaseDefinitionYamlMapper:354-363`) does not read `sg.getTerminal()` — the value is silently dropped. `GoalReachedEventHandler` writes `isTerminal` to EventLog metadata but it has no behavioral effect.

Epic #84 identified this as a design problem:

> "A non-terminal Goal is 'an observable checkpoint with polarity.' That is just a Milestone with a success/failure label — the two concepts now overlap."

> "Non-terminal Goals are removed — they were Milestones."

Issue #581 echoes: "Either remove `Goal.terminal` (Goals are always terminal) or add validation that rejects `terminal: false`."

### S2.2: Design decision

Goals are always terminal. A Goal exists to drive case completion — that is its defining characteristic. A non-terminal goal is a Milestone with polarity, which is conceptual overlap the epic was created to eliminate.

`GoalExpression` membership is the sole mechanism for determining which goals gate case completion. The `terminal` field is redundant at best and contradictory at worst (a goal marked `terminal: false` but referenced in a `GoalExpression` would create a semantic conflict with no clear resolution). The field is removed entirely.

### S2.3: Changes

**Goal.java** — remove `terminal` field, `getTerminal()`, `setTerminal()`. Remove `terminal` from `equals()` and `hashCode()`. Remove `terminal(boolean)` from `Builder` and the `setTerminal()` call from `Builder.build()`.

**Goal.java Javadoc** — remove the paragraph (lines 53-56) about non-terminal goals ("Non-terminal goals — goals not referenced in completion logic — behave as observable checkpoints with polarity. Use Milestone instead..."). The class doc states clearly: Goals are always terminal outcomes that drive case completion via `GoalBasedCompletion`.

**CaseDefinition.yaml** — remove the `terminal` property from the Goal schema definition (lines 429-432).

**GoalReachedEventHandler** — remove `.put("isTerminal", goal.getTerminal())` from EventLog metadata construction (line 90). No other changes — `evaluateCompletion()` is correct as-is.

**CaseDefinitionYamlMapper** — no change needed (it already does not read `terminal`).

**DefaultCaseDefinitionRegistry** — add a registration-time WARNING log for Goals defined in the `CaseDefinition` that are not referenced in any `GoalExpression` (success or failure). The warning fires during `register()`. Message: "Goal 'X' is not referenced in any GoalExpression. Goals should drive case completion — use Milestone for non-terminal checkpoints." This structurally reinforces "goals are always terminal" without hard rejection (which would break existing tests that define goals without completion blocks).

### S2.4: Deferred concern

`GoalBasedCompletion.java:18-19` contains a TODO:

```java
// TODO this must be replaced by a more generic implementation that can support
// multiple goals of different kinds, not just success and failure
```

This is completion mechanism redesign, not conceptual alignment. It is out of scope for epic #84 but must be tracked as a follow-on issue: "Generalize GoalBasedCompletion to support multiple goal kinds beyond success/failure" (refs `GoalBasedCompletion.java:18-19`).

### S2.5: Tests

- Verify `Goal.builder()` no longer exposes `terminal(boolean)` — compilation verification
- Verify `GoalReachedEventHandler` EventLog metadata does not contain `isTerminal` key
- Verify YAML with `terminal:` property is silently ignored (not mapped to API model) — the YAML mapper uses a lenient ObjectMapper with `FAIL_ON_UNKNOWN_PROPERTIES` disabled (`CaseDefinitionYamlMapper.java:157-163`), so unknown properties are not rejected
- All goals participate equally in `evaluateCompletion()` — reaching a goal always invokes the completion evaluation path, regardless of how the goal was defined
- Verify `DefaultCaseDefinitionRegistry.register()` logs a WARNING when a Goal is not referenced in any GoalExpression

## S3: 2. Milestone parentStageId — programmatic containment

### S3.1: Problem

`Stage.containedMilestoneIds` exists (forward pointer) but `Milestone` has no `parentStageId` (back-pointer). `Stage.addMilestone()` is defined but never called from anywhere.

### S3.2: Context

Stage is a blackboard-internal Java construct — it is not in the YAML schema and not in the API model. Stages are built programmatically in Java CaseHub subclasses, not declared in YAML. Milestone-stage containment should follow the same pattern.

YAML nesting of milestones inside stage blocks becomes relevant when/if stages are added to the YAML schema — a follow-on concern, not this issue.

### S3.3: Changes

**Milestone.java** — add `parentStageId` (String) field, getter, and builder method. Immutable after construction. Null for case-level milestones not owned by any stage. Add `parentStageId` to `equals()` and `hashCode()` — it is structural identity, consistent with every other definition-time field on the class.

**DefaultCasePlanModel** — add overloaded `trackMilestone(String name, String parentStageId)` method. This method:
1. If `parentStageId` is null, the milestone is tracked without stage association, equivalent to the single-argument `trackMilestone(name)` overload
2. If `parentStageId` is non-null, looks up the stage by `parentStageId` via `getStage(parentStageId)`
3. Calls `stage.addMilestone(milestoneName)` to wire the forward pointer
4. Throws `IllegalArgumentException` if `parentStageId` is non-null and the named stage does not exist in the plan
5. Registers the milestone in the tracking map as PENDING (see §S4)

**CasePlanModel interface** — add `trackMilestone(String name, String parentStageId)` to the interface.

**Ordering constraint:** Milestones with a `parentStageId` must be registered after their parent stage is added to the plan. This is enforced by the stage lookup — if the stage hasn't been added yet, the `IllegalArgumentException` fires. This matches the existing pattern where required items are registered before `startCase()`.

**No YAML schema changes** — stages are not in YAML; milestone containment is programmatic.

**Not in scope:** Stage exit criteria referencing milestone achievement — `StageAutocompleteEvaluator` already has `containedMilestoneIds` for future use but wiring that evaluation is a follow-on concern.

### S3.4: Tests

- Milestone built with `.parentStageId("kyc-stage")` — verify getter returns the stage name
- Milestone built without `parentStageId` — verify getter returns null
- Milestone equality: two milestones identical except for `parentStageId` — verify they are NOT equal
- `trackMilestone("doc-check", "kyc-stage")` after `addStage(kycStage)` — verify bidirectional: milestone status is PENDING, `stage.getContainedMilestoneIds()` includes "doc-check"
- `trackMilestone("doc-check", "nonexistent")` — verify `IllegalArgumentException`
- `trackMilestone("doc-check", null)` — verify milestone tracked without stage association (backwards-compatible with existing single-arg method)

## S4: 3. Milestone lifecycle state on CasePlanModel

### S4.1: Problem

`CasePlanModel` tracks milestone achievement with `ConcurrentHashMap<String, Boolean>` — a simple boolean, not the `MilestoneLifecycleStatus` enum that already exists in `api/`. The `CasePlanModel` Javadoc (line 33) explicitly says: *"Milestone lifecycle tracking here is an interim approach. Full alignment... tracked in casehubio/engine#84."*

The `MilestoneLifecycleStatus` enum (PENDING, ACTIVE, COMPLETED, FAILED, CANCELLED) already exists. `MilestoneLifecycleManager` already publishes `MilestoneActivatedEvent` and `MilestoneCompletedEvent` with proper lifecycle transitions. But `MilestoneAchievementHandler` in the blackboard module only listens for `MILESTONE_REACHED` and sets a boolean — it does not track the PENDING → ACTIVE → COMPLETED lifecycle.

Additionally, two parallel milestone evaluation paths exist:
1. `CaseContextChangedEventHandler.milestones()` → fires `MILESTONE_REACHED` on every context change where `completionCriteria` is true (not idempotent, no entry criteria check)
2. `MilestoneLifecycleManager` → evaluates full lifecycle (PENDING→ACTIVE→COMPLETED), fires once per transition (idempotent), enforces entry criteria

Path #2 supersedes path #1 entirely. `MilestoneLifecycleManager` is `@ApplicationScoped` and subscribes to `CONTEXT_CHANGED` for all cases — there is no scenario where path #1 fires but path #2 doesn't. Keeping both creates duplicate EventLog entries and two parallel evaluations of the same concern.

### S4.2: Design decision

The lifecycle state stays in `CasePlanModel`, NOT on the `Milestone` definition object. `Milestone` is a definition-time construct (part of `CaseDefinition`). Adding mutable runtime state to it would be architecturally inconsistent — no other definition-time object carries runtime state. The `CasePlanModel` is the correct home for runtime lifecycle state, consistent with how `PlanItem` status is tracked.

The old milestone evaluation path (`CaseContextChangedEventHandler.milestones()`) is removed. `MilestoneLifecycleManager` is the sole milestone evaluation path.

### S4.3: Changes

**CasePlanModel interface** — update milestone methods:
- `trackMilestone(String name)` — unchanged semantics (registers as PENDING), but internal storage changes from `Boolean.FALSE` to `MilestoneLifecycleStatus.PENDING`
- `activateMilestone(String name)` — new method, transitions to ACTIVE. Valid from PENDING only. No-op with logged warning if already ACTIVE or in a terminal state (COMPLETED, FAILED, CANCELLED).
- `completeMilestone(String name)` — new method, transitions to COMPLETED. Valid from PENDING or ACTIVE — accepting PENDING→COMPLETED handles out-of-order Vert.x event delivery (MILESTONE_ACTIVATED and MILESTONE_COMPLETED are on different event bus addresses; Vert.x guarantees ordering within an address but not between addresses). No-op with logged warning if already COMPLETED.
- `getMilestoneStatus(String name)` — new method, returns `Optional<MilestoneLifecycleStatus>`
- `isMilestoneAchieved(String name)` — keep as convenience, returns `true` when status is COMPLETED (backwards-compatible)
- Deprecate `achieveMilestone(String name)` — delegates to `completeMilestone()` for backwards compatibility during transition

Methods called for an untracked milestone name (not previously registered via `trackMilestone()`) are no-ops with logged warnings. The `compute()` lambda treats `null` (untracked) the same as an invalid source state.

**DefaultCasePlanModel** — change `ConcurrentHashMap<String, Boolean>` to `ConcurrentHashMap<String, MilestoneLifecycleStatus>`. Implement the new methods. All state transitions use `ConcurrentHashMap.compute()` for atomic check-and-update — no racy get-then-put.

**MilestoneAchievementHandler** — split into two event listeners (or one handler with two `@ConsumeEvent` methods):
- `onMilestoneActivated(MilestoneActivatedEvent)` — calls `plan.activateMilestone(name)`
- `onMilestoneCompleted(MilestoneCompletedEvent)` — calls `plan.completeMilestone(name)`
- Remove the existing `onMilestoneReached(MilestoneReachedEvent)` handler — the lifecycle events replace it

**CaseContextChangedEventHandler** — remove the `milestones()` method entirely (lines 240-260). `MilestoneLifecycleManager` is the sole milestone evaluation path. This also eliminates the non-idempotent MILESTONE_REACHED firing (which fires on every context change where criteria is true, creating duplicate EventLog entries).

**MilestoneActivatedEventHandler** — add `CaseLifecycleEvent` firing (action "ActivateMilestone", event "MilestoneActivated"), following the same fire-and-forget pattern as `GoalReachedEventHandler`. This replaces the audit trail coverage previously provided by `MilestoneReachedEventHandler`.

**MilestoneCompletedEventHandler** — add `CaseLifecycleEvent` firing (action "CompleteMilestone", event "MilestoneCompleted"), following the same pattern. Together with `MilestoneActivatedEventHandler`, this provides a richer audit trail than the old single "MilestoneReached" event — observers now see activation and completion as distinct audit entries.

**MilestoneReachedEventHandler** — deprecate. With `CaseContextChangedEventHandler.milestones()` removed, no publisher fires `MILESTONE_REACHED`. The handler remains in the codebase (for any external publisher that may use the address) but is effectively dead code. `CaseHubEventType.MILESTONE_REACHED` enum value is retained for backwards compatibility with existing EventLog entries. The `CaseLifecycleEvent` previously fired by this handler ("ReachMilestone"/"MilestoneReached") is replaced by the two lifecycle events above.

**CasePlanModel Javadoc** — remove the "interim approach" note (line 33). Replace with: "Milestone lifecycle tracks PENDING → ACTIVE → COMPLETED via `MilestoneLifecycleStatus`. See `MilestoneLifecycleManager` for the event-driven state machine."

### S4.4: Behavioral differences from the old path

This spec changes when `CasePlanModel` records milestone state. Both differences are intentional corrections toward CMMN alignment:

**1. Two-context-change completion.** Under the old path (`MILESTONE_REACHED`), a milestone with always-true entry criteria and completion criteria met on the same context change was marked achieved immediately (one context change). Under the lifecycle path, `MilestoneLifecycleManager.evaluateMilestone()` evaluates either entry criteria OR completion criteria per invocation (not both), deriving current status from EventLog. The milestone transitions PENDING→ACTIVE on the first `CONTEXT_CHANGED`, then ACTIVE→COMPLETED on the next. This is the correct CMMN behavior — §5.4.7.2 requires a milestone to pass through Available before Completed.

**2. Entry criteria enforcement.** Under the old path, `CaseContextChangedEventHandler.milestones()` fired `MILESTONE_REACHED` when `completionCriteria` was true regardless of whether `entryCriteria` had been met. Under the lifecycle path, a milestone must be ACTIVATED (entry criteria met) before it can be COMPLETED. A milestone whose entry criteria is never met will never be completed, even if its completion criteria is true. This is more correct — a milestone that hasn't been activated shouldn't be completable.

### S4.5: Tests

- `trackMilestone("doc-check")` → `getMilestoneStatus("doc-check")` returns `Optional.of(PENDING)`
- `activateMilestone("doc-check")` → status transitions to ACTIVE, `isMilestoneAchieved()` returns false
- `completeMilestone("doc-check")` → status transitions to COMPLETED, `isMilestoneAchieved()` returns true
- `getMilestoneStatus("unknown")` → returns `Optional.empty()`
- `activateMilestone("doc-check")` when already ACTIVE → no-op (logged warning)
- `activateMilestone("doc-check")` when COMPLETED → no-op (logged warning)
- `completeMilestone("doc-check")` when PENDING → transitions to COMPLETED (out-of-order delivery case)
- `completeMilestone("doc-check")` when already COMPLETED → no-op (logged warning)
- Integration: `MilestoneActivatedEvent` fires → `MilestoneAchievementHandler` updates CasePlanModel to ACTIVE
- Integration: `MilestoneCompletedEvent` fires → `MilestoneAchievementHandler` updates CasePlanModel to COMPLETED
- `activateMilestone("never-tracked")` → no-op with logged warning (untracked milestone)
- `completeMilestone("never-tracked")` → no-op with logged warning (untracked milestone)
- Verify `CaseContextChangedEventHandler` no longer evaluates milestones — no `MILESTONE_REACHED` events published
- Verify `MilestoneActivatedEventHandler` fires `CaseLifecycleEvent` with action "ActivateMilestone"
- Verify `MilestoneCompletedEventHandler` fires `CaseLifecycleEvent` with action "CompleteMilestone"

## S5: 4. CMMN 1.1 Alignment (post-implementation target)

This table describes the state AFTER this spec is implemented.

| Concept | CMMN 1.1 | casehub-engine (after this spec) | Alignment | Status |
|---------|----------|----------------------------------|-----------|--------|
| **Milestone** | §5.4.7 — achievable target for progress evaluation, containable in Stage | Milestone with `parentStageId` (this spec), lifecycle PENDING→ACTIVE→COMPLETED tracked in CasePlanModel (this spec). Single evaluation path via `MilestoneLifecycleManager` (this spec). | Aligned | Changed by this spec |
| **Stage** | §5.4.4 — container for PlanItems and Milestones, nested stages allowed | Stage with `containedPlanItemIds`, `containedMilestoneIds`, `parentStageId`, repeatable. Bidirectional milestone containment wired via `CasePlanModel.trackMilestone(name, parentStageId)` (this spec) | Aligned | Changed by this spec |
| **Goal** | No explicit Goal concept — exit criteria on Case/Stage serve the role | Goal with `GoalKind` (SUCCESS/FAILURE), expression-based evaluation. Always terminal — `terminal` field removed (this spec). Goals drive case completion via `GoalBasedCompletion`. Registration-time warning for unreferenced goals (this spec). | Intentional divergence — casehub extends CMMN with explicit Goals from agent/BDI architectures. Goals map to CMMN exit criteria. | Changed by this spec |
| **Milestone lifecycle** | §5.4.7.2 — Available→Completed (two states) | PENDING→ACTIVE→COMPLETED tracked in CasePlanModel via `MilestoneLifecycleStatus` (this spec). FAILED, CANCELLED reserved for future use. Entry criteria enforced before completion (this spec). | Superset — ACTIVE maps to CMMN Available; additional states for richer lifecycle | Changed by this spec |

## S6: Files affected

| File | Change |
|------|--------|
| `api/.../model/Goal.java` | Remove `terminal` field, getter, setter, builder method. Update Javadoc. Remove from `equals()`/`hashCode()`. |
| `api/.../model/GoalBasedCompletion.java` | No code change — create follow-on issue for the TODO (generalize beyond success/failure) |
| `api/.../model/Milestone.java` | Add `parentStageId` field, getter, builder method. Add to `equals()`/`hashCode()`. |
| `schema/.../CaseDefinition.yaml` | Remove `terminal` property from Goal definition |
| `runtime/.../handler/GoalReachedEventHandler.java` | Remove `isTerminal` from EventLog metadata |
| `runtime/.../handler/CaseContextChangedEventHandler.java` | Remove `milestones()` method — superseded by `MilestoneLifecycleManager` |
| `runtime/.../handler/MilestoneActivatedEventHandler.java` | Add `CaseLifecycleEvent` firing (action "ActivateMilestone", event "MilestoneActivated") |
| `runtime/.../handler/MilestoneCompletedEventHandler.java` | Add `CaseLifecycleEvent` firing (action "CompleteMilestone", event "MilestoneCompleted") |
| `runtime/.../handler/MilestoneReachedEventHandler.java` | Deprecate — no remaining publishers for `MILESTONE_REACHED`; `CaseLifecycleEvent` replaced by lifecycle handlers above |
| `runtime/.../engine/DefaultCaseDefinitionRegistry.java` | Add registration-time WARNING for goals not referenced in any GoalExpression |
| `blackboard/.../plan/CasePlanModel.java` | Add `activateMilestone()`, `completeMilestone()`, `getMilestoneStatus()`, `trackMilestone(name, parentStageId)`. Deprecate `achieveMilestone()`. Update Javadoc. |
| `blackboard/.../plan/DefaultCasePlanModel.java` | Change milestone map from `Boolean` to `MilestoneLifecycleStatus`. Implement new methods with `compute()` atomicity and transition guards. Add stage-wiring overload. |
| `blackboard/.../handler/MilestoneAchievementHandler.java` | Replace `MILESTONE_REACHED` listener with `MILESTONE_ACTIVATED` and `MILESTONE_COMPLETED` listeners |
| Tests (api, runtime, blackboard) | Goal terminal removal; milestone parentStageId containment; milestone lifecycle state transitions; transition guards; out-of-order delivery; unreferenced goal warning |