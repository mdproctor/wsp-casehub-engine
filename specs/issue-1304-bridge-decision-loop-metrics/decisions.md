## D1: Overall bridging approach

**Choice:** Approach B — Direct accumulator as `loopEventSink` consumer
**Alternatives:**
- Approach A (ExecutionEventListener wiring) — Initially selected, but self-review found that `AbstractExecutionDriver` and `ExecutionEventListener` operate on the standard five-phase loop path, while `DecisionLoopEvent` is emitted inside coordination topologies (`CentralisedTopology`), which bypass the driver entirely. The two paths never intersect.
- Approach C (Extend AgentCounters) — Fewer new types but couples StigmergyCoordinator to LLM semantics, widens its surface
**Rationale:** The `loopEventSink` is captured at topology construction time inside `CentralisedTopology`, not at driver execution time. A direct `Consumer<DecisionLoopEvent>` accumulator is the only mechanism that can receive events from this path. The accumulator also implements `AgentLlmStatsProvider` to bridge to the engine side.
**Trade-offs:** Does not enable future `ExecutionEventListener` implementations to observe decision loop events (that would require separate work on the driver path). Coordination topologies have their own event routing.
**Sources:** `SwarmCoordinatorBuilder.java:132` (no-op loopEventSink), `CentralisedTopology.java:47` (coordination topology), `AbstractExecutionDriver.java:229-298` (driver path — confirmed separate from topology path)
**Exploration:** quick → revised after self-review
**Status:** revised

## D2: Accumulator location

**Choice:** New standalone `LlmTelemetryAccumulator` in `engine-adapter-core`
**Alternatives:**
- Extend CheckpointingListener — Initially selected, but self-review found `CheckpointingListener` is an `ExecutionEventListener` wired into `ExecutionModel.listeners()` via `PatternWorkerFunctionHandler`. It operates on the standard driver path. Coordination topologies (where `DecisionLoopEvent` is emitted) bypass this listener entirely.
- Engine SPI impl — Clean boundary but still requires engine-adapter implementation
**Rationale:** The accumulator needs to implement both `Consumer<DecisionLoopEvent>` (to serve as the `loopEventSink`) and `AgentLlmStatsProvider` (to expose stats to the engine). `CheckpointingListener` is on the wrong execution path. A standalone class in `engine-adapter-core` is the natural home — it bridges agency events to engine types.
**Trade-offs:** One new class rather than extending an existing one. But the separation is architecturally correct — coordination topology events and driver lifecycle events are distinct concerns.
**Sources:** `CheckpointingListener.java:32-101` (driver path only), `PatternWorkerFunctionHandler.java:91-98` (where CheckpointingListener is wired)
**Exploration:** quick → revised after self-review
**Depends on:** D1 (direct accumulator approach)
**Status:** revised

## D3: Engine-side stats access

**Choice:** New `AgentLlmStatsProvider` SPI interface in `engine/api`
**Alternatives:**
- Query CheckpointingListener directly — Tight coupling, collector would depend on engine-adapter types
- Extend StigmergyCoordinator — No new type but widens coordinator surface with LLM concerns
**Rationale:** Clean contract at the module boundary. `SwarmTelemetryCollector` takes it as an optional constructor parameter. Testable in isolation — tests can provide a stub without the full agency stack.
**Trade-offs:** One new interface and record in `engine/api`. Minimal overhead for a clean boundary.
**Sources:** `SwarmTelemetryCollector.java:37-39` (current constructor), `StigmergyCoordinator.java:522` (AgentCounters pattern)
**Exploration:** quick
**Depends on:** D1 (listener wiring approach)
**Status:** captured

## D4: Invocation count semantic

**Choice:** Count `ReasoningCompleted` events as invocations
**Alternatives:**
- Count LoopTriggered — Different semantic; a loop trigger may produce 0 or many LLM calls
- Track both — More data but more complexity
**Rationale:** Each `ReasoningCompleted` maps 1:1 to an LLM call. Matches the `invocationCount` field name on `SwarmTelemetry`. Simple and accurate.
**Trade-offs:** Does not distinguish between different types of LLM calls (reasoning vs heuristic fallback).
**Sources:** `DecisionLoopEvent.java:61-77` (ReasoningCompleted definition)
**Exploration:** quick
**Status:** captured
