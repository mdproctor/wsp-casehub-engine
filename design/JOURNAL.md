# Design Journal — Hive Mind Epic

## 2026-09-16 — Environment Observation SPI (#1105)

**What:** Built the perception layer for stigmergic coordination — `EnvironmentObserver` SPI enabling agents to observe CaseContext state and detect patterns beyond static JQ triggers.

**Key design decision:** Observation is a separate perception layer orthogonal to binding dispatch. Observers produce structured `Observation` results consumed by local rules (#1109), NOT trigger bindings. This keeps the existing `ContextChangeTrigger` model untouched.

**Architecture:** `EnvironmentObserver` evaluates reactively after `rules()` and `goals()` inside the `CaseEvaluationSerializer` gate. Per-observer 100ms timeout via `CompletableFuture.orTimeout()`. Key filtering optimization skips observers whose `watchedKeys()` are disjoint with changed keys. Results stored in `ObservationRegistry` (ephemeral, per-case in-memory).

**Review findings that shaped the design:**
- `EnvironmentObserver` does NOT extend `NamedStrategy` — observers aren't strategy-resolved, they're registered programmatically via `WorkerRuntime`
- BINDING-scoped observer registration rejected — no lifecycle cleanup event exists for BINDING scope
- `changedKeys` computed via diff against stored `lastProcessedSnapshot` in `ContextHistoryBuffer` — self-contained, no changes to `CaseContextChangedEvent`
- `Observation.details` uses `Map<String, JsonNode>` (not `Map<String, Object>`) for serialization safety
- Neocortex memory integration explicitly deferred — bounded evaluation (<100ms) precludes synchronous memory queries

**Delivered:** 5 SPI types, 2 infrastructure components, registration API, pipeline integration, lifecycle cleanup, 3 classical observers (threshold, correlation, temporal sequence), 50 tests.

**Next:** #1106 — Signal/pheromone model with decay and reinforcement.
