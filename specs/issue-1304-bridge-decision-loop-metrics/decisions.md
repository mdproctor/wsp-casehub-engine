## D1: Overall bridging approach

**Choice:** Approach A — Complete the intended ExecutionEventListener wiring path
**Alternatives:**
- Approach B (Direct accumulator) — Simpler short-term but bypasses listener architecture, creates parallel event path
- Approach C (Extend AgentCounters) — Fewer new types but couples StigmergyCoordinator to LLM semantics, widens its surface
**Rationale:** The `onDecisionLoopEvent` method already exists on `ExecutionEventListener` but is never called. The `loopEventSink` comment ("wired at execution layer") proves this was the intended design. Completing it follows the established `notifyXxx` pattern in `AbstractExecutionDriver` and enables future listeners to observe decision loop events.
**Trade-offs:** Touches more files across two repos than Approach B. Requires changes to `AbstractExecutionDriver`, `CentralisedTopology`, and `SwarmCoordinatorBuilder` in agency.
**Sources:** `AbstractExecutionDriver.java:229-298` (notify pattern), `SwarmCoordinatorBuilder.java:132` (no-op loopEventSink comment), `ExecutionEventListener.java:29` (existing unused method)
**Exploration:** quick
**Status:** captured

## D2: Accumulator location

**Choice:** Extend CheckpointingListener with `onDecisionLoopEvent` override
**Alternatives:**
- New LlmStatsListener — Cleaner SRP but adds another listener to wire and discover
- Engine SPI impl — Clean boundary but still requires engine-adapter implementation
**Rationale:** `CheckpointingListener` already bridges agency execution events to engine persistence. It handles `onActivation`, `onAgentResult`, `onTermination`. Adding LLM stats accumulation keeps all engine-adapter bridging in one class.
**Trade-offs:** `CheckpointingListener` grows in responsibility. If future event types need accumulation, it may need splitting.
**Sources:** `CheckpointingListener.java:32-101` (current implementation)
**Exploration:** quick
**Depends on:** D1 (listener wiring approach)
**Status:** captured

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
