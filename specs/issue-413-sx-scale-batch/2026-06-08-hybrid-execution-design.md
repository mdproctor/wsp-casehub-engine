# True Hybrid Execution — Design

**Issue:** casehubio/engine#200  
**Date:** 2026-06-08  
**Status:** Design — no implementation

---

## Status of the Three Use Cases

### Use case 1 — Workflow-driven orchestration (FlowWorker dispatching casehub workers)

**Closed.** `casehub-engine-flow` ships `CasehubDispatch` + `CasehubCallableTaskBuilder`.
A FlowWorker step with `call: casehub:dispatch` + `capability: <name>` dispatches a casehub
worker from within a quarkus-flow step and awaits its result. Full ledger lineage via
`WORKFLOW_STEP_DISPATCHED/COMPLETED/FAILED` events. Landed in `issue-206-flowworker-bridge`.

### Use case 2 — Rules-driven orchestration (Drools/DMN produces capability sequence)

**Deferred to Drools epic #445.** Depends on `ExpressionEvaluatorFactory` SPI (#289) and
`DroolsExpressionEvaluator` (#5). The rules engine producing an ordered capability list is
a natural output of the `WorkOrchestrator.submitAndWait()` path once Drools is wired.
Design belongs in the Drools epic, not here.

### Use case 3 — Plan-based execution

**Open.** A planner (LLM, human, rules engine) produces an explicit ordered plan and the
engine executes it step by step with full lineage. This is the remaining genuine gap in the
hybrid execution model.

---

## Plan-Based Execution — Design

### The gap

There is no first-class way to declare "execute these capabilities in this order" in the
CaseDefinition DSL. A case author wanting sequential execution today must:
- Write a Java `Worker(submitAndWait...)` chain manually, or
- Use a FlowWorker YAML with an explicit state per step.

Both are verbose for simple A→B→C sequential plans and impossible to make dynamic (plan
determined at runtime by LLM output).

### What plan execution needs

1. **Sequential dispatch** — each step starts only after the previous completes.
2. **Output threading** — step N's output is available to step N+1 as input context.
3. **Full lineage** — each step produces a `CaseLedgerEntry`; `causedByEntryId` chains correctly.
4. **Dynamic plans** — the plan may be determined at runtime (LLM output, not CaseDefinition).

### Design decision: binding target type vs. Worker function type

Two candidate approaches:

**A) `plan` binding target type** (alongside `capability`, `humanTask`, `subCase` in YAML DSL)

```yaml
bindings:
  - name: research-phase
    trigger: contextChange(".startResearch == true")
    plan:
      steps:
        - capability: researcher
          inputMapping: "{ input: .caseContext.brief }"
        - capability: security-auditor
          inputMapping: "{ findings: .planResults.researcher }"
        - capability: coder
```

Static plans declared in the CaseDefinition. Steps are executed by a new
`PlanExecutionHandler` that calls `WorkOrchestrator.submitAndWait()` for each step.
Each step result is accumulated into `planResults.{stepName}` in case context.

**B) `Worker(Plan)` function type** (Java fluent DSL)

```java
Worker(Plan.of(
    WorkRequest.of("researcher", Map.of("brief", "$.caseContext.brief")),
    WorkRequest.of("security-auditor", Map.of()),
    WorkRequest.of("coder", Map.of())
))
```

Static plans in the fluent DSL. The Plan is a `WorkerFunction` that drives
`WorkOrchestrator.submitAndWait()` in a loop and returns the accumulated output.

**C) FlowWorker YAML foreach** (dynamic plans, no new machinery)

```yaml
states:
  - name: execute-plan
    type: foreach
    inputCollection: ${ .executionPlan }
    iterationParam: step
    actions:
      - functionRef: casehub:dispatch
        arguments: { capability: ${ .step } }
```

Uses existing `casehub-engine-flow` + `CasehubDispatch`. LLM writes `executionPlan: [...]`
to case context; the FlowWorker iterates it. No new machinery.

### Recommendation

**Adopt all three as a composable set, not alternatives.**

| Scenario | Mechanism |
|----------|-----------|
| Static plan known at definition time, YAML DSL | `plan` binding target type (Option A) |
| Static plan known at definition time, Java DSL | `Worker(Plan.of(...))` (Option B) |
| Dynamic plan from LLM/rules at runtime | FlowWorker + `casehub:dispatch` foreach (Option C) |

Option C already works. Options A and B are implementation work.

**Priority:** Implement Option B first (Worker function type in the fluent DSL) — it requires
no new YAML machinery, no new binding infrastructure, and no new event types. It maps cleanly
onto the existing Worker function model. Option A is then the YAML equivalent, sharing the
same execution engine.

---

## Option B — Worker(Plan) specification

### Interface

```java
// In api/src/main/java/io/casehub/api/model/
public record PlanStep(String capability, Map<String, Object> input) {
    public static PlanStep of(String capability) { return new PlanStep(capability, Map.of()); }
    public static PlanStep of(String capability, Map<String, Object> input) { return new PlanStep(capability, input); }
}
```

```java
// WorkerFunction variant — signature matches existing Worker contract
public final class PlanWorkerFunction implements WorkerFunction {
    private final List<PlanStep> steps;
    // Constructed by Worker(Plan.of(...))
}
```

```java
// Fluent DSL — mirrors Worker(Workflow), Worker(Agent)
Worker(Plan.of(
    PlanStep.of("researcher"),
    PlanStep.of("security-auditor"),
    PlanStep.of("coder")
))
```

### Execution contract

1. The plan is executed by a new `PlanWorkerExecutor` (mirrors `QuartzWorkerExecutionJob`
   for capability workers, `FlowWorkerExecutor` for Workflow workers).
2. Each step: `WorkOrchestrator.submitAndWait(instance, WorkRequest.of(step.capability(), step.input()))`.
3. Step output merged into result map: `{ "step0": output0, "step1": output1, ... }` keyed
   by step index. Named steps (optional) use `capability` as key.
4. On step failure: plan execution stops; the Worker is marked FAULTED with the failed
   step's error. Retry semantics are per-step (Quartz retry config applies).

### Lineage

Each `submitAndWait()` call creates a ledger entry. The plan's own ledger entry
(`WORKER_EXECUTION_STARTED` / `WORKER_EXECUTION_FINISHED`) wraps the step entries.
`causedByEntryId` chains: plan entry → step 0 entry → step 1 entry (step 1 is caused by
step 0's completion). Full audit trail.

This matches how `CasehubDispatch` emits `WORKFLOW_STEP_DISPATCHED/COMPLETED` for each
flow step — the same audit pattern applies to plan steps.

### Input threading between steps

```java
Worker(Plan.of(
    PlanStep.of("researcher"),                            // gets case context as input
    PlanStep.of("security-auditor",                      // gets researcher output + context
        Map.of("priorFindings", "$.planResults.step0")), // JQ from accumulated results
    PlanStep.of("coder")
))
```

Input map values prefixed with `$.` are resolved as JQ expressions against the accumulated
`planResults` map at execution time. This is optional — `Map.of()` sends no extra input.

### Dynamic plan — context-driven variant

```java
Worker(Plan.fromContext(".executionPlan"))
// where executionPlan: ["researcher", "security-auditor", "coder"]
```

Reads the capability list from a JQ path against case context at execution time. Allows LLM
output to determine the plan without re-deploying the CaseDefinition.

---

## Option A — `plan` binding target type (YAML DSL)

Deferred to a follow-on issue. Requires:
- `CaseDefinitionYamlMapper` to parse `plan:` binding blocks.
- `PlanTarget` POJO (like `HumanTaskTarget` for `humanTask:`).
- `PlanScheduleEvent` + `PlanExecutionHandler` on the event bus.
- Same `PlanWorkerExecutor` as Option B drives execution.

The execution engine is shared — Option A is a YAML front-end for the same machinery.

---

## Follow-on issues to file

- `feat: Worker(Plan.of(...)) — plan worker function type` (implements Option B above)
- `feat: plan binding target type in CaseDefinition YAML` (implements Option A, depends on B)
- `feat: rules-driven plan generation — Drools WorkOrchestrator output` (Drools epic #445)

---

## Hybrid composition

All four execution models can be mixed within a single case:

```yaml
bindings:
  - name: initial-scan     # choreography — fires on context change
    trigger: contextChange(".received")
    capability: scanner

  - name: research-phase   # plan — sequential A→B→C
    trigger: contextChange(".scanComplete")
    plan:
      steps:
        - capability: researcher
        - capability: security-auditor

  - name: review           # humanTask — waits for human
    trigger: contextChange(".researchComplete")
    humanTask:
      title: "Review research findings"

  - name: execute          # workflow — durable multi-step with branching
    trigger: contextChange(".reviewApproved")
    worker:
      workflow: execute-plan.yaml
```

No constraints. The binding evaluator routes each trigger to its handler regardless of
target type.
