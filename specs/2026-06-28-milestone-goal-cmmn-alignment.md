# Milestone, Goal, and Stage — Full Conceptual Alignment

**Issue:** [#581](https://github.com/casehubio/engine/issues/581)  
**Epic:** [#84](https://github.com/casehubio/engine/issues/84)  
**Date:** 2026-06-28

## Summary

Three remaining gaps prevent closing epic #84:

1. `Goal.terminal` exists in the model and YAML schema but is never read by the mapper and never gates case completion
2. `Milestone` has no `parentStageId` — Stage tracks contained milestones but Milestone has no back-pointer
3. No CMMN 1.1 alignment documentation

## 1. Goal.terminal — wire the existing design

### Problem

`Goal.terminal` defaults to `false` in both the Java model (`Goal.java:64`) and the YAML schema (`CaseDefinition.yaml:429-432`). The YAML mapper (`CaseDefinitionYamlMapper:354-363`) does not read `sg.getTerminal()` — the value is silently dropped. `GoalReachedEventHandler` writes `isTerminal` to EventLog metadata but does not gate case completion on it — all reached goals evaluate completion regardless.

### Design decision

Non-terminal goals are a distinct concept from milestones. A milestone is observational ("we reached point X"). A non-terminal goal is intentional with polarity ("we were trying to achieve X, and did/didn't"). This distinction matters for CBR learning, outcome observation, and stage-scoped objectives. The field stays — the fix is to wire it properly.

### Changes

**CaseDefinitionYamlMapper** — add `goal.setTerminal(sg.getTerminal())` when converting schema Goal to API Goal.

**GoalReachedEventHandler** — gate case completion evaluation on `goal.getTerminal()`:
- Terminal goal reached → existing behaviour (evaluate `CaseStatus.COMPLETED` or `CaseStatus.FAULTED`)
- Non-terminal goal reached → publish `GoalReachedEvent`, write EventLog with `isTerminal: false`, skip completion evaluation

### Tests

- Case definition with one terminal goal (`.amount > 1000`) and one non-terminal goal (`.entityResolved == true`)
- Trigger non-terminal goal: verify event fires, EventLog written, case stays RUNNING
- Trigger terminal goal: verify case completes/faults as before

## 2. Milestone parentStageId — programmatic containment

### Problem

`Stage.containedMilestoneIds` exists (forward pointer) but `Milestone` has no `parentStageId` (back-pointer). `Stage.addMilestone()` is defined but never called from anywhere.

### Context

Stage is a blackboard-internal Java construct — it is not in the YAML schema and not in the API model. Stages are built programmatically in Java CaseHub subclasses, not declared in YAML. Milestone-stage containment should follow the same pattern.

YAML nesting of milestones inside stage blocks becomes relevant when/if stages are added to the YAML schema — a follow-on concern, not this issue.

### Changes

**Milestone.java** — add `parentStageId` (String) field, getter, and builder method. Immutable after construction. Null for case-level milestones not owned by any stage.

**Blackboard wiring** — when stages are activated and milestones are registered, milestones whose `parentStageId` matches a stage name are auto-registered via `stage.addMilestone(milestoneName)`. This wires the bidirectional relationship (milestone knows its parent, stage knows its contained milestones).

**No YAML schema changes** — stages are not in YAML; milestone containment is programmatic.

**Not in scope:** Stage exit criteria referencing milestone achievement — `StageAutocompleteEvaluator` already has `containedMilestoneIds` for future use but wiring that evaluation is a follow-on concern.

### Tests

- Milestone built with `parentStageId("kyc-stage")` — verify getter returns the stage name
- Milestone built without `parentStageId` — verify getter returns null
- Stage with contained milestone — verify bidirectional: `milestone.getParentStageId()` matches stage name, `stage.getContainedMilestoneIds()` includes the milestone

## 3. CMMN 1.1 Alignment

| Concept | CMMN 1.1 | casehub-engine | Alignment |
|---------|----------|----------------|-----------|
| **Milestone** | §5.4.7 — achievable target for progress evaluation, containable in Stage | Milestone with `parentStageId`, lifecycle PENDING→ACTIVE→COMPLETED, containable in stages via programmatic API | Aligned |
| **Stage** | §5.4.4 — container for PlanItems and Milestones, nested stages allowed | Stage with `containedPlanItemIds`, `containedMilestoneIds`, `parentStageId`, repeatable | Aligned |
| **Goal** | No explicit Goal concept — exit criteria on Case/Stage serve the role | Goal with `GoalKind` (SUCCESS/FAILURE), `terminal` flag, expression-based evaluation | **Intentional divergence** — casehub extends CMMN with explicit Goals from agent/BDI architectures. Terminal goals map to CMMN exit criteria. Non-terminal goals have no CMMN analog — they represent intermediate objectives with polarity. |
| **Milestone lifecycle** | §5.4.7.2 — Available→Completed (two states) | PENDING→ACTIVE→COMPLETED (+ FAILED, CANCELLED for future use) | **Superset** — ACTIVE maps to CMMN Available; additional states for richer lifecycle |

## Files affected

| File | Change |
|------|--------|
| `api/.../model/Milestone.java` | Add `parentStageId` field, getter, builder method |
| `api/.../model/converter/CaseDefinitionYamlMapper.java` | Read `Goal.terminal` from schema |
| `runtime/.../handler/GoalReachedEventHandler.java` | Gate completion evaluation on `goal.getTerminal()` |
| Blackboard milestone registration | Wire `parentStageId` → `stage.addMilestone()` during milestone registration |
| Tests (runtime, blackboard) | Goal terminal gating; milestone stage containment |
