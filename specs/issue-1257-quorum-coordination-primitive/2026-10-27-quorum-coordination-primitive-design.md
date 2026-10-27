# Quorum Coordination Primitive — Design Spec

**Issue:** casehubio/engine#1257
**Branch:** issue-1257-quorum-coordination-primitive
**Date:** 2026-10-27

## Problem

Workers in the engine coordinate via choreography (all eligible fire) or sequence (one at a time). There is no "wait for N of M workers" barrier at the binding level.

Compound `CompletionSemantics.MofN` exists but gates compound *completion*, not binding *dispatch*. It answers "when is this group done?" — not "when should the next binding fire?"

The blocks-ui `CoordinationPipeline` type fabricates quorum phase items (`{ type: 'quorum', required, of }`) with no engine backing.

A secondary bug: compound-level MofN declared in YAML is silently dropped — `YamlCompound.mofnRequired()` is never read during conversion, and no `BlackboardPlanConfigurer` bridges YAML compounds to runtime `PlanItemDefinition.Compound`.

## Scope

Two work streams:

1. **Binding-level `requires` guard** — new coordination primitive
2. **Compound MofN YAML wiring fix** — bug fix for existing infrastructure

## Design

### Stream 1: Binding-level `requires` guard

#### YAML DSL

A new `requires` property on bindings declares prerequisites:

```yaml
compounds:
  - name: extraction-pipeline
    scopedBindings:
      extract-a: PARTICIPANT
      extract-b: PARTICIPANT
      extract-c: PARTICIPANT
      aggregate: PARTICIPANT

bindings:
  - name: extract-a
    capability: extract
    lifecycleScope: COMPOUND
    on: { scopeActivated: {} }

  - name: extract-b
    capability: extract
    lifecycleScope: COMPOUND
    on: { scopeActivated: {} }

  - name: extract-c
    capability: extract
    lifecycleScope: COMPOUND
    on: { scopeActivated: {} }

  - name: aggregate
    capability: aggregate
    lifecycleScope: COMPOUND
    on: { scopeActivated: {} }
    requires:
      bindings: [extract-a, extract-b, extract-c]
      atLeast: 2
      onThresholdReached: KEEP    # optional, default KEEP
```

The `aggregate` binding fires when the `extraction-pipeline` compound activates but is held by `PlanningStrategyLoopControl` until at least 2 of the 3 extractors have COMPLETED. All bindings use `lifecycleScope: COMPOUND` because `scopeActivated` requires COMPOUND or CASE scope — `Binding.Builder.build()` rejects `ScopeActivatedTrigger` with the default `BINDING` scope.

`requires` is a guard (like `when`), not a trigger. All bindings still need a trigger. For pipeline barriers where the quorum IS the activation reason, use `scopeActivated` as the trigger and `requires` as the gate.

#### Scope constraints

`requires` references must be **co-scoped** — the referenced bindings must be in the same compound as the binding with the `requires` clause, or both must be at case scope (no compound). Cross-compound references are forbidden because compound activation is runtime-dependent, making it impossible to guarantee that the referenced binding's compound will ever activate.

Bindings in repeatable compounds must not use `requires`. Repeatable compounds create new PlanItems per cycle; `findPlanItemByBindingName()` returns the latest PlanItem, which may be PENDING (new cycle) while the previous cycle's item was COMPLETED. This makes the quorum check unreliable across repetition boundaries.

#### Schema changes

Add to `CaseDefinition.yaml` `$defs`:

```yaml
BindingRequires:
  type: object
  unevaluatedProperties: false
  required:
    - bindings
    - atLeast
  properties:
    bindings:
      type: array
      items:
        type: string
        minLength: 1
      minItems: 2
      description: Binding names to wait for.
    atLeast:
      type: integer
      minimum: 1
      description: Number of referenced bindings that must COMPLETE before this binding becomes eligible.
    onThresholdReached:
      type: string
      enum:
        - KEEP
        - CANCEL
      default: KEEP
      description: What happens to remaining referenced bindings once the threshold is met.
```

Add to `Binding` properties:

```yaml
requires:
  $ref: "#/$defs/BindingRequires"
  description: Quorum prerequisite — binding is held until atLeast of the referenced bindings have COMPLETED.
```

#### YAML record mapping

Add to `yaml-record-mappings.yaml` under `Binding`:

```yaml
Binding:
  record: YamlBinding
  fields:
    # ... existing fields ...
    requires: { type: YamlBindingRequires }
```

New type:

```yaml
YamlBindingRequires:
  record: YamlBindingRequires
  extra:
    - { name: bindings, type: "List<String>" }
    - { name: atLeast, type: Integer }
    - { name: onThresholdReached, type: String }
```

#### API model

New record in `io.casehub.api.model`:

```java
public record BindingRequires(
    List<String> bindings,
    int atLeast,
    OnThresholdReached onThresholdReached
) {
    public BindingRequires {
        Objects.requireNonNull(bindings);
        if (bindings.size() < 2) throw new IllegalArgumentException("requires.bindings must have at least 2 entries");
        if (atLeast < 1 || atLeast > bindings.size()) throw new IllegalArgumentException("atLeast must be between 1 and bindings.size()");
        if (onThresholdReached == null) onThresholdReached = OnThresholdReached.KEEP;
    }
}
```

Add to `Binding` class (`io.casehub.api.model.Binding` — a mutable class with private fields, getters/setters, and a `Builder`):

1. Private field: `private BindingRequires requires;`
2. Getter: `public BindingRequires getRequires()` (matching `getCompensatedBy()`, `getOutcomePolicy()` conventions)
3. Setter: `public void setRequires(BindingRequires requires)` (called from `Builder.build()`)
4. Convenience accessor: `public BindingRequires requires()` (matching the existing `target()` convention)
5. Builder method: `public Builder requires(BindingRequires requires)` in `Binding.Builder`
6. Builder validation: `requires` is only valid with `ScopeActivatedTrigger` or `ContextChangeTrigger` — not with `CronTrigger` or `EventTrigger`

#### Load-time validation

Add `validateRequiresBindings(List<Binding>, List<CompoundDeclaration>)` to `Binding.java`, modelled after `validateCompensationBindings()` (lines 326–381):

1. Build `byName` map of all bindings
2. **Duplicate check:** `requires.bindings` must not contain duplicate names — `extract-a` can only complete once, so `atLeast: 2` with `[extract-a, extract-a]` is unreachable by definition. Deduplicate and re-validate `atLeast` against the deduplicated count.
3. **Self-reference check:** `requires.bindings` must not contain the binding's own name
4. **Existence check:** every name in `requires.bindings` must exist in `byName`
5. **Scope compatibility check:** all referenced bindings must be co-scoped with the binding declaring `requires` — same compound or both at case scope. Build a `bindingToCompound` map from `CompoundDeclaration.scopedBindings()` and verify all referenced names map to the same compound (or all are unscoped).
6. **Repeatable compound check:** if the binding is in a compound with `repeatable: true`, report an error — `requires` is not supported in repeatable compounds (see §Scope constraints).
7. **Cycle detection:** visited-set traversal through `requires` chains — if binding A requires B and B requires A, report cycle
8. Return `List<String>` of errors

Call from the same validation point that calls `validateCompensationBindings()`.

#### Runtime evaluation

**Evaluation pipeline order.** The full binding eligibility pipeline in `PlanningStrategyLoopControl.select()` is:

1. **Trigger** — `scopeActivated` or `contextChange` determines the candidate set
2. **`when` guard** — evaluated inside `collectScopeActivatedBindings()` or by the trigger engine
3. **Compound gating** — bindings in inactive compounds are filtered out
4. **`requires` guard** — NEW: bindings whose prerequisite quorum is unsatisfied are filtered out
5. **Implementation routing** — `applyImplementationRouting()` selects among competing implementations
6. **PlanItem creation** — `addPlanItemIfAbsent()` registers the binding in the plan model
7. **Compound dispatch** — `compoundDispatcher.dispatch()` applies compound-level dispatch strategy

Insert the `requires` filter after compound gating (line 112, after `gatedEligible`) and before `applyImplementationRouting`:

```java
List<Binding> requiresSatisfied = filterByRequires(gatedEligible, plan);
```

The filter method:

```java
private List<Binding> filterByRequires(List<Binding> eligible, CasePlanModel plan) {
    return eligible.stream()
        .filter(b -> isRequiresSatisfied(b, plan))
        .toList();
}

private boolean isRequiresSatisfied(Binding binding, CasePlanModel plan) {
    if (binding.requires() == null) return true;
    BindingRequires req = binding.requires();
    long completed = req.bindings().stream()
        .map(plan::findPlanItemByBindingName)
        .filter(Optional::isPresent)
        .map(Optional::get)
        .filter(pi -> pi.status() == TaskStatus.COMPLETED)
        .count();
    return completed >= req.atLeast();
}
```

#### Threshold reached handling

Cancellation when `onThresholdReached == CANCEL` must NOT run inside `select()` — `select()` is a query method that determines dispatch eligibility. Inserting side effects (PlanItem cancellation) violates its contract and diverges from every other cancellation pattern in the codebase. `SubCaseCompletionService.cancelPlanItemOnRejected()` (line 266) is called from a dedicated event handler chain, not from within selection logic.

**Design:** Create a `RequiresThresholdEvaluator` in `planning-core`, invoked from `PlanItemCompletionHandler.onWorkerFinished()` after marking the prerequisite PlanItem COMPLETED — the same integration point where `CompoundCompletionEvaluator.evaluate()` is called.

```java
public class RequiresThresholdEvaluator {

    public void evaluate(UUID caseId, CasePlanModel plan, CaseDefinition definition,
                         String completedBindingName) {
        for (Binding b : definition.getBindings()) {
            if (b.getRequires() == null) continue;
            if (!b.getRequires().bindings().contains(completedBindingName)) continue;
            if (b.getRequires().onThresholdReached() != OnThresholdReached.CANCEL) continue;

            long completed = countCompleted(b.getRequires(), plan);
            if (completed >= b.getRequires().atLeast()
                    && !plan.isRequiresThresholdTriggered(b.getName())) {
                plan.markRequiresThresholdTriggered(b.getName());
                cancelRemaining(b.getRequires(), plan);
            }
        }
    }

    private void cancelRemaining(BindingRequires req, CasePlanModel plan) {
        req.bindings().stream()
            .map(plan::findPlanItemByBindingName)
            .filter(Optional::isPresent)
            .map(Optional::get)
            .filter(pi -> !pi.status().isTerminal())
            .forEach(PlanItem::markCancelled);
    }
}
```

The `>=` check (not `==`) prevents missed thresholds when multiple prerequisite bindings complete between evaluation cycles. The one-shot `isRequiresThresholdTriggered` / `markRequiresThresholdTriggered` flag on `CasePlanModel` prevents redundant re-cancellation, following the `SubCaseGroup.policyTriggered()` pattern (PP-20260513-ecabd7). The cancellation itself is idempotent (filters to non-terminal items), so the flag is for efficiency, not correctness.

#### Re-evaluation timing

When a referenced binding completes (or reaches a terminal state), the next `PlanningStrategyLoopControl.select()` cycle re-evaluates all `requires`-gated bindings. No explicit notification is needed — the planning loop already runs on every context change, and worker completion triggers a context change via `WorkflowExecutionCompletedHandler`.

#### Unreachable quorum detection

Unreachable quorum detection runs in `RequiresThresholdEvaluator`, not in `select()`. When a prerequisite binding reaches a terminal non-COMPLETED state (FAULTED, CANCELLED, REJECTED), `RequiresThresholdEvaluator` checks if any gated binding's quorum is now unreachable using the `SubCaseGroupPolicy` formula:

```java
public void evaluateUnreachable(UUID caseId, CasePlanModel plan,
                                 CaseDefinition definition, String faultedBindingName) {
    for (Binding b : definition.getBindings()) {
        if (b.getRequires() == null) continue;
        if (!b.getRequires().bindings().contains(faultedBindingName)) continue;

        BindingRequires req = b.getRequires();
        long completed = countCompleted(req, plan);
        long nonCompletableTerminal = countNonCompletableTerminal(req, plan);
        long remaining = req.bindings().size() - completed - nonCompletableTerminal;
        long needed = req.atLeast() - completed;

        if (remaining < needed) {
            // Create PlanItem and immediately fault it — the gated binding was
            // never dispatched so no PlanItem exists yet. This makes the fault
            // observable in the plan model and triggers compound completion evaluation.
            PlanItem pi = PlanItem.create(b.getName(), ExecutorRef.NONE, 0, b.target());
            plan.addPlanItemIfAbsent(pi);
            pi.markFaulted();
        }
    }
}
```

The `requires`-gated binding's PlanItem does not exist at this point because `filterByRequires` removed it from the eligible set before `addPlanItemIfAbsent` ran. The evaluator explicitly creates a PlanItem and immediately faults it, making the unreachable quorum observable in the plan model. This triggers compound completion evaluation via `CompoundCompletionEvaluator`.

`evaluateUnreachable()` is called from the fault/cancellation handlers (e.g., `WorkerOutcomeResolvedHandler`) when a PlanItem reaches a non-COMPLETED terminal state — mirroring how `evaluate()` is called from the completion handler.

#### Annotation support

Add `requires` to the `@Bind` annotation (`io.casehub.engine.annotations.Bind` — the `@Repeatable` binding annotation with `@Bindings` as container). The annotation currently has attributes like `capability()`, `contextChange()`, `scopeActivated()`, `when()`, etc. but no `name()` attribute.

Since `requires` references other bindings by name, and `@Bind` currently has no `name()` attribute, two additions are needed:

1. Add `String name() default ""` to `@Bind` — allows naming a binding for reference by other bindings' `requires`.
2. Add `@Requires requires() default @Requires(bindings = {})` to `@Bind` — the prerequisite declaration.

```java
@interface Requires {
    String[] bindings();
    int atLeast() default 0;  // 0 means "all" — atLeast == bindings.length
    OnThresholdReached onThresholdReached() default OnThresholdReached.KEEP;
}
```

Usage:

```java
@Bind(name = "aggregate", capability = "aggregate", scopeActivated = true,
      requires = @Requires(bindings = {"extract-a", "extract-b"}, atLeast = 2))
```

The `BindingDescriptor` record and `AnnotationCaseDefinitionBuilder` must be updated to read `@Requires` from `@Bind` and construct `BindingRequires` accordingly.

#### CoordinationPipelineProjection

Update `CoordinationPipelineProjection` (rest-core) to detect `requires` on bindings and emit `PhaseItemView` entries with `required`/`of` populated. `PhaseItemView` already has `Integer required` and `Integer of` fields (currently always null).

The `requires`-gated binding is emitted as `type: "step"` (same as any other binding) with `required` and `of` populated:

```java
private PhaseItemView stepPhase(Binding b) {
    CoordinationStepView step = bindingToStep(b);
    Integer required = b.getRequires() != null ? b.getRequires().atLeast() : null;
    Integer of = b.getRequires() != null ? b.getRequires().bindings().size() : null;
    return new PhaseItemView(
        "step", step.action(), step.target(), step.model(), step.outputFormat(),
        step.channel(), step.params(), null, null, required, of, null);
}
```

This provides the engine-backed data that blocks-ui's `CoordinationPipeline` quorum phase items (`{ type: 'quorum', required, of }`) currently fabricate. blocks-ui can switch from fabrication to reading these fields from the projection response.

### Stream 2: Compound MofN YAML wiring fix

Two issues to fix:

#### Issue A: `mofnRequired` dropped during conversion

`YamlCaseDefinitionConverter.convertCompound()` (line 1315) reads `yc.completionSemantics()` but ignores `yc.mofnRequired()`. The `CompoundDeclaration` record accepts `completionSemantics` as a string where an integer value (e.g. `"3"`) means MofN.

Three changes are needed — not just the converter fix:

**1. Schema:** Add `mofnRequired` to the `CompoundDeclaration` properties in `CaseDefinition.yaml`. Currently, `CompoundDeclaration` uses `unevaluatedProperties: false` (line 267), which means any YAML with `mofnRequired` on a compound is rejected at schema validation time. The field exists in `yaml-record-mappings.yaml` (line 190) as an `extra` field but was never added to the JSON Schema:

```yaml
CompoundDeclaration:
  type: object
  properties:
    # ... existing properties ...
    mofnRequired:
      type: integer
      minimum: 1
      description: Number of scoped bindings that must complete for MofN completion semantics.
  unevaluatedProperties: false
```

**2. Converter:** In `convertCompound()`, if `yc.mofnRequired() != null`:
- If `yc.completionSemantics()` is null or `"mofn"`: set `completionSemantics = String.valueOf(yc.mofnRequired())`
- If `yc.completionSemantics()` is already a numeric string: validate it matches `mofnRequired`, warn on mismatch
- Pass the resolved value to `CompoundDeclaration` constructor

**3. Validation:** Add validation that `mofnRequired <= scopedBindings.size()` when `completionSemantics` is `mofn`.

This preserves both YAML styles: `completionSemantics: "3"` (direct) and `completionSemantics: mofn` + `mofnRequired: 3` (explicit).

#### Issue B: No BlackboardPlanConfigurer for YAML compounds

No production `BlackboardPlanConfigurer` implementation bridges `CaseDefinition.getCompounds()` to `PlanItemDefinition.Compound` registration in the `CasePlanModel`.

**Fix:** Implement a `CompoundBlackboardPlanConfigurer` in planning-core that:
1. Reads `CaseDefinition.getCompounds()` from the `PlanExecutionContext`
2. For each `CompoundDeclaration`, creates a `PlanItemDefinition.Compound` with:
   - `completionSemantics` parsed from the string (All / MofN / FirstWins)
   - `dispatchMode` parsed from string
   - `scopedBindings` mapped to `Participation` enum
   - `entryCondition` / `exitCondition` preserved
3. Registers each compound in the `CasePlanModel` via `registerCompound()`

This follows the `BlackboardPlanConfigurer` SPI contract — `PlanningStrategyLoopControl` calls all `BlackboardPlanConfigurer` instances on first plan model creation.

## API Surface Parity

All three pathways must expose `requires` to maintain parity across the definition surface:

| Pathway | Mechanism |
|---------|-----------|
| YAML | `requires` property on Binding with `BindingRequires` $def |
| Java builder | `BindingRequires` record, `Binding.Builder.requires(BindingRequires)` |
| Annotations | `@Requires` annotation on `@Bind` declaration |

## Testing Strategy

| Test | Module | What it validates |
|------|--------|-------------------|
| `BindingRequiresValidationTest` | api | Self-reference, existence, cycle detection, duplicate names, scope compatibility |
| `RequiresFilterTest` | planning-core | Pre-strategy filter: eligible/blocked/threshold transitions |
| `RequiresUnreachableTest` | planning-core | Unreachable quorum detection and FAULTED transition |
| `RequiresThresholdEvaluatorTest` | planning-core | Threshold evaluation: one-shot cancellation, idempotent re-evaluation, unreachable quorum faulting |
| `RequiresCancelTest` | planning-core | OnThresholdReached.CANCEL cancels remaining plan items |
| `RequiresIntegrationTest` | runtime | End-to-end: YAML definition → binding dispatch gating → quorum met → aggregator fires |
| `CompoundMofnYamlWiringTest` | planning-core | YAML compound with completionSemantics parses and registers correctly |
| `CompoundBlackboardPlanConfigurerTest` | planning-core | Compound registration from CaseDefinition |
| `CoordinationPipelineQuorumTest` | rest-core | PhaseItemView emits required/of for requires bindings |

## Protocols

- **subcase-coordination-strategy** (PP-20260513-ecabd7): The `requires` guard IS threshold counting. Per this protocol, threshold counting uses "a persisted counter + policyTriggered flag behind a repository SPI." The `requires` evaluation runs behind the `LoopControl` SPI; the threshold state is tracked in `CasePlanModel`; cancellation uses a one-shot `thresholdTriggered` flag following `SubCaseGroup.policyTriggered()`. The SPI requirement is satisfied by `LoopControl` (evaluation) and `CasePlanModel` (state) — no additional SPI is needed because the threshold state is in-memory per case instance, not cross-JVM.
- **Module boundary:** `BindingRequires` is a definition type consumers reference → `engine-api`. `RequiresThresholdEvaluator` and evaluation logic are internal → `planning-core`.
- **case-definition-layers** (FOUNDATION-INDEX): The `requires` property flows through the three-layer architecture: YAML schema → `YamlBindingRequires` record → `BindingRequires` via `YamlCaseDefinitionConverter`.

## References

- `Binding.validateCompensationBindings()` (api, Binding.java:326–381) — validation pattern precedent
- `PlanningStrategyLoopControl.select()` (planning-core) — dispatch pipeline, insertion point at line 112
- `CasePlanModel.findPlanItemByBindingName()` (planning-core, CasePlanModel.java:66) — plan item lookup
- `PlanItem.markCancelled()` (planning-core, PlanItem.java:315) — CAS-guarded cancellation
- `SubCaseGroupPolicy.evaluate()` (planning-core) — unreachable quorum formula pattern
- `SubCaseCompletionService.cancelPlanItemOnRejected()` (planning-core:266) — cancel remaining pattern
- `OnThresholdReached` enum (api) — shared KEEP/CANCEL semantics
- `CompoundDeclaration` record (api) — 8 fields, completionSemantics as string
- `YamlCaseDefinitionConverter.convertCompound()` (api:1315) — mofnRequired drop site
- `BlackboardPlanConfigurer` SPI (planning-core) — compound registration hook
- `CoordinationPipelineProjection` (rest-core) — PhaseItemView projection
- `PlanItemCompletionHandler` (planning-core) — completion handler integration point for `RequiresThresholdEvaluator`
- `SubCaseGroup.policyTriggered()` (common-core) — one-shot flag pattern for threshold evaluation
- `@Bind` annotation (annotations/runtime, Bind.java) — `@Repeatable` binding annotation
- blocks-ui#258 — gallery demo data should come from real YAML
- blocks-ui#243 — coordination diff view renders quorum changes

## Deferred Concerns — GitHub Issues

The following out-of-scope items must be filed as GitHub issues:

1. **Cross-compound `requires` dependencies** — semantics for referencing bindings in a different compound. Blocked by compound lifecycle formalization. File as `engine#TBD`.
2. **`requires` in repeatable compounds** — `findPlanItemByBindingName()` returns the latest PlanItem, which breaks quorum checks across repetition cycles. Needs a per-cycle scoping mechanism. File as `engine#TBD`.
3. **Protocol creation: api-surface-parity** — the principle (YAML, builder, and annotations all expose the same features) is real but the referenced protocol (PP-20260915-aa504e) does not exist. File as `parent#TBD` to create the protocol.
4. **Protocol creation: plan-type-module-boundary** — the principle (definition types in api, evaluation in planning-core) is real but the referenced protocol (PP-20260727-5267d2) does not exist. File as `parent#TBD` to create the protocol.
