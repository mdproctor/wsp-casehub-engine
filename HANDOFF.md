# HANDOFF — casehub-engine

## Last Session

Working on #1304 (bridge DecisionLoopEvent metrics into SwarmTelemetry LLM fields). Cross-repo implementation spanning engine and agency repos.

### Engine (casehubio/engine#1304) — branch `issue-1304-bridge-decision-loop-metrics`
- Created `AgentLlmStatsProvider` SPI in `api/src/main/java/io/casehub/api/spi/stigmergy/`
- Modified `SwarmTelemetryCollector` to accept optional provider, query it instead of hardcoded zeros
- Added 3 new tests: provider supplies stats, null provider preserves zeros, per-agent stats
- All engine tests pass

### Agency (casehubio/agency#444) — committed to main
- Added `loopEventSink(Consumer<DecisionLoopEvent>)` builder method to `SwarmCoordinatorBuilder`
- Replaced hardcoded `e -> {}` with configurable sink (default no-op for backward compat)
- Created `LlmTelemetryAccumulator` in `engine-adapter-core` implementing both `Consumer<DecisionLoopEvent>` and `AgentLlmStatsProvider`
- All accumulator tests pass (5/5). Builder tests verified clean via IDE diagnostics (pre-existing `CuriosityDrive` error in `AgencyBeans.java` blocks `agency` module compilation)

### Design Decisions
- Self-review caught that `AbstractExecutionDriver` and `ExecutionEventListener` operate on the standard five-phase loop path, while `DecisionLoopEvent` is emitted inside coordination topologies (`CentralisedTopology`) which bypass the driver. The accumulator is wired directly as the `loopEventSink` consumer, not through the driver's listener chain.
- `CaseContextChangedEventHandler` not changed — null provider = zeros, correct until coordination topologies are integrated with the engine runtime.

### Pre-existing Blocker
`AgencyBeans.java:305` has a `CuriosityDrive` constructor mismatch that blocks compilation of the `agency` module. Builder tests in that module cannot run via Maven or IntelliJ. The `engine-adapter` module compiles and tests fine independently.

## Immediate Next Step

Run `work end` on the engine branch to close #1304. This is the last issue before epic #1281 (Swarm Mechanical Enforcement Layer) can close.

## Queue Status

Epic #1281 sub-issues: all complete except #1304 (in progress on branch).

## References

- `specs/issue-1304-bridge-decision-loop-metrics/2026-11-08-bridge-decision-loop-metrics-design.md`
- `specs/issue-1304-bridge-decision-loop-metrics/decisions.md` (D1–D4, D1-D2 revised after self-review)
- `plans/2026-11-08-bridge-decision-loop-metrics.md`
