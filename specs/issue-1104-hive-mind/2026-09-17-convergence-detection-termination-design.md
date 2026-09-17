# Convergence Detection & Termination — Design Spec

**Issue:** casehubio/engine#1110
**Epic:** casehubio/engine#1104 (Hive Mind)
**Date:** 2026-09-17
**Decisions:** D45–D56 (with supplementary D57–D59)

## Problem

Multi-agent swarm coordination needs two safety nets that the engine currently lacks:

1. **Convergence detection** — detecting when the swarm has reached a useful outcome and should stop. Without this, cases with emergent coordination (no explicit goal satisfaction) run indefinitely.
2. **Pathology prevention** — detecting and stopping runaway resource consumption, coordination storms, and output convergence that may indicate groupthink.

The existing Watchdog (qhorus) handles communication-level pathology (stalls, loops, echo chambers). This design handles coordination-level monitoring — they are complementary, not overlapping.

## Architecture Overview

Five components, all engine-internal:

```
┌────────────────────────────────────────────────────────────┐
│              CaseContextChangedEventHandler                │
│                                                            │
│  rules() → goals() → observations() → localRules()        │
│                         ↓                                  │
│                convergenceDetection()  ← NEW (5th phase)   │
│                    │         │                              │
│               ┌────┘         └────┐                        │
│               ▼                   ▼                        │
│    ConvergenceDetector    BudgetEnforcer                   │
│    (activity quiescence)  (hard caps)                      │
│               │                   │                        │
│               ▼                   ▼                        │
│    GoalReachedEvent       CaseStatusChanged                │
│    ("_converged")         (FAULTED)                        │
└────────────────────────────────────────────────────────────┘

┌──────────────────────┐    ┌──────────────────────────────┐
│   ActivityTracker    │    │  OutputConvergenceMonitor     │
│  (common-core)       │    │  (runtime-core)              │
│                      │    │                              │
│  Per-case metrics:   │    │  Per-binding output window:  │
│  • totalDispatches   │    │  • key-set Jaccard           │
│  • totalSignalDep.   │    │  • value hash comparison     │
│  • totalCtxMutations │    │  • OUTPUT_CONVERGENCE_       │
│  • totalEvalCycles   │    │    DETECTED event            │
│  + sliding window    │    │                              │
│    rates for each    │    │  Injected into               │
│                      │    │  WorkflowExecution-          │
│  Instrumented at:    │    │  CompletedHandler via        │
│  • evaluateAndDisp() │    │  Instance<> guard            │
│  • publishWorkerSch()│    │                              │
│  • SignalRegistry    │    │                              │
│    .deposit()        │    │                              │
└──────────────────────┘    └──────────────────────────────┘
```

## 1. ActivityTracker

`ActivityTracker` (`common-core`, `@ApplicationScoped`, `Resettable`) — per-case cumulative metrics and sliding-window activity rates.

### Storage

`ConcurrentHashMap<UUID, CaseActivityState>` where `CaseActivityState` holds:
- Four `AtomicLong` cumulative counters (dispatches, signal deposits, context mutations, evaluation cycles)
- Four `SlidingWindowCounter` instances for wall-clock rate computation

### SlidingWindowCounter

Bounded circular buffer of `Instant` timestamps. API: `record(Instant now)`, `rate(Duration window, Instant now) → double` (events per second within the window).

Memory cap: derived from `rateWindow` as `rateWindow.toSeconds() * 10` (supports up to 10 events/second before oldest entries are evicted). Default for 60s window = 600 entries. Configurable override via `ConvergenceConfig.maxWindowEntries`.

At extreme event rates (>10/s), oldest entries are evicted and `rate()` underestimates — acceptable since high rates are by definition not converged.

### Instrumentation Points

Each metric is recorded at its single canonical source:

| Metric | Instrumented at | Handler |
|--------|----------------|---------|
| `totalDispatches` | After successful `WorkerScheduleEvent` publish | `CaseContextChangedEventHandler.publishWorkerSchedule()` |
| `totalSignalDeposits` | Inside `SignalRegistry.deposit()` | Single source of truth for all deposits (worker, rule, any future path) |
| `totalContextMutations` | At evaluation cycle start | `CaseContextChangedEventHandler.evaluateAndDispatch()` — counts `event.changedKeys().size()` |
| `totalEvaluationCycles` | At evaluation cycle start | `CaseContextChangedEventHandler.evaluateAndDispatch()` |

`ActivityTracker` is injected into `CaseContextChangedEventHandler` and `SignalRegistry`.

### Lifecycle

`CaseStatusChangedHandler` calls `activityTracker.evictByCase(caseId)` on terminal case status. `ActivityTracker implements Resettable`.

## 2. Budget Enforcement

Hard gate checked at two points in the evaluation pipeline:

1. **Evaluation cycle budget** — checked at the top of `evaluateAndDispatch()`. If `totalEvaluationCycles > maxEvaluationCycles`, skip evaluation entirely.
2. **Per-operation budgets** — dispatch and signal deposit budgets checked at their respective operation sites. Context mutation budget checked at cycle start.

When any cumulative count exceeds its configured budget cap:
1. Write `BUDGET_EXHAUSTED` to EventLog (metadata: `exhaustedMetric`, `currentCount`, `budgetCap`)
2. Dispatch `CaseStatusChanged(FAULTED)` with reason "Budget exhausted: \<metric\>"

Budget caps are nullable on `ConvergenceConfig` — null means no limit (backward compatible).

**Precision note:** Context mutation budget is checked at cycle start, but `localRules()` can write new context keys later in the same cycle. One cycle's worth of rule writes can overshoot the budget. Accepted imprecision — budget caps are order-of-magnitude safety nets, not precise limits.

## 3. ConvergenceDetector

`ConvergenceDetector` (`runtime-core`, `@ApplicationScoped`) — evaluates convergence during the 5th pipeline phase (`convergenceDetection()`).

### Convergence Condition

ALL four activity rates must be below their respective thresholds simultaneously for a sustained duration:

```
converged = (dispatchRate < dispatchRateThreshold)
          AND (signalDepositRate < signalDepositRateThreshold)
          AND (contextMutationRate < contextMutationRateThreshold)
          AND (evaluationRate < evaluationRateThreshold)
          AND Duration.between(firstQuietCycle, now) >= stabilityWindow
```

### Per-Case State

`ConcurrentHashMap<UUID, ConvergenceState>`:
- `firstQuietCycle: Instant` — when all rates first dropped below threshold (null when any rate exceeds)
- `converged: boolean` — set true on first detection, prevents repeated firing

### Detection Flow

1. Read all four rates from `ActivityTracker`
2. If all below threshold:
   - If `firstQuietCycle == null` → set to `now` (start of quiet period)
   - If `Duration.between(firstQuietCycle, now) >= stabilityWindow` AND not already converged:
     - Write `CONVERGENCE_DETECTED` to EventLog
     - Fire synthetic `GoalReachedEvent` with goal name `"_converged"`
     - Set `converged = true`
3. If any rate exceeds threshold → reset `firstQuietCycle = null`

### Goal Integration

The `_converged` goal fires with `StandardGoalKind.SUCCESS` (terminal status: COMPLETED). If the CaseDefinition declares `_converged` in its `GoalBasedCompletion`, the existing `GoalReachedEventHandler` transitions the case to COMPLETED. If not declared, convergence is detected and audited but does not trigger termination.

```yaml
completion:
  success:
    anyOf: [case-resolved, _converged]
```

The `_` prefix convention distinguishes engine-fired goals from agent-fired goals.

**Async note:** The synthetic `GoalReachedEvent` is published on the event bus and processed asynchronously. Convergence detection and case termination are not atomic within one cycle. This is safe — `GoalReachedEventHandler` checks `currentState.isTerminal()` and `CaseStatusChangedHandler` uses CAS for terminal transitions.

## 4. OutputConvergenceMonitor

`OutputConvergenceMonitor` (`runtime-core`, `@ApplicationScoped`) — tracks per-binding output structural similarity across agents.

### Recording

On each successful worker completion (`WorkflowExecutionCompletedHandler` success path):
1. Extract output `Map<String, Object>`
2. Compute key set and per-key value hash (SHA-256 of canonical JSON)
3. Store in per-binding sliding window (size: `outputWindowSize`, default 10)

Injected into `WorkflowExecutionCompletedHandler` via `Instance<OutputConvergenceMonitor>` with `isResolvable()` guard — transparent no-op when absent.

### Similarity Detection

When `windowSize >= convergenceMinSamples` (default 3):
1. Compute pairwise Jaccard coefficient on key sets across all outputs in the window
2. If average Jaccard > `convergenceThreshold` (default 0.9) AND value hashes match for overlapping keys:
   - Write `OUTPUT_CONVERGENCE_DETECTED` to EventLog
   - Metadata: `bindingName`, `averageJaccard`, `matchingOutputCount`, `totalSamples`, `affectedAgents`

This is an informational event, not a judgment. Structural similarity may indicate independent consensus (correct behavior) or groupthink (a concern). Semantic interpretation belongs in blocks, not the engine.

### Lifecycle

`CaseStatusChangedHandler` calls `outputConvergenceMonitor.evictByCase(caseId)` on terminal status. `OutputConvergenceMonitor implements Resettable`.

## 5. ConvergenceConfig

Three independent config records on `CaseDefinition`:

### BudgetConfig

```java
BudgetConfig(
    Integer maxDispatches,           // null = no limit
    Integer maxSignalDeposits,
    Integer maxContextMutations,
    Integer maxEvaluationCycles
)
```

### ConvergenceThresholdConfig

```java
ConvergenceThresholdConfig(
    Double dispatchRateThreshold,        // default 0.1 events/sec
    Double signalDepositRateThreshold,   // default 0.1
    Double contextMutationRateThreshold, // default 0.1
    Double evaluationRateThreshold,      // default 0.5
    Duration stabilityWindow,            // default 30 seconds
    Duration rateWindow,                 // default 60 seconds
    Integer maxWindowEntries             // null = derived from rateWindow
)
```

### OutputConvergenceConfig

```java
OutputConvergenceConfig(
    Double convergenceThreshold,    // default 0.9
    Integer convergenceMinSamples,  // default 3
    Integer outputWindowSize        // default 10
)
```

`CaseDefinition` gains:
- `budgetConfig` (nullable `BudgetConfig`)
- `convergenceThresholdConfig` (nullable `ConvergenceThresholdConfig`)
- `outputConvergenceConfig` (nullable `OutputConvergenceConfig`)

All nullable — null means disabled (backward compatible). Builder methods and YAML blocks follow the `ObservationConfig`/`SignalConfig`/`RuleConfig` pattern.

### YAML

```yaml
spec:
  budgetConfig:
    maxDispatches: 1000
    maxSignalDeposits: 5000
    maxContextMutations: 10000
    maxEvaluationCycles: 2000

  convergenceThresholdConfig:
    dispatchRateThreshold: 0.1
    signalDepositRateThreshold: 0.1
    contextMutationRateThreshold: 0.1
    evaluationRateThreshold: 0.5
    stabilityWindow: PT30S
    rateWindow: PT60S

  outputConvergenceConfig:
    convergenceThreshold: 0.9
    convergenceMinSamples: 3
    outputWindowSize: 10
```

## 6. Audit Events

Three new `CaseHubEventType` values:

| Event | When | Key Metadata |
|-------|------|-------------|
| `CONVERGENCE_DETECTED` | All activity rates below threshold for `stabilityWindow` | `dispatchRate`, `signalDepositRate`, `contextMutationRate`, `evaluationRate`, `stabilityDuration`, cumulative totals |
| `BUDGET_EXHAUSTED` | Any cumulative count exceeds budget cap | `exhaustedMetric`, `currentCount`, `budgetCap` |
| `OUTPUT_CONVERGENCE_DETECTED` | Output similarity exceeds threshold | `bindingName`, `averageJaccard`, `matchingOutputCount`, `totalSamples`, `affectedAgents` |

All written to EventLog immediately on detection.

## 7. Module Placement

| Type | Package | Module |
|------|---------|--------|
| `BudgetConfig`, `ConvergenceThresholdConfig`, `OutputConvergenceConfig` | `io.casehub.api.model.convergence` | engine-api |
| `ActivityTracker`, `SlidingWindowCounter`, `CaseActivityState` | `io.casehub.engine.common.internal.convergence` | common-core |
| `ConvergenceDetector`, `OutputConvergenceMonitor`, `BudgetEnforcer` | `io.casehub.engine.internal.convergence` | runtime-core |

## 8. Agent Surfacing

No new WorkerRuntime facet. Convergence detection is a system-level supervisory function — agents coordinate via signals, observations, interests, neighbors, and rules. The engine monitors aggregate behavior and intervenes when thresholds are breached. If future issues (#1111-#1115) need agent-visible activity metrics, a read-only `MetricsSpace` facet can be added without changing the tracker infrastructure.

## 9. Relationship to Existing Infrastructure

| Existing | Relationship |
|----------|-------------|
| **Watchdog** (qhorus) | Complementary. Watchdog monitors conversations. Convergence monitors coordination state. No dependency. |
| **GoalBasedCompletion** | Reused. `_converged` goal integrates with existing completion system. |
| **CompoundCompletionEvaluator** | Unmodified. Structural completion is orthogonal to emergent convergence. |
| **QuiescenceTracker** | Complementary. QuiescenceTracker detects "nothing in-flight" (transient). ConvergenceDetector detects "activity has stabilized" (persistent). |
| **DispatchBudget SPI** | Complementary. DispatchBudget is external/advisory. BudgetConfig is internal/enforced. |
| **maxConcurrentDispatches** | Complementary. Concurrent cap (in-flight). BudgetConfig is cumulative cap (lifetime). |

## References

- `CaseContextChangedEventHandler.java:246-257` — evaluation pipeline structure
- `GoalReachedEventHandler.java:102-148` — goal-based completion evaluation
- `GoalBasedCompletion.java`, `GoalKind.java`, `StandardGoalKind.java` — completion model
- `WatchdogRecoveryBridge.java` — watchdog integration pattern
- `QuiescenceTracker.java` — per-case state tracking pattern
- `SignalRegistry.java` — per-case ConcurrentHashMap + Resettable pattern
- `ObservationConfig.java`, `SignalConfig.java`, `RuleConfig.java` — configuration record patterns
- `CaseStatusChangedHandler.java` — terminal state cleanup and eviction
- `WorkflowExecutionCompletedHandler.java` — worker completion handling, Instance<> injection pattern
- engine#1110 — issue specification
- engine#1104 — Hive Mind epic
- arXiv:2510.10047 (SwarmSys) — multi-agent coordination patterns
- D45-D56, D57-D59 — design decisions
