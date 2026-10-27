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
bindings:
  - name: extract-a
    capability: extract
    on: { scopeActivated: {} }

  - name: extract-b
    capability: extract
    on: { scopeActivated: {} }

  - name: extract-c
    capability: extract
    on: { scopeActivated: {} }

  - name: aggregate
    capability: aggregate
    on: { scopeActivated: {} }
    requires:
      bindings: [extract-a, extract-b, extract-c]
      atLeast: 2
      onThresholdReached: KEEP    # optional, default KEEP
```

The `aggregate` binding fires on scope activation but is held by `PlanningStrategyLoopControl` until at least 2 of the 3 extractors have COMPLETED.

`requires` is a guard (like `when`), not a trigger. All bindings still need a trigger. For pipeline barriers where the quorum IS the activation reason, use `scopeActivated` as the trigger and `requires` as the gate.

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

Add `@Nullable BindingRequires requires` field to `Binding` record.

#### Load-time validation

Add `validateRequiresBindings(List<Binding>)` to `Binding.java`, modelled after `validateCompensationBindings()` (lines 326–381):

1. Build `byName` map of all bindings
2. **Self-reference check:** `requires.bindings` must not contain the binding's own name
3. **Existence check:** every name in `requires.bindings` must exist in `byName`
4. **Cycle detection:** visited-set traversal through `requires` chains — if binding A requires B and B requires A, report cycle
5. Return `List<String>` of errors

Call from the same validation point that calls `validateCompensationBindings()`.

#### Runtime evaluation

In `PlanningStrategyLoopControl.select()`, insert a new filter step after compound gating (line 112, after `gatedEligible`) and before `applyImplementationRouting`:

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

When the quorum threshold is met and `onThresholdReached == CANCEL`:

After the `requires` filter passes a binding through, check if the threshold was *just* met (completed count == atLeast). If so, cancel remaining non-terminal plan items for the referenced bindings:

```java
private void handleThresholdReached(Binding binding, CasePlanModel plan) {
    if (binding.requires() == null) return;
    BindingRequires req = binding.requires();
    if (req.onThresholdReached() != OnThresholdReached.CANCEL) return;

    req.bindings().stream()
        .map(plan::findPlanItemByBindingName)
        .filter(Optional::isPresent)
        .map(Optional::get)
        .filter(pi -> !pi.status().isTerminal())
        .forEach(PlanItem::markCancelled);
}
```

This follows the `SubCaseCompletionService.cancelPlanItemOnRejected()` pattern (line 266).

#### Re-evaluation timing

When a referenced binding completes (or reaches a terminal state), the next `PlanningStrategyLoopControl.select()` cycle re-evaluates all `requires`-gated bindings. No explicit notification is needed — the planning loop already runs on every context change, and worker completion triggers a context change via `WorkflowExecutionCompletedHandler`.

#### Unreachable quorum detection

When evaluating `requires`, also detect unreachable quorum using the `SubCaseGroupPolicy` formula:

```java
long nonCompletableTerminal = req.bindings().stream()
    .map(plan::findPlanItemByBindingName)
    .filter(Optional::isPresent)
    .map(Optional::get)
    .filter(pi -> pi.status().isTerminal() && pi.status() != TaskStatus.COMPLETED)
    .count();

long remaining = req.bindings().size() - completed - nonCompletableTerminal;
long needed = req.atLeast() - completed;

if (remaining < needed) {
    // quorum unreachable — mark this binding's plan item FAULTED
}
```

#### Annotation support

Add `requires` to `@Binding` annotation (or equivalent) per api-surface-parity protocol. The annotation mirrors the YAML structure:

```java
@interface Requires {
    String[] bindings();
    int atLeast();
    OnThresholdReached onThresholdReached() default OnThresholdReached.KEEP;
}
```

#### CoordinationPipelineProjection

Update `CoordinationPipelineProjection` (rest-core) to detect `requires` on bindings and emit `PhaseItemView` entries with `required`/`of` populated. This provides the backing data that blocks-ui's quorum phase items currently fabricate.

### Stream 2: Compound MofN YAML wiring fix

Two issues to fix:

#### Issue A: `mofnRequired` dropped during conversion

`YamlCaseDefinitionConverter.convertCompound()` (line 1315) reads `yc.completionSemantics()` but ignores `yc.mofnRequired()`. The `CompoundDeclaration` record accepts `completionSemantics` as a string where an integer value (e.g. `"3"`) means MofN.

**Fix:** In `convertCompound()`, if `yc.mofnRequired() != null`:
- If `yc.completionSemantics()` is null or `"mofn"`: set `completionSemantics = String.valueOf(yc.mofnRequired())`
- If `yc.completionSemantics()` is already a numeric string: validate it matches `mofnRequired`, warn on mismatch
- Pass the resolved value to `CompoundDeclaration` constructor

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

Per protocol PP-20260915-aa504e, all three pathways must expose `requires`:

| Pathway | Mechanism |
|---------|-----------|
| YAML | `requires` property on Binding with `BindingRequires` $def |
| Java builder | `BindingRequires` record, `Binding.Builder.requires(BindingRequires)` |
| Annotations | `@Requires` annotation on binding declaration |

## Testing Strategy

| Test | Module | What it validates |
|------|--------|-------------------|
| `BindingRequiresValidationTest` | api | Self-reference, existence, cycle detection |
| `RequiresFilterTest` | planning-core | Pre-strategy filter: eligible/blocked/threshold transitions |
| `RequiresUnreachableTest` | planning-core | Unreachable quorum detection and FAULTED transition |
| `RequiresCancelTest` | planning-core | OnThresholdReached.CANCEL cancels remaining plan items |
| `RequiresIntegrationTest` | runtime | End-to-end: YAML definition → binding dispatch gating → quorum met → aggregator fires |
| `CompoundMofnYamlWiringTest` | planning-core | YAML compound with completionSemantics parses and registers correctly |
| `CompoundBlackboardPlanConfigurerTest` | planning-core | Compound registration from CaseDefinition |
| `CoordinationPipelineQuorumTest` | rest-core | PhaseItemView emits required/of for requires bindings |

## Protocols

- **api-surface-parity** (PP-20260915-aa504e): YAML, builder, and annotations all expose `requires`
- **plan-type-module-boundary** (PP-20260727-5267d2): `BindingRequires` is a definition type consumers reference → `engine-api`. Evaluation logic is internal → `planning-core`.

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
- blocks-ui#258 — gallery demo data should come from real YAML
- blocks-ui#243 — coordination diff view renders quorum changes
