# Human Task Binding — Design Spec
*2026-05-12*

## Context

`casehub-engine-work-adapter` already implements the **inbound** path: `WorkItemLifecycleAdapter`
observes `WorkItemLifecycleEvent`, parses `callerRef` (`case:{caseId}/pi:{planItemId}`), marks
the `PlanItem`, and fires `CONTEXT_CHANGED` (8 tests passing). Refs casehubio/work#136.

The **outbound** path is missing: nothing in the engine creates a casehub-work `WorkItem` when a
human-task binding activates. This spec covers the full design for both directions, including
`outputMapping` round-trip.

Tracked in casehubio/engine#245 (Human Task Binding). Closes casehubio/work#136.

---

## Design

### 1. `BindingTarget` sealed interface — `casehub-engine-api`

Replace `Binding`'s nullable `capability` + `subCase` fields with a single `BindingTarget target`:

```java
public sealed interface BindingTarget
    permits CapabilityTarget, SubCaseTarget, HumanTaskTarget, ExtensionTarget {}

public record CapabilityTarget(Capability capability) implements BindingTarget {}
public record SubCaseTarget(SubCase subCase)         implements BindingTarget {}
public non-sealed interface ExtensionTarget extends BindingTarget {}  // runtime plugin escape hatch
```

`Binding.target()` replaces `getCapability()` / `getSubCase()`. Builder retains typed convenience
methods. The `exactly-one-of` constraint is now structural. `ExtensionTarget` is a non-sealed
escape hatch for future runtime plugins — no dispatcher implemented until a real consumer exists.

`CaseContextChangedEventHandler` switches exhaustively:

```java
switch (binding.target()) {
    case CapabilityTarget t  -> publishWorkerSchedule(caseInstance, t.capability(), workers)
    case SubCaseTarget t     -> publishSubCaseSchedule(caseInstance, t.subCase())
    case HumanTaskTarget t   -> publishHumanTaskSchedule(caseInstance, binding.getName(), t)
    case ExtensionTarget t   -> LOG.warnf("No handler for ExtensionTarget type %s — ignored",
                                    t.getClass().getName())
}
```

---

### 2. `HumanTaskTarget` — `casehub-engine-api`

Pure-Java record (no Quarkus, no casehub-work types). Two factory entry points — template ref for
reuse, inline for one-off. Both support optional `inputMapping` and `outputMapping` via pluggable
`ExpressionEvaluator` (String shorthand = JQ by default, same as `Binding.when(String)`).

```java
// Template — operational config in casehub-work; engine just names it
HumanTaskTarget.template("irb-72h-review")
    .inputMapping("{ description: (\"Trial \" + .trialId + \" Phase \" + .phase) }")
    .outputMapping("{ irbOutcome: .decision, conditions: .conditions }")
    .build()

// Inline — self-contained, one-off
HumanTaskTarget.inline()
    .title("IRB Ethics Committee Review")
    .candidateGroups(Set.of("ethics-committee"))
    .expiresIn(Duration.ofHours(72))
    .build()

// Lambda / custom evaluator overload
HumanTaskTarget.template("irb-72h-review")
    .outputMapping(ctx -> Map.of("irbOutcome", ctx.get("decision")))
    .build()
```

**Fields:**

| Field | Type | Template | Inline |
|---|---|---|---|
| `templateId` | `String` | required | null |
| `title` | `String` | optional override | required |
| `description` | `String` | optional override | optional |
| `candidateGroups` | `Set<String>` | optional override | recommended |
| `candidateUsers` | `Set<String>` | optional override | optional |
| `priority` | `String` | optional override | optional |
| `expiresIn` | `Duration` | optional override | optional |
| `inputMapping` | `ExpressionEvaluator` | optional | optional |
| `outputMapping` | `ExpressionEvaluator` | optional | optional |

`priority` is a plain `String` ("LOW"/"MEDIUM"/"HIGH"/"URGENT") — the work-adapter maps to
`WorkItemPriority`. No casehub-work types in the API module.

**Startup validation (template mode):** on `@Startup`, `HumanTaskScheduleHandler` iterates all
registered `CasePlanModel`s and warns for any `templateId` that doesn't resolve via
`WorkItemTemplateService.findById()`. Fail-fast option configurable.

---

### 3. `PlanItem` stores `BindingTarget` — `casehub-engine-blackboard`

`PlanItem` gains a `BindingTarget target()` field:

```java
public static PlanItem create(String planItemId, String workerName, int priority, BindingTarget target)
```

Wherever the planning loop creates plan items (already has `Binding` objects), pass
`binding.target()`. This makes `PlanItem` fully self-describing and enables `outputMapping` lookup
at completion time for **all** binding types (HumanTask, SubCase — not just HumanTask).

A backward-compatible 3-arg `create(planItemId, workerName, priority)` overload sets `target = null`
for existing tests that don't yet need the target.

---

### 4. Outbound flow — engine runtime (`casehub-engine`)

`EventBusAddresses.HUMAN_TASK_SCHEDULE` added.

`HumanTaskScheduleEvent` (internal runtime record):
```
UUID caseId
String planItemId
HumanTaskTarget target
Map<String, Object> inputData   // pre-evaluated from inputMapping against CaseContext
```

`publishHumanTaskSchedule()` in `CaseContextChangedEventHandler`:
```java
private void publishHumanTaskSchedule(CaseInstance instance, String planItemId, HumanTaskTarget t) {
    Map<String, Object> inputData = t.inputMapping() != null
        ? instance.getCaseContext().evalObjectTemplate(t.inputMapping())
        : Map.of();
    eventBus.publish(HUMAN_TASK_SCHEDULE,
        new HumanTaskScheduleEvent(instance.getUuid(), planItemId, t, inputData));
}
```

`inputMapping` is evaluated in the engine (where CaseContext lives) before publishing. The
work-adapter handler receives pre-evaluated data and doesn't need CaseContext access.

---

### 5. `HumanTaskScheduleHandler` — `casehub-engine-work-adapter`

```java
@ApplicationScoped
public class HumanTaskScheduleHandler {

    @Inject BlackboardRegistry blackboardRegistry;
    @Inject WorkItemTemplateService workItemTemplateService;
    @Inject WorkItemService workItemService;

    @ConsumeEvent(EventBusAddresses.HUMAN_TASK_SCHEDULE)
    public void onHumanTaskSchedule(HumanTaskScheduleEvent event) {
        CasePlanModel plan = blackboardRegistry.get(event.caseId()).orElse(null);
        if (plan == null) { LOG.warnf("No CasePlanModel for %s", event.caseId()); return; }

        PlanItem planItem = plan.getPlanItem(event.planItemId()).orElse(null);
        if (planItem == null) { LOG.warnf("PlanItem %s not found", event.planItemId()); return; }

        // Mark RUNNING before creating WorkItem — prevents binding re-evaluation
        planItem.markRunning();

        String callerRef = CallerRef.encode(event.caseId(), event.planItemId());
        HumanTaskTarget t = event.target();

        if (t.templateId() != null) {
            WorkItemTemplate template = workItemTemplateService.findById(UUID.fromString(t.templateId()))
                .orElseThrow(() -> new IllegalArgumentException("Template not found: " + t.templateId()));
            workItemTemplateService.instantiate(template, t.title(), null, "system:engine", callerRef);
        } else {
            workItemService.create(new WorkItemCreateRequest(
                t.title(), t.description(), null, null,
                mapPriority(t.priority()), null,
                t.candidateGroups(), t.candidateUsers(), null,
                "system:engine", null, null, null, null, null, null,
                callerRef, null, t.expiresIn() != null ? Duration.toExpiresAt(t.expiresIn()) : null));
        }
    }
}
```

---

### 6. `WorkItemLifecycleAdapter` extension — `casehub-engine-work-adapter`

After marking the `PlanItem` and before firing `CONTEXT_CHANGED`, evaluate `outputMapping`:

```java
// Look up HumanTaskTarget via PlanItem (already fetched above)
if (item.target() instanceof HumanTaskTarget humanTask
        && humanTask.outputMapping() != null
        && workItem.resolution != null) {
    try {
        Map<String, Object> output = humanTask.outputMapping().evaluate(workItem.resolution);
        instance.getCaseContext().putAll(output);
    } catch (Exception e) {
        LOG.warnf(e, "outputMapping evaluation failed for PlanItem %s — CONTEXT_CHANGED fires without output",
            item.getPlanItemId());
    }
}
// fire CONTEXT_CHANGED as before
```

No separate registry. The `PlanItem.target()` is the source of truth — clean, restart-safe,
no parallel state.

---

### 7. callerRef support in template instantiation — `casehub-work`

`WorkItemTemplateService.instantiate()` currently hardcodes `callerRef = null`. A new overload
accepts `callerRef` and passes it through `toCreateRequest()`:

```java
@Transactional
public WorkItem instantiate(WorkItemTemplate template, String titleOverride,
        String assigneeIdOverride, String createdBy, String callerRef)
```

`toCreateRequest()` gains `callerRef` parameter. Existing 4-arg overload delegates to the new one
with `callerRef = null` — no existing callers break. Tracked in casehubio/work#<TBD>.

---

## Data flow — full HITL round-trip

```
1. CaseContext condition met → CaseContextChangedEventHandler evaluates binding
2. binding.target() = HumanTaskTarget → publishHumanTaskSchedule()
3. inputMapping evaluated against CaseContext → inputData
4. HumanTaskScheduleEvent published on Vert.x event bus
5. HumanTaskScheduleHandler.onHumanTaskSchedule():
     a. PlanItem.markRunning()
     b. callerRef = CallerRef.encode(caseId, planItemId)
     c. WorkItemService/TemplateService.create/instantiate(... callerRef ...)
6. WorkItem created (PENDING) — appears in human task inbox
7. Human acts → WorkItem transitions to COMPLETED/REJECTED/EXPIRED/etc.
8. WorkItemLifecycleEvent fired (carries callerRef + resolution)
9. WorkItemLifecycleAdapter.onWorkItemLifecycle():
     a. CallerRef.parse() → caseId + planItemId
     b. PlanItem.markCompleted() / markFaulted() / markCancelled()
     c. outputMapping evaluated against workItem.resolution → CaseContext updated
     d. CaseContextChangedEvent published → engine re-evaluates
10. Case proceeds to next stage
```

---

## Error handling

| Scenario | Behaviour |
|---|---|
| `CasePlanModel` not found at schedule time | Warn + return. Case may have completed. |
| `PlanItem` not found | Warn + return. Configuration error — binding name mismatch. |
| Template ID not found | `IllegalArgumentException` propagated. Binding stays eligible for next tick. |
| `WorkItemService.create()` throws | Log + propagate. Same pattern as `ProvisioningException`. |
| `outputMapping` eval fails | Log warn. `CONTEXT_CHANGED` fires without output write. Case not faulted. |
| Completion arrives after case completed | `registry.get(caseId)` returns empty → warn + return. |

---

## Module touch points

| Module | Changes |
|---|---|
| `casehub-engine-api` | `BindingTarget` (sealed, 4 permits); `CapabilityTarget`, `SubCaseTarget`, `HumanTaskTarget`, `ExtensionTarget`; `Binding` refactored |
| `casehub-engine-blackboard` | `PlanItem` gains `BindingTarget target`; `PlanItem.create()` overloads; planning loop passes `binding.target()` |
| `casehub-engine` runtime | `EventBusAddresses.HUMAN_TASK_SCHEDULE`; `HumanTaskScheduleEvent`; `CaseContextChangedEventHandler` pattern match + new method |
| `casehub-engine-work-adapter` | `HumanTaskScheduleHandler` (new); `WorkItemLifecycleAdapter` (outputMapping); new tests |
| `casehub-work` runtime | `WorkItemTemplateService.instantiate()` new overload with `callerRef`; `toCreateRequest()` updated |

---

## Testing

**`casehub-engine-api`** (unit):
- `HumanTaskTargetTest`: builder validation, template vs inline, both ExpressionEvaluator overloads
- `BindingTest`: updated for sealed target

**`casehub-engine-blackboard`** (unit):
- `PlanItemTest`: `target` field round-trip for each target type

**`casehub-engine` runtime** (`@QuarkusTest`):
- `CaseContextChangedEventHandlerTest`: `HumanTaskTarget` branch publishes `HUMAN_TASK_SCHEDULE`
  with correct caseId, planItemId, pre-evaluated inputData

**`casehub-engine-work-adapter`** (`@QuarkusTest`):
- `HumanTaskScheduleHandlerTest`: template mode → `instantiate()` called with correct templateId +
  callerRef; inline mode → `create()` with correct fields; PlanItem marked RUNNING before create;
  CasePlanModel not found → no-op
- `WorkItemLifecycleAdapterTest` extended: outputMapping present → CaseContext updated; eval
  throws → CONTEXT_CHANGED still fires
- `WorkItemRoundTripTest`: full end-to-end — binding activates → WorkItem created with callerRef +
  PlanItem RUNNING → WorkItem COMPLETED with resolution → PlanItem COMPLETED + CaseContext updated
  → CONTEXT_CHANGED fires

**`casehub-work` runtime** (`@QuarkusTest`):
- `WorkItemTemplateServiceTest`: new overload sets callerRef correctly; existing tests unchanged

---

## Issues

- casehubio/engine#245 — implement this spec (refs #115 Human Escalation epic; closes casehubio/work#136)
- casehubio/work#165 — `WorkItemTemplateService.instantiate()` callerRef overload (prerequisite)
