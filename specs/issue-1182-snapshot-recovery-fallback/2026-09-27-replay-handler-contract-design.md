# Enforce Replay Handler Registration — Design Spec

**Issue:** casehubio/engine#1184
**Epic:** casehubio/engine#210 — cancellation, timeout, and error recovery
**Date:** 2026-09-27

## Problem

`EventLogReplayRecoveryStrategy.rebuildStateContext()` replays events by type — each context-mutating event type needs an explicit handler. Nothing enforces this. Issues #1151, #1152, and #1154 were caused by missing replay handlers that went undetected until production cache misses.

## Solution

### 1. Extract REPLAYED_TYPES constant

Move the inline `EnumSet` (lines 74-84) to a package-visible `static final` field:

```java
static final EnumSet<CaseHubEventType> REPLAYED_TYPES = EnumSet.of(
    CaseHubEventType.CASE_STARTED,
    CaseHubEventType.WORKER_EXECUTION_COMPLETED,
    CaseHubEventType.SUBCASE_COMPLETED,
    CaseHubEventType.SIGNAL_RECEIVED,
    CaseHubEventType.SCOPED_WORKER_OUTPUT,
    CaseHubEventType.CONTEXT_SIGNAL_APPLIED,
    CaseHubEventType.MILESTONE_ACTIVATED,
    CaseHubEventType.MILESTONE_COMPLETED,
    CaseHubEventType.MILESTONE_SLA_VIOLATED,
    CaseHubEventType.GOAL_REACHED);
```

The `rebuildStateContext()` method uses this field instead of the inline set.

### 2. Contract test

A new test method in `EventLogReplayRecoveryStrategyTest`:

```java
@Test
void allContextMutatingEventTypes_haveReplayHandlers() {
    EnumSet<CaseHubEventType> allTypes = EnumSet.allOf(CaseHubEventType.class);
    EnumSet<CaseHubEventType> accounted = EnumSet.copyOf(REPLAYED_TYPES);
    accounted.addAll(NON_MUTATING_TYPES);

    EnumSet<CaseHubEventType> unaccounted = EnumSet.copyOf(allTypes);
    unaccounted.removeAll(accounted);

    assertThat(unaccounted)
        .as("New CaseHubEventType values must be added to either "
            + "EventLogReplayRecoveryStrategy.REPLAYED_TYPES (if context-mutating) "
            + "or NON_MUTATING_TYPES in this test (if not). "
            + "Unaccounted types: %s", unaccounted)
        .isEmpty();
}
```

`NON_MUTATING_TYPES` is an `EnumSet` defined in the test containing every `CaseHubEventType` value that does NOT mutate `CaseContext`. This is the exhaustive allowlist — approximately 90+ types covering status changes, lifecycle events, orchestration events, stigmergy events, etc.

When a developer adds a new `CaseHubEventType` value, the test fails immediately. They must either add a handler in `rebuildStateContext()` and include the type in `REPLAYED_TYPES`, or add it to `NON_MUTATING_TYPES` with the assertion that it doesn't mutate context.

## Files Changed

| File | Change |
|------|--------|
| `runtime/.../recovery/EventLogReplayRecoveryStrategy.java` | Extract `REPLAYED_TYPES` constant. Use it in `rebuildStateContext()`. |
| `runtime/src/test/.../EventLogReplayRecoveryStrategyTest.java` | Add contract test with `NON_MUTATING_TYPES` allowlist. |

## References

- `EventLogReplayRecoveryStrategy.java:74-84` — current inline EnumSet
- `CaseHubEventType.java` — full enum (~100+ values)
- Issues #1151, #1152, #1154 — prior gaps from missing handlers
- decisions.md D8
