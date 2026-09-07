# Decisions — issue-468-concurrency-throttle-recovery

## D1: Layered intervention architecture

**Choice:** Three independent mechanisms at different points in the execution loop, not a unified abstraction.

- **Budget** (pre-dispatch): case-level cap (engine-internal, from CaseDefinition) + external dispatch budget SPI (engine-api, claudony implements). Sits between `loopControl.select()` and dispatch in `CaseContextChangedEventHandler.rules()`.
- **Watchdog bridge** (post-dispatch): two-channel response based on condition class. Worker-hung conditions (AGENT_STALE, BARRIER_STUCK, LOOP_DETECTED, CONVERSATION_STALL, ECHO_CHAMBER, CIRCULAR_DELEGATION) → cancel affected worker → synthetic `Expired` → existing failure pipeline → RecoveryCoordinator. Case-level conditions (CONTEXT_PRESSURE, CHANNEL_IDLE, QUEUE_DEPTH, OBLIGATION_FAN_OUT, APPROVAL_PENDING, DELIVERY_LAG) → context signal at `.watchdogAlert` → case-definition bindings react (same pattern as existing PathologyCondition).
- **Recovery** (post-failure): existing RecoveryCoordinator, unchanged. Receives genuine worker failures including watchdog-cancelled workers.

Composability: stall resolution and worker completion both free budget implicitly via PlanItem state transitions. CONTEXT_CHANGED re-evaluates and admits more bindings. No new coordination needed.

**Alternatives:**
- Unified CaseIntervention abstraction covering budget + watchdog + recovery — over-abstracts genuinely different operations at different pipeline points
- Extend RecoveryCoordinator.handleFailure() with watchdog data — semantic mismatch (RecoveryContext requires worker-scoped data that watchdog alerts don't have)
- Watchdog routes directly into RecoveryCoordinator — impedance mismatch forces fabrication of synthetic RecoveryContext fields

**Rationale:** Budget is pre-dispatch governance, watchdog is post-dispatch observation, recovery is post-failure escalation. Different data, different actions, different pipeline positions. Forcing a shared abstraction creates semantic tension. The existing CONTEXT_CHANGED loop provides composability without coupling.

**Trade-offs:** No single place to see "all governance" for a case. Each mechanism is independently configurable, which means more config surface on CaseDefinition.

**Sources:**
- `CaseContextChangedEventHandler.rules()` (runtime, line 224-324) — dispatch fan-out, no throttle today
- `RecoveryCoordinator` / `RecoveryContext` (common/spi/recovery/) — worker-failure-scoped
- `WatchdogAlertRouter` / `WatchdogAlertEvent` / `AlertContext` (qhorus-api) — 12 condition types, routes to notification endpoints
- `QhorusMessageSignalBridge.handlePathologyAlert()` (runtime, line 138) — existing pathology→context signal pattern
- `PathologyCondition` enum (api/model/) — existing 4-condition subset

**Exploration:** deep-analysis
**Status:** captured

## D2: Watchdog bridge scope and delivery mechanism

**Choice:** Engine bridge is a CDI `@ObservesAsync WatchdogAlertEvent` observer — qhorus already fires this event. Bridge scope limited to worker-hung conditions only (AGENT_STALE, BARRIER_STUCK, LOOP_DETECTED, CONVERSATION_STALL, ECHO_CHAMBER, CIRCULAR_DELEGATION). Action: cancel the pending Quartz/db-scheduler job for affected workers → synthetic `WorkerOutcome.Expired("Watchdog: <condition>")` → existing failure pipeline → RecoveryCoordinator handles escalation. No context signaling, no notification delivery, no case-level condition handling.

**Alternatives:**
- Extend to case-level conditions (CONTEXT_PRESSURE, QUEUE_DEPTH, etc.) via `.watchdogAlert` context signal — reinvents notification inside the engine; case-level conditions need human judgment, not automated recovery
- Route case-level alerts through platform notification service (inbox/human-attention model) — wrong abstraction; watchdog alerts are operational signals needing durable event delivery (CloudEvents/JMS tier), not human inboxes
- New `WatchdogTrigger` binding type for automated case-level response — deferred; no concrete use case yet where the automated action is clear

**Rationale:** Worker-hung conditions have clear automated responses (cancel → retry → recover). Case-level conditions don't — CONTEXT_PRESSURE is a capacity planning signal, QUEUE_DEPTH is an operational concern. The bridge does the thing that's unambiguously useful (unstick hung workers) and defers the thing that needs more design (automated case-level response, durable operational event bus).

**Trade-offs:** 6 of 12 watchdog conditions have no engine-side automated response in v1. Follow-up issue tracks automated case-level handling.

**Depends on:** D1 (layered architecture — bridge is the post-dispatch layer)

**Sources:**
- `WatchdogEvaluationService` (qhorus runtime) — already fires `alertEvents.fireAsync(WatchdogAlertEvent)`, has containment actions (PAUSE_CHANNEL, DEREGISTER_AGENT, QUARANTINE)
- `WatchdogAlertEvent.context().affectedAgentIds()` — identifies hung agents
- Platform notification service (`io.casehub.platform.api.notification`) — human inbox model (Notification, NotificationStore, subscriptions, preferences, delivery channels), wrong tier for operational signals
- `QhorusMessageSignalBridge.handlePathologyAlert()` — existing PathologyCondition→context signal pattern; NOT reused because context signaling for case-level alerts reinvents notification

**Exploration:** deep-analysis
**Status:** captured
