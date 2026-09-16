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

**Choice:** Agents register observers through `WorkerRuntime.registerObserver(EnvironmentObserver)` during worker execution. Engine manages lifecycle — observers are scoped to the agent's `LifecycleScope` (BINDING/COMPOUND/CASE). `WorkerRuntime` (in engine-api) is the correct interface — `WorkerScope` (in worker-api) cannot reference engine-api types due to dependency direction (engine → worker, not reverse).

**Alternatives:**
- Defer entirely to issue #1107 (Dynamic interest registration) — delays usability of observation SPI
- CDI discovery + case binding — static wiring, doesn't support per-agent dynamic observation interests

**Rationale:** Natural integration point — agents already receive `WorkerScope` as a parameter. Registration during execution means the agent controls what it observes based on its own state and goals. Lifecycle scoping follows the existing `ScopedWorkerRegistry` pattern — no new lifecycle infrastructure needed.

**Trade-offs:** Requires `WorkerScope` and `WorkerRuntime` API extensions. BINDING-scoped observers are destroyed after single dispatch — temporal patterns only work with COMPOUND or CASE scope.

**Sources:** `ScopedWorkerRegistry.java:23`, `WorkerRuntime.java:24` (engine-api), `WorkerScope` (worker-api), `LifecycleScope` (api/model)
**Depends on:** D1 (SPI design defines what is registered)
**Exploration:** quick
**Status:** revised — corrected `WorkerScope` to `WorkerRuntime`; `WorkerScope` (worker-api) cannot reference engine-api types

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

**Choice:** Observation evaluates AFTER `CaseContextChangedEventHandler.rules()` completes, still within the `CaseEvaluationSerializer` gate. Bindings dispatch first (existing behavior untouched), then observers evaluate. Observers must have bounded execution time (<100ms target) — the gate serializes all evaluation for a case. Observations are stored in ObservationRegistry (see D7) and available to the next evaluation cycle's local rules (#1109). LLM-backed observation (blocks #284) does NOT execute inside this gate — the engine observer detects a trigger pattern; the LLM call is dispatched as a separate worker.

**Alternatives:**
- Before binding dispatch — couples observation to the dispatch pipeline
- Parallel to binding dispatch — race between observations and dispatch results

**Rationale:** Existing dispatch is untouched (additive, not replacement). Serializer gate prevents concurrent observation evaluation for the same case. Observations from cycle N inform local rules in cycle N+1 — no circular dependency.

**Trade-offs:** One-cycle delay between context change and observation availability. Acceptable — observations inform strategy, not immediate dispatch. Slow observers degrade evaluation throughput for the entire case — timeout enforcement is the safety net (see D6).

**Sources:** `CaseContextChangedEventHandler.java:185-203`, `CaseEvaluationSerializer.java:35-55`, issue #1104 ("Engine mechanics, blocks intelligence")
**Depends on:** D1 (SPI design determines evaluation contract), D4 (placement determines where handler lives), D6 (thread model), D7 (materialization)
**Exploration:** quick
**Status:** revised — added bounded execution constraint; clarified LLM-backed observation is out of scope for serializer gate

## D6: Observer thread model — Synchronous, bounded execution

**Choice:** All observers execute synchronously within the evaluation cycle (inside the serializer gate per D5). Observers must have bounded execution time — target <100ms per observer. The SPI contract (D1) is a synchronous functional interface; the engine calls `observe()` and uses the returned `List<Observation>` immediately. LLM-backed observation (blocks #284) does not call LLM inside the observer — the engine observer detects a fast trigger pattern, and the LLM call is dispatched as a separate worker via normal binding dispatch.

**Alternatives:**
- Asynchronous evaluation — decouples from evaluation cycle, loses deterministic ordering guarantee, indeterminate availability of results in cycle N+1
- Hybrid sync/async with marker interface — complexity of two execution models, interaction semantics undefined, harder to reason about

**Rationale:** Engine-level observation is mechanical pattern detection — multi-key correlation, threshold crossings, temporal sequence detection. These are fast, bounded computations. The "engine mechanics, blocks intelligence" split from epic #1104 means the engine provides the detection mechanism; blocks provides the LLM intelligence that acts on detections. The observer detects; a worker dispatch handles the LLM call.

**Trade-offs:** Slow observers degrade evaluation throughput. Timeout enforcement is the safety net — observers exceeding the bound are interrupted and their contribution lost for that cycle.

**Sources:** `CaseEvaluationSerializer.java:23`, issue #1104 ("Engine mechanics, blocks intelligence")
**Depends on:** D1 (SPI design), D5 (pipeline integration)
**Exploration:** quick (surfaced by review R1-03, R1-07)
**Status:** captured

## D7: Observation materialization — ObservationRegistry, no CaseContext writes

**Choice:** Observations produced by observers are stored in the `ObservationRegistry` (per-case, in-memory, in `common/internal` per D4). Observations are NOT written to `CaseContext`. Rules in cycle N+1 query the `ObservationRegistry` directly via a mechanism defined by issue #1109 (local rule evaluation). Observations decay or are replaced when the next observation cycle runs for that case.

**Alternatives:**
- Write to CaseContext working layer — creates feedback loop (write → `CaseContextChangedEvent` → re-evaluation → observe → write → ...)
- Dedicated context layer (e.g., OBSERVATION) with handler skip — requires layer infrastructure changes, couples observation to context layer model
- Context key prefix (`_observations.*`) with `engineSet` — still triggers `CaseContextChangedEvent` through event bus, still creates feedback loop

**Rationale:** `ObservationRegistry` is already defined in D4 for per-case observation storage. Using it as the materialization target avoids feedback loops entirely — registry writes don't trigger `CaseContextChangedEvent`. Rules access observations through a clean query interface rather than through context key conventions. This cleanly separates observation data from case domain state.

**Trade-offs:** Rules need a mechanism to query the `ObservationRegistry` — depends on #1109 (local rule evaluation). Until #1109 is implemented, observations are stored but not consumed by rules.

**Sources:** `CaseContextChangedEventHandler.java:175-185`, `CaseContextImpl.java:46`, issue #1109, issue #1110
**Depends on:** D1, D4, D5
**Exploration:** quick (surfaced by review R1-05, R1-08)
**Status:** captured

## D8: Observer error isolation — Per-observer try-catch, log and skip

**Choice:** Each observer evaluates inside its own try-catch block. On failure: log at WARN level, skip the observer, continue evaluating remaining observers. A failing observer loses its contribution for that cycle only — it does not block other observers or the evaluation pipeline. The observer will be re-evaluated in the next cycle.

**Alternatives:**
- Kill entire evaluation cycle on any observer failure — one bad observer blocks all binding dispatch and goal evaluation
- Silently swallow all errors — hides observation data loss, makes debugging impossible

**Rationale:** Follows the existing pattern in the evaluation pipeline where individual binding dispatches are isolated (try-catch in `evaluateAndDispatch`). Observer failure is not fatal — the observation is a best-effort contribution to the next cycle. WARN-level logging makes failures visible without propagating exceptions up the evaluation chain.

**Sources:** `CaseContextChangedEventHandler.java` (evaluateAndDispatch exception handling pattern)
**Depends on:** D5 (pipeline integration), D6 (synchronous execution)
**Exploration:** quick (surfaced by review R1-12)
**Status:** captured

## D9: Observer cardinality — CaseDefinition maxObserversPerCase

**Choice:** Maximum number of observers per case is configurable via `CaseDefinition.maxObserversPerCase`. Default: 20. Exceeding the cap logs WARN and silently drops the registration attempt. Per-binding observer count is uncapped within the per-case limit.

**Alternatives:**
- Unbounded registration — accumulation risk in swarm scenarios (#1112) with many agents registering observers
- Hard-coded cap — not configurable for different case types with different observation needs
- Per-agent cap — harder to enforce, doesn't address the aggregate latency problem

**Rationale:** The serializer gate (D5) means every observer adds latency to the evaluation cycle. Bounding the count bounds the worst-case evaluation time (20 observers × 100ms target = 2s worst case). The cap is per-case (not per-agent) because aggregate evaluation latency is what matters. `CaseDefinition` is the natural configuration surface, consistent with existing `maxConcurrentDispatches`.

**Sources:** `CaseDefinition` (`maxConcurrentDispatches` pattern), issue #1112 (swarm scenarios)
**Depends on:** D2 (registration mechanism), D5 (pipeline integration), D6 (thread model)
**Exploration:** quick (surfaced by review R1-14)
**Status:** captured

## D10: Signal storage — Dedicated SignalRegistry, not CaseContext

**Choice:** Signals live in a dedicated `SignalRegistry` (`engine-common`, `@ApplicationScoped`, `Resettable`), following the `ObservationRegistry` pattern. Not in CaseContext working layer. Workers read/write via `WorkerRuntime`. Observers perceive signals through an injected signal view. No CaseContext writes means no feedback loops.

**Alternatives:**
- CaseContext working layer at `_signals.<name>` — simpler, but `engineSet()` doesn't fire change listeners so observations can't detect signal changes via standard `changedKeys`. Also risks feedback loops without careful version-suppression discipline.
- Dedicated `SIGNAL` context layer — clean separation, but requires layer infrastructure changes rejected in D7

**Rationale:** #1105 Decision D7 established that coordination state should not live in the domain data layer. Signals are coordination primitives consumed by observers and local rules, not domain facts consumed by business triggers. Registry-based storage also enables lazy decay computation at read time without periodic CaseContext rewrites.

**Trade-offs:** Signals are not automatically visible to JQ trigger conditions (which evaluate against CaseContext). Agents that want to trigger bindings based on signal state must use the observation pipeline (#1109 local rules). REST visibility requires explicit snapshot serialization.

**Sources:** `ObservationRegistry.java:27`, `WritableLayerImpl.java:672` (engineSet), `CaseContextChangedEventHandler.java:1168` (observation pipeline), D7 (observation materialization), engine#1105, engine#1106
**Depends on:** D7 (materialization pattern)
**Exploration:** quick
**Status:** captured
