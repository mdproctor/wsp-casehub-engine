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
