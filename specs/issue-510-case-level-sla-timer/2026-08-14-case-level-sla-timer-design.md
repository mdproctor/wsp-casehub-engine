# Case-Level SLA — SignalTarget and ScheduleTrigger YAML Parity

**Issue:** casehubio/engine#510
**Date:** 2026-08-14
**Status:** Draft

## Problem

The engine has no case-level timeout mechanism. Milestone-level SLA exists (`MilestoneSLATimeoutJob`, `MilestoneSLAViolatedEvent`) for individual milestone deadlines, but a case that runs indefinitely because agents keep failing has no overall deadline.

Application-tier harnesses need a case-level SLA to trigger a failure goal when the overall case exceeds its time budget — regardless of which tier the failure cascade is in.

The infrastructure for timer-triggered bindings (`ScheduleTrigger`, `SchedulerService`, `ScheduledTriggerJob`) already exists at the Java DSL level. Two gaps block the use case:

1. **Missing target type.** The `BindingTarget` hierarchy only models external actor dispatch (workers, sub-cases, human tasks). There is no target type for "engine writes a signal to its own context." Timer-triggered SLA expiry is an engine-internal state transition, not work dispatched to an external actor.

2. **YAML parity gap.** `CaseDefinitionYamlMapper.convertTrigger()` throws `UnsupportedOperationException` for `ScheduleTrigger` (TODO at line 1116). The Java DSL supports `ScheduleTrigger.delay()` and `ScheduleTrigger.cron()` but YAML definitions cannot use them.

## Design

### SignalTarget — new BindingTarget permit

`SignalTarget` is a new sealed permit on `BindingTarget`, representing engine-internal context mutations.

```java
// engine-api: io.casehub.api.model
public record SignalTarget(Map<String, Object> payload) implements BindingTarget {
    public SignalTarget {
        Objects.requireNonNull(payload, "payload must not be null");
        if (payload.isEmpty()) {
            throw new IllegalArgumentException("SignalTarget payload must not be empty");
        }
        payload = Map.copyOf(payload);
    }
}
```

`BindingTarget` becomes:

```java
public sealed interface BindingTarget
    permits CapabilityTarget, SubCaseTarget, HumanTaskTarget, SignalTarget, ExtensionTarget {}
```

**Java DSL:**

```java
Binding.builder()
    .name("case-timeout")
    .signal(Map.of("caseSla", Map.of("expired", true)))
    .on(ScheduleTrigger.delay(Duration.ofHours(48)))
    .when(".caseSla.expired == null")
    .build()
```

`Binding.Builder` gains `.signal(Map<String, Object>)` convenience method which creates `new SignalTarget(payload)` and sets it as the target.

**Build-time validation** in `Binding.Builder.build()`:
- `SignalTarget` requires `LifecycleScope.BINDING` — opposite constraint to `ScopeActivatedTrigger` (which requires COMPOUND or CASE), same validation site
- `SignalTarget` rejects `Participation.COMPANION`
- `SignalTarget` is compatible with `ContextChangeTrigger` and `ScheduleTrigger`. Not compatible with `ScopeActivatedTrigger` — the existing validation rejects `ScopeActivatedTrigger + BINDING scope`

The existing `instanceof CapabilityTarget` guard at line 368 (`ls != LifecycleScope.BINDING && !(target instanceof CapabilityTarget)`) already rejects non-BINDING scopes for non-CapabilityTarget types. A new explicit guard for `SignalTarget` is clearer but functionally redundant.

### YAML syntax

**Trigger — ScheduleTrigger parity:**

```yaml
on:
  schedule:
    delay: PT48H        # ISO-8601 Duration — one-shot
    # OR
    cron: "0 0 * * *"   # Quartz cron expression — periodic
```

Exactly one of `delay` or `cron` must be set. Maps to `ScheduleTrigger.delay(Duration)` or `ScheduleTrigger.cron(String)`.

**Target — signal block:**

```yaml
signal:
  caseSla:
    expired: true
```

Mutually exclusive with `capability:`, `subCase:`, `humanTask:`. The nested map is the payload written to context when the binding fires.

### Dispatch path — context-change-triggered signals

When a `ContextChangeTrigger` binding with `SignalTarget` fires, `CaseContextChangedEventHandler.publishByTarget()` handles it in a new switch branch:

```java
case SignalTarget st -> applySignal(caseInstance, binding, st);
```

`applySignal()`:
1. Writes each entry in `st.payload()` to the working layer via `context.set(key, value)`
2. Writes an EventLog entry: type `CONTEXT_SIGNAL_APPLIED`, metadata `{bindingName, signalKeys, triggerType: "contextChange"}`
3. Publishes `CONTEXT_CHANGED` so downstream bindings react

The `when` guard prevents infinite re-evaluation (e.g., `.caseSla.expired == null` becomes false after the first write).

No PlanItem is created. Signals are instantaneous engine-internal actions, not tracked work. The audit trail comes from the EventLog entry.

### Dispatch path — schedule-triggered signals

**SchedulerService.registerScheduledTriggers()** gains a `SignalTarget` branch in the target switch:

```java
case SignalTarget st -> {
    scheduleSignal(caseInstance.getUuid(), binding, trigger, st);
}
```

`scheduleSignal()` creates a Quartz job with:
- `triggerType=signal`
- `caseId`, `bindingName`
- `signalPayload` — serialized JSON of the signal map
- `hasCondition` — whether the binding has a `when` guard
- `conditionExpression` — the JQ expression string from `binding.getWhen()` (only when `hasCondition` is true)

No worker or capability lookup needed.

**Note:** The current `registerScheduledTriggers()` uses a switch expression that yields `CapabilityTarget`. The `SignalTarget` branch requires restructuring to a switch statement — `SignalTarget` does not yield a `CapabilityTarget`, and the subsequent worker lookup is not needed.

**QuartzJobScheduler.resolveJobClass()** gains:

```java
} else if ("signal".equals(triggerType)) {
    jobClass = ScheduledSignalJob.class;
}
```

**ScheduledSignalJob** — new Quartz job in `scheduler-quartz`:
1. Loads case instance via `WorkerExecutionRecoveryService.loadOrRestoreCaseInstance()`
2. Checks case is still RUNNING (skips if terminal)
3. If `hasCondition` is true, evaluates the `when` JQ expression against the case context. Skips if condition is false.
4. Publishes `ContextSignalEvent` to the event bus

**ContextSignalEvent** — new record in `engine-common`:

```java
// engine-common: io.casehub.engine.common.internal.event
public record ContextSignalEvent(
    CaseInstance caseInstance,
    String bindingName,
    Map<String, Object> payload
) {}
```

**ContextSignalEventHandler** — new handler in `runtime`:

```java
@ApplicationScoped
public class ContextSignalEventHandler {

    @ConsumeEvent(EventBusAddresses.CONTEXT_SIGNAL)
    @RunOnVirtualThread
    public void onContextSignal(ContextSignalEvent event) {
        // 1. Apply payload to working layer
        // 2. Write CONTEXT_SIGNAL_APPLIED EventLog
        // 3. Publish CONTEXT_CHANGED
    }
}
```

This follows the virtual-thread-handler-convention protocol (PP-20260723-c4c1cf).

### Cancellation

`CaseStatusChangedHandler` already calls `SchedulerService.cancelAllTriggers(caseId)` on terminal state. Signal jobs use the same group naming convention (`case-{caseId}`) and are cancelled automatically. No additional work needed.

### New event type and event bus address

```java
// CaseHubEventType
CONTEXT_SIGNAL_APPLIED

// EventBusAddresses
public static final String CONTEXT_SIGNAL = "casehub.context.signal";
```

EventLog metadata for `CONTEXT_SIGNAL_APPLIED`:
- `bindingName` — which binding fired
- `signalKeys` — list of top-level keys written (e.g., `["caseSla"]`)
- `triggerType` — `"contextChange"` or `"schedule"`

### Exhaustive switch updates

Every exhaustive `switch` on `BindingTarget` needs a `case SignalTarget` branch. Known sites:

| File | Method | Signal handling |
|------|--------|----------------|
| `CaseContextChangedEventHandler` | `publishByTarget()` | Apply signal (main dispatch) |
| `SchedulerService` | `registerScheduledTriggers()` | Schedule signal job |
| `SchedulerService` | `createJobData()` | Not called for signals (separate path) |
| `CbrCaseRetainObserver` | `buildRoutingKeyMap()` | Skip — no capability trace |

`Binding.Builder.build()` uses `instanceof` guards, not an exhaustive switch. The existing guard at line 368 already rejects non-BINDING scopes for non-CapabilityTarget types — no new branch needed, though an explicit `SignalTarget` guard improves readability.

`instanceof CapabilityTarget` guards across the codebase are naturally correct — signal bindings don't have capabilities, so they're excluded from worker routing, agent matching, and capability-based lookups.

### PlanItem lifecycle

Signal bindings do not create PlanItems. Signals are atomic engine actions, not work-in-progress. The audit trail comes from the `CONTEXT_SIGNAL_APPLIED` EventLog entry.

Compound completion: signal bindings are excluded from completion counting. This is enforced by the `LifecycleScope.BINDING` constraint — signal bindings cannot be scoped to compounds.

### v1 limitations

- **Static payload only.** `SignalTarget` payload is `Map<String, Object>` — literal values frozen at definition time. Cannot express derived values (timestamps, elapsed time, context-dependent flags). Dynamic payloads via JQ expression evaluation are a backward-compatible v2 addition when a concrete use case demands it.
- **No retry.** Signal application is a synchronous in-memory operation. If it fails (which requires a bug — it's a map write), the EventLog entry is not written and the signal is lost. Acceptable for v1 given the operation is trivial.

## Complete YAML example

```yaml
spec:
  capabilities:
    - name: review-code
      inputSchema: "."

  goals:
    - name: review-complete
      when: ".reviewResult != null"
    - name: review-timed-out
      when: ".caseSla.expired == true"

  completion:
    success:
      allOf: [review-complete]
    failure:
      anyOf: [review-timed-out]

  workers:
    - name: code-reviewer
      capabilities: [review-code]
      agent:
        model: { provider: anthropic, name: claude-sonnet-4-20250514 }

  bindings:
    - name: do-review
      on:
        contextChange:
          filter: ".reviewRequest != null"
      capability: review-code

    - name: case-timeout
      on:
        schedule:
          delay: PT48H
      when: ".caseSla.expired == null"
      signal:
        caseSla:
          expired: true
```

## References

- Garden: GE-20260802-7ad695 (Quartz jobs and in-memory CaseContext after restart)
- Garden: GE-20260511-3e5a75 (casehub-work SLA breach patterns)
- Protocol: PP-20260723-c4c1cf (virtual-thread-handler-convention)
- Protocol: PP-20260727-5267d2 (plan-type-module-boundary)
- Blocks: casehubio/devtown#14 (review-timed-out failure goal)
