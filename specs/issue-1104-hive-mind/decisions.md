## D1: Core SPI design — Observer-function with key filtering

**Choice:** Approach A — `EnvironmentObserver` is a functional interface (`observe(ObservationContext) → List<Observation>`) with declared `watchedKeys()` for evaluation optimization. Extends `NamedStrategy`.

**Alternatives:**
- Declarative pattern vocabulary (sealed hierarchy + `PatternEvaluator` chain) — closed vocabulary fights LLM extension in blocks #284
- Stateful stream processor (`onContextChanged` + `drain`) — lifecycle complexity not justified; history buffer achieves temporal patterns without observer state

**Rationale:** Maximum flexibility for both engine (classical pattern implementations) and blocks (LLM-backed observation). Key filtering provides bounded evaluation. Single interface, clean contract, consistent with platform `NamedStrategy` convention. History buffer in `ObservationContext` gives temporal capability without per-observer state.

**Trade-offs:** Unbounded observer logic could hang evaluation — need timeout enforcement. Less structured than a pattern vocabulary — observation logic is opaque to the engine (harder to audit what an observer does vs inspecting a serialized pattern).

**Sources:** `CaseContext.java:26`, `ContextChangeTrigger.java:21`, `CaseEvaluationSerializer.java:23`, `CaseContextChangedEventHandler.java:252-354`, issue #1105, arXiv:2512.10166 (Emergent Collective Memory)

**Exploration:** quick
**Status:** captured

## D2: Registration mechanism — WorkerScope at dispatch

**Choice:** Agents register observers through `WorkerScope.registerObserver(EnvironmentObserver)` during worker execution. Engine manages lifecycle — observers are scoped to the agent's `LifecycleScope` (BINDING/COMPOUND/CASE).

**Alternatives:**
- Defer entirely to issue #1107 (Dynamic interest registration) — delays usability of observation SPI
- CDI discovery + case binding — static wiring, doesn't support per-agent dynamic observation interests

**Rationale:** Natural integration point — agents already receive `WorkerScope` as a parameter. Registration during execution means the agent controls what it observes based on its own state and goals. Lifecycle scoping follows the existing `ScopedWorkerRegistry` pattern — no new lifecycle infrastructure needed.

**Trade-offs:** Requires `WorkerScope` and `WorkerRuntime` API extensions. BINDING-scoped observers are destroyed after single dispatch — temporal patterns only work with COMPOUND or CASE scope.

**Sources:** `ScopedWorkerRegistry.java:23`, `WorkerScope` (worker-api), `LifecycleScope` (api/model)
**Depends on:** D1 (SPI design defines what is registered)
**Exploration:** quick
**Status:** captured

## D3: History buffer — Count-bounded + time-bounded sliding window

**Choice:** Keep the last N context snapshots AND discard entries older than T. Both limits enforced, whichever triggers first. Configurable per case via `CaseDefinition` (`maxHistoryEntries`, `maxHistoryAge`). Defaults: 50 entries, 5 minutes.

**Alternatives:**
- Count-bounded only — long-running slow cases lose temporal context
- Time-bounded only — fast-changing cases accumulate unbounded entries

**Rationale:** Dual bounds give predictable memory usage (count cap) while preserving temporal relevance (time cap). Observers inspect the history for temporal patterns (sequences, trends) without maintaining their own state.

**Trade-offs:** Each snapshot is a shallow copy of changed keys (not full context) — limits what temporal observers can inspect to what changed in each event. Full snapshots would be prohibitively expensive.

**Sources:** `CaseEvaluationSerializer.java:23`, `CaseContextChangedEvent.java`
**Depends on:** D1 (history buffer is part of ObservationContext)
**Exploration:** quick
**Status:** captured

## D4: Module placement — api/spi/observation

**Choice:** `EnvironmentObserver`, `ObservationContext`, `Observation` in `engine-api` under `io.casehub.api.spi.observation`. `ObservationRegistry` (per-case in-memory store) in `engine-common` under `io.casehub.engine.common.internal.observation`. Default implementations and handler integration in `runtime`.

**Alternatives:**
- Everything in engine-common — SPIs that don't take common/internal types should live in api/spi per the SPI placement rule
- Dedicated module (casehub-engine-observation) — overkill for a foundation SPI

**Rationale:** Follows the SPI placement rule: operational SPIs go in `api/spi/`. `ObservationRegistry` needs `CaseInstance` references so it goes in `common`. Default no-op and handler integration go in `runtime`.

**Sources:** SPI placement rule in CLAUDE.md, `api/spi/` directory structure
**Depends on:** D1 (placement follows from SPI shape)
**Exploration:** quick
**Status:** captured

## D5: Pipeline integration — After binding dispatch, same serializer

**Choice:** Observation evaluates AFTER `CaseContextChangedEventHandler.rules()` completes, still within the `CaseEvaluationSerializer` gate. Bindings dispatch first (existing behavior untouched), then observers evaluate. Observations are available for the next evaluation cycle's local rules (#1109).

**Alternatives:**
- Before binding dispatch — couples observation to the dispatch pipeline
- Parallel to binding dispatch — race between observations and dispatch results

**Rationale:** Existing dispatch is untouched (additive, not replacement). Serializer gate prevents concurrent observation evaluation for the same case. Observations from cycle N inform local rules in cycle N+1 — no circular dependency.

**Trade-offs:** One-cycle delay between context change and observation availability. Acceptable — observations inform strategy, not immediate dispatch.

**Sources:** `CaseContextChangedEventHandler.java:185-203`, `CaseEvaluationSerializer.java:35-55`
**Depends on:** D1 (SPI design determines evaluation contract), D4 (placement determines where handler lives)
**Exploration:** quick
**Status:** captured
