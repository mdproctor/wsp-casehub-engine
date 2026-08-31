# YAML HTN Decomposition Tree — Design Spec

**Issue:** #987
**Parent epic:** #978 (pure-YAML execution model examples and DSL completeness)

## Problem

HTN decomposition is entirely runtime-generated today — `GoapDecompositionStrategy`, `LlmDecompositionStrategy`, and `PortfolioDecompositionStrategy` produce the task tree dynamically. There's no way to declare an explicit HTN tree in YAML. Cases with known-upfront decomposition (incident response playbooks, approval workflows, multi-phase assessments) require Java code to define the task hierarchy.

## Solution

Add a `spec.decomposition:` YAML block that declares an explicit HTN tree — compound tasks with guard-gated methods decomposing into leaf tasks (capability references) or nested compound tasks. The tree maps directly to the existing `CompoundTask<T>` / `DecompositionMethod<T>` / `LeafTask<T>` type hierarchy in `engine-api`.

## YAML Shape

```yaml
name: incident-response
namespace: io.casehub.examples
version: "1.0"
spec:
  capabilities:
    - name: triage-assessment
    - name: escalation
    - name: incident-resolution
    - name: auto-resolution

  decomposition:
    root:
      name: investigate-incident
      methods:
        - name: high-severity
          guardLabel: "High severity path"
          guard: ".severity == 'high'"
          tasks:
            - name: triage
              capability: triage-assessment
            - name: escalate
              capability: escalation
            - name: resolve
              capability: incident-resolution
        - name: low-severity
          guardLabel: "Low severity path"
          guard: ".severity == 'low'"
          tasks:
            - name: triage
              capability: triage-assessment
            - name: auto-resolve
              capability: auto-resolution
```

### Nested compound example

```yaml
  decomposition:
    root:
      name: loan-application
      methods:
        - guardLabel: "Standard path"
          guard: ".amount < 100000"
          tasks:
            - name: credit-check
              capability: credit-scoring
            - name: approval-decision
              methods:
                - guardLabel: "Auto-approve"
                  guard: ".creditScore > 750"
                  tasks:
                    - name: auto-approve
                      capability: auto-approval
                - guardLabel: "Manual review"
                  guard: ".creditScore <= 750"
                  tasks:
                    - name: underwrite
                      capability: underwriting
                    - name: manual-review
                      capability: manual-review
```

A task node is a **leaf** when it has `capability:` and no `methods:`. It is a **compound** when it has `methods:` (with or without `capability:`).

### Guard expressions

Guards are JQ expressions by default (matching `expressionLang` on the definition). They evaluate against the case context working layer — the same surface as binding `when:` conditions. The `{lang: expr}` map syntax from `resolveExpression()` is also supported for non-JQ languages.

A method with no `guard:` is unconditional — it always matches. Methods are evaluated in declaration order; the first matching method is selected (consistent with HTN semantics).

### Optional fields on leaf tasks

```yaml
- name: triage
  capability: triage-assessment
  description: "Assess incident severity and assign priority"
  estimatedDuration: PT5M       # ISO-8601 duration
  estimatedCost:
    tokens: 1000
    apiCalls: 2
```

`description`, `estimatedDuration`, and `estimatedCost` are optional — carried through to `GoalStep` / `LeafTask` for planner hints and UI display.

### Optional fields on methods

```yaml
methods:
  - name: high-severity
    guardLabel: "High severity path"
    guard: ".severity == 'high'"
    estimatedDuration: PT30M
    estimatedCost:
      tokens: 5000
    tasks: [...]
```

## Type Mapping

| YAML | Java type | Notes |
|------|-----------|-------|
| `decomposition.root` | `CompoundTask<JsonNode>` | Root of the HTN tree |
| `methods[]:` | `DecompositionMethod<JsonNode>` | Guard-gated decomposition |
| `guard:` | `Predicate<JsonNode>` via JQ expression | Evaluated against working layer |
| `guardLabel:` | String | Human-readable method label |
| `tasks[]:` with `capability:` | `LeafTask<JsonNode>` (as `GoalStep`) | Terminal task |
| `tasks[]:` with `methods:` | `CompoundTask<JsonNode>` (recursive) | Nested compound |

## Integration with DecompositionStrategy

When `spec.decomposition` is present and `decompositionStrategy` is omitted (or `"identity"`), the converter registers an `ExplicitHtnDecompositionStrategy` (id=`"explicit"`) that returns the declared tree directly — no planning or search.

When `decompositionStrategy` is set to `"goap"`, `"llm"`, or `"portfolio"`, the runtime strategy takes precedence. The explicit tree is ignored.

`CaseDefinition` gains a nullable `decompositionTree` field (`CompoundTask<JsonNode>`). `ExplicitHtnDecompositionStrategy` reads this field and produces the `DagPlan<LeafTask<JsonNode>>` by evaluating method guards against the decomposition context's state.

## Implementation Layers

### Layer 1 — YAML Records

New records in `io.casehub.api.model.converter.yaml`:

```java
record YamlDecomposition(YamlHtnNode root) {}

record YamlHtnNode(
    String name,
    String capability,       // leaf: non-null, compound: null
    String description,
    String estimatedDuration, // ISO-8601
    Map<String, Integer> estimatedCost,
    List<YamlHtnMethod> methods) {}  // leaf: null/empty, compound: non-empty

record YamlHtnMethod(
    String name,
    String guardLabel,
    ExpressionEvaluator guard,   // @JsonDeserialize via module
    String estimatedDuration,
    Map<String, Integer> estimatedCost,
    List<YamlHtnNode> tasks) {}  // recursive — contains leaf or compound nodes
```

`YamlHtnNode` is self-discriminating: `capability` present → leaf, `methods` present → compound. No `@JsonTypeInfo` needed.

### Layer 2 — Converter

`YamlCaseDefinitionConverter` gains `convertDecomposition(YamlDecomposition, ExpressionEngineRegistry)`:
- Walks the tree recursively
- Leaf nodes → `GoalStep` (existing `LeafTask<JsonNode>` implementation)
- Compound nodes → `CompoundTask<JsonNode>` with converted methods
- Guard expressions → `Predicate<JsonNode>` wrapping the expression evaluator
- Sets `CaseDefinition.setDecompositionTree(root)`
- If no `decompositionStrategy` is explicitly set, sets `decompositionStrategy` to `"explicit"`

### Layer 3 — Strategy

`ExplicitHtnDecompositionStrategy` (`planning/decomposition/`, `@ApplicationScoped`, id=`"explicit"`):
- `decompose(task, context)` → evaluates method guards in order, selects first match, recursively decomposes child tasks
- Produces a `DagPlan<LeafTask<JsonNode>>` with causal dependencies (sequential by default within a method's task list)
- Registered via `EngineStrategyResolver`

### Layer 4 — CaseDefinition

`CaseDefinition` gains:
- `decompositionTree` (`CompoundTask<JsonNode>`, nullable) — the explicit HTN root
- `getDecompositionTree()` / `setDecompositionTree()` — getter/setter pair
- Builder: `.decompositionTree(CompoundTask<JsonNode>)`

## Validation

- Every leaf `capability:` must reference a declared `spec.capabilities` entry (warning if not)
- Guard expressions are validated via `ExpressionEngineRegistry.validate()` at definition load time
- Method with no `tasks:` is rejected (empty decomposition)
- Circular nesting is structurally impossible (YAML is a tree, not a graph)

## What Changes

| File | Change |
|------|--------|
| `YamlCaseSpec.java` | Add `YamlDecomposition decomposition` field |
| New: `YamlDecomposition.java` | Record |
| New: `YamlHtnNode.java` | Record (self-discriminating leaf/compound) |
| New: `YamlHtnMethod.java` | Record |
| `YamlCaseDefinitionConverter.java` | Add `convertDecomposition()` method |
| `CaseDefinition.java` | Add `decompositionTree` field + getter/setter |
| `CaseDefinition.Builder` | Add `.decompositionTree()` method |
| New: `ExplicitHtnDecompositionStrategy.java` | Planning module, `@ApplicationScoped` |
| `EngineStrategyResolver.java` | Register `ExplicitHtnDecompositionStrategy` |

## Not in Scope

- Runtime-generated plans (GOAP, LLM) → already implemented
- Compound PlanItemDefinition (`spec.compounds:`) → different concept (runtime grouping, not planning hierarchy)
- Schema updates (`CaseDefinition.yaml`, `.ts`) → separate issue
- DAG wiring (parallel branches within a method's tasks) → future enhancement; v1 is sequential

## References

- `TaskNode.java` — sealed interface (CompoundTask, LeafTask)
- `DecompositionMethod.java` — guard-gated method record
- `DecompositionStrategy.java` — SPI interface
- `GoalStep.java` — existing LeafTask implementation used by LlmDecompositionStrategy
- `GoapDecompositionStrategy.java` — reference for how strategies produce DagPlans
- Issue #987 — proposed YAML shape
- Issue #978 — parent epic
