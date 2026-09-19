## D1: Core SPI design — Observer-function with key filtering

**Choice:** Approach A — `EnvironmentObserver` is a functional interface (`observe(ObservationContext) → List<Observation>`) with declared `watchedKeys()` for evaluation optimization. Does NOT extend `NamedStrategy` — observers are identified by `observerType()` (analogous to `NamedStrategy.id()` but not the same interface) and are NOT resolved via `EngineStrategyResolver`. They are created programmatically by workers and registered via `WorkerRuntime`.

**Alternatives:**
- Declarative pattern vocabulary (sealed hierarchy + `PatternEvaluator` chain) — closed vocabulary fights LLM extension in blocks #284
- Stateful stream processor (`onContextChanged` + `drain`) — lifecycle complexity not justified; history buffer achieves temporal patterns without observer state

**Rationale:** Maximum flexibility for both engine (classical pattern implementations) and blocks (LLM-backed observation). Key filtering provides bounded evaluation. Single interface, clean contract. History buffer in `ObservationContext` gives temporal capability without per-observer state.

**Trade-offs:** Unbounded observer logic could hang evaluation — need timeout enforcement. Less structured than a pattern vocabulary — observation logic is opaque to the engine (harder to audit what an observer does vs inspecting a serialized pattern).

**Sources:** `CaseContext.java:26`, `ContextChangeTrigger.java:21`, `CaseEvaluationSerializer.java:23`, `CaseContextChangedEventHandler.java:252-354`, issue #1105, arXiv:2512.10166 (Emergent Collective Memory)

**Exploration:** quick
**Status:** revised — corrected: EnvironmentObserver does NOT extend NamedStrategy; removed inaccurate NamedStrategy convention claim from rationale

## D2: Registration mechanism — WorkerScope at dispatch

**Choice:** Agents register observers through `WorkerRuntime.registerObserver(EnvironmentObserver)` during worker execution. Engine manages lifecycle — observers are scoped to the agent's `LifecycleScope` (BINDING/COMPOUND/CASE). `WorkerRuntime` (in engine-api) is the correct interface — `WorkerScope` (in worker-api) cannot reference engine-api types due to dependency direction (engine → worker, not reverse). Under D19 faceting, `registerObserver()` moves to `InterestSpace` — the flat method on `WorkerRuntime` is removed (pre-release clean break). Registration path becomes `runtime.interests().register(InterestDeclaration)` (D20) or `runtime.interests().registerObserver(EnvironmentObserver)` for programmatic observers.

**Alternatives:**
- Defer entirely to issue #1107 (Dynamic interest registration) — delays usability of observation SPI
- CDI discovery + case binding — static wiring, doesn't support per-agent dynamic observation interests

**Rationale:** Natural integration point — agents already receive `WorkerScope` as a parameter. Registration during execution means the agent controls what it observes based on its own state and goals. Lifecycle scoping follows the existing `ScopedWorkerRegistry` pattern — no new lifecycle infrastructure needed.

**Trade-offs:** Requires `WorkerScope` and `WorkerRuntime` API extensions. BINDING-scoped observers are destroyed after single dispatch — temporal patterns only work with COMPOUND or CASE scope.

**Sources:** `ScopedWorkerRegistry.java:23`, `WorkerRuntime.java:24` (engine-api), `WorkerScope` (worker-api), `LifecycleScope` (api/model)
**Depends on:** D1 (SPI design defines what is registered)
**Exploration:** quick
**Status:** revised — corrected `WorkerScope` to `WorkerRuntime`; clarified that `registerObserver()` flat method is superseded by `InterestSpace.register()` under D19 faceting

## D3: History buffer — Count-bounded + time-bounded sliding window

**Choice:** Keep the last N context snapshots AND discard entries older than T. Both limits enforced, whichever triggers first. Configurable per case via `CaseDefinition` (`maxHistoryEntries`, `maxHistoryAge`). Defaults: 50 entries, 5 minutes.

**Alternatives:**
- Count-bounded only — long-running slow cases lose temporal context
- Time-bounded only — fast-changing cases accumulate unbounded entries

**Rationale:** Dual bounds give predictable memory usage (count cap) while preserving temporal relevance (time cap). Observers inspect the history for temporal patterns (sequences, trends) without maintaining their own state. Default rationale: 50 entries × ~100ms per context change = ~5 seconds of temporal context — sufficient for 3–5 event correlation windows in active cases. 5-minute time cap covers temporal patterns spanning minutes of wall-clock time. Both defaults are provisional and should be validated with deployment telemetry; cases with significantly different activity patterns should configure explicitly.

**Trade-offs:** Each snapshot is a shallow copy of changed keys (not full context) — limits what temporal observers can inspect to what changed in each event. Full snapshots would be prohibitively expensive.

**Sources:** `CaseEvaluationSerializer.java:23`, `CaseContextChangedEvent.java`
**Depends on:** D1 (history buffer is part of ObservationContext)
**Exploration:** quick
**Status:** revised — R1-09: documented default rationale, marked defaults as provisional pending deployment telemetry

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
**Depends on:** D1 (SPI design determines evaluation contract), D4 (placement determines where handler lives), D6 (thread model)
**Exploration:** quick
**Status:** revised — added bounded execution constraint; clarified LLM-backed observation is out of scope for serializer gate; removed circular D7 dependency (D5 is independent of materialization target)

## D6: Observer thread model — Synchronous, bounded execution

**Choice:** All observers execute within the evaluation cycle (inside the serializer gate per D5), blocking-synchronous from the evaluator's perspective. All observers across all agents are dispatched concurrently to virtual threads via `ExecutorService.submit(Callable)`, each returning a `Future`. The evaluator collects all futures and calls `future.get(100, MILLISECONDS)` with a collective 100ms timeout. On `TimeoutException`, the observer's virtual thread is interrupted via `future.cancel(true)` — ensuring the observer actually stops executing, not just the future completing exceptionally. Observers run in parallel — worst-case evaluation time is 100ms regardless of observer count (vs N×100ms sequential). Observer code runs on the virtual thread pool: implementations must be thread-safe, should avoid `synchronized` blocks (which pin platform threads — a known virtual thread anti-pattern), and should be responsive to `Thread.interrupted()`. The SPI contract (D1) is a synchronous functional interface; the engine calls `observe()` and uses the returned `List<Observation>` immediately. LLM-backed observation (blocks #284) does not call LLM inside the observer — the engine observer detects a fast trigger pattern, and the LLM call is dispatched as a separate worker via normal binding dispatch.

**Alternatives:**
- Asynchronous evaluation — decouples from evaluation cycle, loses deterministic ordering guarantee, indeterminate availability of results in cycle N+1
- Hybrid sync/async with marker interface — complexity of two execution models, interaction semantics undefined, harder to reason about

**Rationale:** Engine-level observation is mechanical pattern detection — multi-key correlation, threshold crossings, temporal sequence detection. These are fast, bounded computations. The "engine mechanics, blocks intelligence" split from epic #1104 means the engine provides the detection mechanism; blocks provides the LLM intelligence that acts on detections. The observer detects; a worker dispatch handles the LLM call.

**Trade-offs:** Slow observers degrade evaluation throughput. Timeout enforcement is the safety net — observers exceeding the timeout have their virtual thread interrupted via `Future.cancel(true)` and their contribution lost for that cycle. Implementation: each observer is submitted via `ExecutorService.submit(Callable)` to obtain a `Future`, collected with `future.get(100, MILLISECONDS)`, and on `TimeoutException` cancelled via `future.cancel(true)` to interrupt the virtual thread. This differs from `CompletableFuture.orTimeout()` which only completes the future exceptionally but does NOT interrupt the underlying thread — the observer would continue executing indefinitely. The SPI contract documents that observer implementations should be responsive to `Thread.interrupted()` — long-running computations should periodically check the interrupt flag, and I/O operations should use interruptible channels.

**Sources:** `CaseEvaluationSerializer.java:23`, issue #1104 ("Engine mechanics, blocks intelligence"), JDK `CompletableFuture.orTimeout()` javadoc (confirms no thread interruption)
**Depends on:** D1 (SPI design), D5 (pipeline integration)
**Exploration:** quick (surfaced by review R1-03, R1-07)
**Status:** revised — R1-04: changed from sequential to parallel observer evaluation; collective 100ms timeout via CompletableFuture.allOf(); R1-02: changed timeout mechanism from CompletableFuture.orTimeout to Future.cancel(true) for actual thread interruption; ADR-R2-02: reconciled Choice paragraph to describe the prescribed Future.cancel(true) mechanism consistently (was: still describing the pre-revision CompletableFuture.orTimeout pattern)

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

**Rationale:** The serializer gate (D5) means observers add latency to the evaluation cycle. With parallel evaluation (D6 revision), worst-case evaluation time is the collective timeout (100ms) regardless of observer count. The per-case cap bounds memory and prevents registration exhaustion in swarm scenarios. `CaseDefinition` is the natural configuration surface, consistent with existing `maxConcurrentDispatches`.

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

## D11: Decay model — Exponential decay at read time

**Choice:** `effectiveStrength = initialStrength * e^(-λ * elapsed)` where `λ = ln(2) / halfLife` and `elapsed = now - lastReinforced`. Decay is never physically applied — the stored signal retains its original strength and timestamp. Every read computes the perceived strength lazily. `halfLife` is configurable per-case via `SignalConfig` on `CaseDefinition`. Sub-minute granularity (Duration, not days).

**Alternatives:**
- Discrete-step decay on each evaluation cycle (`strength *= (1-α)` per cycle) — couples decay rate to evaluation frequency, fast-changing cases decay faster than slow ones
- Clock-based periodic decay via scheduler — over-engineered for a pure perception operation

**Rationale:** Mathematically equivalent to continuous evaporation. Stateless — no mutation of stored values. Decouples decay from evaluation frequency. Half-life parameterization is intuitive ("loses half its strength every 5 minutes") and consistent with CBR's `temporalDecayHalfLifeDays` model already in the platform.

**Trade-offs:** Every read pays the `exp()` computation — acceptable since it's O(1) per signal and the registry bounds signal count per case. Signals that have decayed below threshold are still stored (never physically deleted) — requires effective-zero filtering at read time.

**Sources:** `CbrConfig.temporalDecayHalfLifeDays` (CBR temporal decay precedent), `DispositionSignalStore` (eidos signal decay pattern), engine#1106 issue spec (ACO formula), arXiv:2512.10166
**Depends on:** D10 (registry-based storage enables lazy read-time computation)
**Exploration:** quick
**Status:** captured

## D12: Signal identity — Name-keyed with reinforcement

**Choice:** A signal is a single value per `(caseId, signalName)`. When multiple agents write the same signal name, the write is a reinforcement: strength is set to `max(currentEffective, newStrength)` where `currentEffective = SignalDecay.effectiveStrength(existing.strength(), existing.lastReinforced(), existing.halfLife(), now)` — comparing the DECAYED effective strength against the new deposit strength — and `halfLife` is set to `max(existing.halfLife(), newHalfLife)`. Timestamp resets to now and `reinforcementCount` increments. `lastSource` (agent ID) is tracked for audit. This makes heavily-trafficked signals persist longer — exactly the ACO behavior. Reinforcement can only increase strength and slow decay, never weaken a signal.

**Alternatives:**
- Per-agent signal instances `(caseId, signalName, agentId)` with aggregation — more faithful to multi-ant pheromone, but aggregation strategy becomes a sub-decision, N entries per signal per agent
- Append-only signal log — maximally faithful but unbounded storage, O(N) reads

**Rationale:** `max()` on decayed effective strength means reinforcement takes the stronger of the two effective signals: if the existing signal has decayed below the new deposit's strength, the new deposit's value wins; if the existing signal is still stronger, it retains its effective strength with a fresh timestamp. This avoids the resurrection problem where a weak reinforcement would restore a fully-decayed signal to its historical peak. `reinforcementCount` separately represents the breadth of consensus. Observers can weigh these independently — e.g., `effectiveStrength * log(reinforcementCount)` for consensus-weighted strength. With additive-and-clamp (`min(1.0, current + deposit)`), these dimensions collapse: 3 deposits of 0.4 saturate to 1.0, making strength meaningless and losing individual signal quality to clamping. The `max()` semantics also mean reinforcement resets the decay timestamp, so frequently-reinforced signals persist longer — consensus manifests through temporal persistence, not strength amplification. Using `max()` on halfLife means reinforcement can only slow decay, never accelerate it — an agent cannot "hijack" another agent's signal by reinforcing with a shorter halfLife. Note: this DIFFERS from classical ACO where pheromone deposit is additive (`τ ← τ + Σ Δτ`). The platform's signal model is stigmergy-inspired but not an ACO implementation — agents have varying confidence levels and the strongest endorsement should dominate strength, while consensus is captured separately via `reinforcementCount`.

**Trade-offs:** Loses individual agent contribution history. If two agents reinforce and then one "retracts," there's no mechanism to reduce strength other than natural decay. Acceptable — pheromone trails don't support retraction in the biological model either.

**Sources:** engine#1106 issue spec (reinforcement model), ACO literature (pheromone deposit/evaporation), `DispositionSignalStore` (eidos uses per-agent signals — different use case, personality is inherently per-agent)
**Depends on:** D10 (registry storage), D11 (read-time decay)
**Exploration:** quick
**Status:** revised — R1-01/R1-02: halfLife uses `max(existing.halfLife(), newHalfLife)` to prevent decay-rate hijacking; ADR-R1-03: strength comparison changed to DECAYED effective value `max(currentEffective, newStrength)` — avoids resurrection of fully-decayed signals while preserving max-semantics for undecayed reinforcement

## D13: Worker API — WorkerRuntime methods for deposit and perception

**Choice:** `WorkerRuntime` (engine-api) gains `depositSignal(String name, double strength)` and `perceiveSignals() → Map<String, PerceivedSignal>`. `depositSignal` deposits or reinforces a signal (per D12). `perceiveSignals` returns only signals above the effective-zero threshold, with `effectiveStrength`, `reinforcementCount`, `lastSource`, `age` on `PerceivedSignal`. `DefaultWorkerRuntime` delegates to injected `SignalRegistry`.

**Alternatives:**
- Dedicated `SignalService` CDI bean — requires CDI injection in worker functions; workers currently only receive `WorkerScope`/`WorkerRuntime` as parameters
- Signal deposit via `WorkerResult` metadata — doesn't support mid-execution perception or multi-deposit within a single worker run

**Rationale:** `WorkerRuntime` is the established coordination surface — `registerObserver()` already lives there from #1105. Adding signal methods keeps the pattern consistent. Workers already cast `WorkerScope` to `WorkerRuntime` for engine methods, so no new injection mechanism is needed.

**Trade-offs:** `WorkerRuntime` grows wider (two more methods). Acceptable — it's the coordination API surface for agents, and these are fundamental coordination primitives. `default` methods on the interface with no-op returns preserve backward compat.

**Sources:** `WorkerRuntime.java` (registerObserver, execute, spawnCase), `DefaultWorkerRuntime` (runtime delegation pattern), D2 (registration via WorkerRuntime precedent), engine#1106
**Depends on:** D10 (registry storage), D11 (decay model), D12 (reinforcement mechanics)
**Exploration:** quick
**Status:** captured

## D14: Observation integration — ObservationContext gains signals() accessor

**Choice:** `ObservationContext` (engine-api record) gains `Map<String, PerceivedSignal> signals()`. The `observations()` method in `CaseContextChangedEventHandler` reads from `SignalRegistry`, applies decay and effective-zero filtering, and passes the result into the `ObservationContext` constructor. Observers inspect `signals()` alongside `snapshot()` — no separate signal-change detection mechanism.

**Alternatives:**
- Separate `SignalObserver` interface with `watchedSignals()` — duplicates observation pipeline infrastructure
- Inject `SignalRegistry` into observers directly — leaks engine-common internals into engine-api SPI types

**Rationale:** Minimal extension — one new field on an existing record. Observers already have full context access; adding the signal view is the natural extension. Signal-aware classical observers (e.g., `SignalStrengthObserver`) follow the existing `ThresholdObserver` pattern.

**Trade-offs:** Signals are evaluated for all observers on every cycle, even those that don't use them. Acceptable — reading from the registry and filtering is O(S) where S is bounded by per-case signal count (small). Observers that don't care about signals simply ignore the field.

**Sources:** `ObservationContext.java` (existing record), `CaseContextChangedEventHandler.java:1168` (observations() method), `ThresholdObserver.java` (classical observer pattern), D1 (observer SPI), D10 (registry storage)
**Depends on:** D10 (signals in registry), D11 (decay at read time), D1 (observer SPI design)
**Exploration:** quick
**Status:** captured

## D15: Configuration — SignalConfig on CaseDefinition

**Choice:** `SignalConfig` record in engine-api: `SignalConfig(Duration defaultHalfLife, double effectiveZeroThreshold, int maxSignalsPerCase)`. Defaults: `halfLife = 5 minutes`, `effectiveZeroThreshold = 0.01`, `maxSignalsPerCase = 100`. Per-signal half-life override via `depositSignal(String name, double strength, Duration halfLife)` overload — falls back to case-level default when not provided. YAML: `signalConfig:` block under `spec:`. Follows `ObservationConfig` pattern.

**Alternatives:**
- Per-signal-name configuration in CaseDefinition YAML — overly rigid; stigmergy is emergent, agents should create ad-hoc signals dynamically
- No configuration, hardcoded defaults — doesn't allow tuning for different case types (fast-turnaround vs multi-day cases need different decay rates)

**Rationale:** Case-level defaults with per-signal override at deposit time gives the right balance. Dynamic signal creation remains possible — no pre-declaration required. Effective-zero threshold prevents unbounded growth of dead signals polluting the perception view. Pattern follows `ObservationConfig` exactly.

**Trade-offs:** Per-signal half-life override is only available at deposit time (via `WorkerRuntime`), not in YAML. Acceptable — YAML declares the system defaults, workers tune at runtime. `maxSignalsPerCase` bounds memory even in swarm scenarios (#1112).

**Sources:** `ObservationConfig.java` (record pattern), `CaseDefinition` (config surface), `CbrConfig.temporalDecayHalfLifeDays` (precedent for temporal config), engine#1106
**Depends on:** D10 (registry storage), D11 (decay model)
**Exploration:** quick
**Status:** captured

## D16: Audit — EventLog for deposit and effective-zero expiry

**Choice:** Two new `CaseHubEventType` values: `PHEROMONE_DEPOSITED` (on every deposit/reinforcement — metadata: `signalName`, `strength`, `reinforcementCount`, `source`, `halfLife`) and `PHEROMONE_EXPIRED` (when a signal crosses effective-zero threshold during a read — metadata: `signalName`, `finalStrength`, `totalReinforcementCount`, `lifetimeMs`). `PHEROMONE_EXPIRED` is fired lazily once during the `observations()` pipeline and the signal is marked as expired in the registry. No EventLog on perception reads.

**Alternatives:**
- Deposit only, no expiry tracking — loses ability to audit signal lifetimes and diagnose swarm behavior
- Full audit (deposit, reinforce, perceive, expire) — prohibitively noisy; perception events fire N×M per cycle

**Rationale:** Deposit events capture coordination intent. Expiry events close the audit loop — signal lifetimes are reconstructible from `PHEROMONE_DEPOSITED → PHEROMONE_EXPIRED` pairs. Perception is a read operation; the observation pipeline already has its own audit path (`OBSERVATION_DETECTED`). Event type naming uses "pheromone" (matching the stigmergic metaphor) rather than "signal" — consistent with the existing `CaseHubEventType` enum values and the companion spec.

**Trade-offs:** Expiry detection is lazy (only fires when the `observations()` pipeline runs). A signal could be effectively zero between evaluation cycles without an immediate event. Acceptable — signals are a coordination tool, not a real-time alerting mechanism.

**Sources:** `CaseHubEventType` (existing event type pattern), `OBSERVER_REGISTERED`/`OBSERVATION_DETECTED` (observation audit precedent from #1105), engine#1106
**Depends on:** D10 (registry storage), D11 (decay model), D5 (pipeline integration)
**Exploration:** quick
**Status:** revised — corrected event type names from SIGNAL_* to PHEROMONE_* to match codebase and companion spec

## D17: Module placement — api/model + common/internal split

**Choice:** `Signal` (stored value type), `PerceivedSignal` (read model), `SignalConfig` in `engine-api` under `io.casehub.api.model.signal`. `SignalRegistry` in `engine-common` under `io.casehub.engine.common.internal.signal`. `WorkerRuntime` method additions in `engine-api` (existing file). Deposit event publishing, expiry detection, and handler integration in `runtime`. Package is `api.model.signal` (not `api.spi`) — signals are model types consumed by workers, not an SPI that workers implement.

**Alternatives:**
- Everything in engine-common — violates SPI placement rule; `PerceivedSignal` is returned via `WorkerRuntime` (engine-api) so must be in engine-api
- New module `casehub-engine-signal` — overkill for foundation types

**Rationale:** Direct analog of D4 (observation placement). SPI/model types in api, mutable state management in common, handler integration in runtime. `PerceivedSignal` crosses the engine-api boundary (returned by `WorkerRuntime`) so must live in engine-api.

**Trade-offs:** None significant — follows established pattern exactly.

**Sources:** D4 (observation module placement), SPI placement rule in CLAUDE.md, `io.casehub.api.spi.observation` package (observation precedent)
**Depends on:** D10 (registry), D13 (WorkerRuntime API)
**Exploration:** quick
**Status:** captured

## D18: Lifecycle — Case termination eviction + Resettable

**Choice:** `CaseStatusChangedHandler` calls `signalRegistry.evictByCase(caseId)` on terminal case status (COMPLETED, FAULTED, CANCELLED) — same pattern as `ObservationRegistry.unregisterByCase()` and `ContextHistoryBuffer.evict()`. `SignalRegistry implements Resettable` for demo/test replay. No compound-scoped signal cleanup — signals are case-scoped coordination primitives. Expired signals (below effective-zero) are retained with an `expired` flag until case eviction — they don't consume perception bandwidth but remain for audit.

**Alternatives:**
- Compound-scoped cleanup — breaks cross-compound coordination, which is the primary use case for stigmergy
- No eviction, rely on decay — registry grows unboundedly across long-running cases; `maxSignalsPerCase` mitigates but doesn't eliminate for terminated cases

**Rationale:** Case-scoped lifecycle is clean and consistent with all other per-case registries (`ObservationRegistry`, `ContextHistoryBuffer`, `CbrRetrievalService` cache, `CaseRecoveryStateRegistry`). Expired signals are filtered from perception views but retained for `SIGNAL_EXPIRED` audit. Case termination is the natural garbage collection point.

**Trade-offs:** Long-running cases with many expired signals accumulate entries until termination. Bounded by `maxSignalsPerCase` (D15, default 100) — worst case is 100 entries with expired flags per case.

**Sources:** `CaseStatusChangedHandler` (terminal state eviction pattern), `ObservationRegistry.unregisterByCase()`, `ContextHistoryBuffer.evict()`, `Resettable` interface, engine#1106
**Depends on:** D10 (registry storage), D15 (maxSignalsPerCase bound), D16 (expiry audit)
**Exploration:** quick
**Status:** captured

## D19: WorkerRuntime faceting — SignalSpace + InterestSpace

**Choice:** Full restructure of WorkerRuntime coordination surface into domain-organized facets. Two facet interfaces: `SignalSpace` (deposit, perceive — moved from flat methods) and `InterestSpace` (registerInterest, deregisterInterest, myInterests, interestLandscape, registerObserver — new plus moved). Existing flat methods (`depositSignal`, `perceiveSignals`, `registerObserver`) deprecated with delegates to facets. `default` methods on WorkerRuntime return `NOOP` implementations. Future issues add new facets: `NeighborSpace` (#1108), `RuleSpace` (#1109).

**Alternatives:**
- Keep flat WorkerRuntime — simple but god interface by #1112 (20+ methods across perception, communication, coordination, self-awareness, social awareness)
- Single CoordinationSurface facet — simpler top-level but the single facet grows into the same god interface problem
- Flat now, facet later — pragmatic but defers architectural commitment; user chose to commit now

**Rationale:** The Hive Mind epic (#1107-#1115) will add significant coordination surface. Two facets map to two clear domains: signals (shared environment state) and interests (observation management). Each is cohesive. Domain-organized facets avoid premature ISP splitting while preventing the god interface. LLM agents discover methods within a focused facet.

**Trade-offs:** Extra indirection (`runtime.signals().deposit()` vs `runtime.depositSignal()`). Commits to facet boundaries before #1111 (stigmergy) reveals how the domains interact — risk of wrong boundary. Mitigated: facet interfaces can be merged or split later without breaking consumers (add `extends` or move methods).

**Pre-release revision:** Since the project is pre-release, flat methods (`depositSignal`, `perceiveSignals`, `registerObserver`) are REMOVED from WorkerRuntime entirely — no deprecated delegates, no migration bridges. Facets are the only path. All existing call sites updated. Clean break.

**Sources:** `WorkerRuntime.java` (engine-api), `DefaultWorkerRuntime.java` (runtime-core), `LifecycleScope` (api/model), engine#1105 (registerObserver), engine#1106 (depositSignal/perceiveSignals), engine#1108-#1115 (future surface growth)
**Exploration:** quick
**Status:** revised — pre-release: removed deprecated delegates, clean break

## D20: InterestDeclaration sealed hierarchy — five permits with JQ gate

**Choice:** `InterestDeclaration` is a sealed interface with five permits: `KeyThreshold` (key, operator, threshold), `KeyCorrelation` (keys, jqCondition), `TemporalSequence` (steps, window), `SignalThreshold` (signalName, operator, threshold), and `JqInterest` (expression, watchedKeys). Each of the first four maps 1:1 to a classical observer. `JqInterest` is a general-purpose catchall that creates a `CorrelationObserver` internally but with arbitrary JQ logic. `JqInterest` registration is gated by `ObservationConfig.allowJqInterests()` (boolean, default `true`) — throws `IllegalArgumentException` when `false`. Compliance teams in regulated domains (AML, clinical) set `allowJqInterests: false` to restrict agents to the four auditable typed permits. `registerObserver()` on `InterestSpace` remains as the low-level escape hatch for programmatic observers that don't fit any interest type.

**Alternatives:**
- Four permits only — clean 1:1 mapping but forces complex agent logic through the registerObserver escape hatch
- Three permits (merge signal into key threshold) — simpler hierarchy but conflates context keys and signal names
- Four permits without gate — misses the compliance requirement

**Rationale:** Five types give full expressiveness. The four typed permits are fully auditable — you can inspect exactly what an agent watches, what keys, what thresholds. The JQ catchall provides flexibility for unregulated domains. The config gate makes it a per-case-definition policy decision, consistent with `ObservationConfig`'s existing role as the observation policy surface. `registerObserver()` on `InterestSpace` is also gated by `ObservationConfig.allowProgrammaticObservers()` (default `true`). When `false`, `InterestSpace.registerObserver()` throws `IllegalStateException`. This ensures the compliance posture is consistent — either agents are restricted to typed declarations, or they have full flexibility including programmatic observers. The engine-internal `ObservationRegistry.registerObserver()` remains ungated — the gate is at the agent-facing `InterestSpace` boundary, not the engine-internal registry.

**Trade-offs:** JQ gate enforcement is at registration time only — if the config changes after registration, existing JQ observers continue running. Acceptable — config changes don't retroactively invalidate live case behavior. The five-type vocabulary may need extension for future interest patterns (e.g. rate-of-change) — the sealed hierarchy would need a new permit, which is a source-compatible addition.

**Sources:** `ThresholdObserver.java`, `CorrelationObserver.java`, `TemporalSequenceObserver.java`, `SignalStrengthObserver.java` (runtime-core), `ObservationConfig.java` (engine-api), engine#1107
**Depends on:** D19 (faceted architecture — interests live on InterestSpace)
**Exploration:** quick
**Status:** revised — R1-07: added `allowProgrammaticObservers` gate on InterestSpace.registerObserver() to close the compliance escape hatch

## D21: Interest lifecycle — InterestRegistration handle with deregister by ID

**Choice:** `InterestSpace.register(InterestDeclaration)` returns `InterestRegistration(String interestId, InterestDeclaration declaration, Instant registeredAt)` — an immutable value record. `interestId` is engine-generated (follows `ObservationRegistry`'s `observerType-N` pattern). Deregistration via `InterestSpace.deregister(String interestId)`. `InterestSpace.mine()` returns `List<InterestRegistration>`.

**Alternatives:**
- Deregister by declaration equality — ambiguous when same agent registers same interest twice
- Mutable InterestHandle with update()/deregister() — lifecycle coupling between handle and registry

**Rationale:** Immutable records, string-based deregistration, no mutable state. Consistent with platform patterns. Serializable for REINVOKED worker state accumulation.

**Trade-offs:** No in-place update — agent must deregister + register to change an interest. Acceptable for v1.

**Sources:** `ObservationRegistry.java:60` (instanceId pattern), engine#1107
**Depends on:** D19 (InterestSpace facet), D20 (InterestDeclaration types)
**Exploration:** quick
**Status:** captured

## D22: myInterests() returns List<InterestRegistration>

**Choice:** `InterestSpace.mine()` returns `List<InterestRegistration>` — reuses the registration record type. Agent already has the type from `register()`. Simple, consistent.

**Alternatives:**
- List<InterestSummary> with observation stats — requires per-interest stats tracking not yet in the registry
- Map<String, InterestDeclaration> keyed by ID — less natural for iteration

**Rationale:** Reuse existing type. If per-interest stats are needed later, `InterestRegistration` can gain optional fields.

**Trade-offs:** None significant.

**Sources:** engine#1107
**Depends on:** D21 (InterestRegistration type)
**Exploration:** quick
**Status:** captured

## D23: Module placement — facets in api/engine, interest types in api/spi/observation

**Choice:** Pre-release clean design. Facet interfaces (`SignalSpace`, `InterestSpace`) in `io.casehub.api.engine` alongside `WorkerRuntime`. Interest types (`InterestDeclaration`, `InterestRegistration`, `InterestLandscape`) in `io.casehub.api.spi.interest`. Signal types unchanged in `io.casehub.api.model.signal`. Implementations: `DefaultSignalSpace` in `runtime-core/internal/signal/`, `DefaultInterestSpace` in `runtime-core/internal/interest/`. `DefaultWorkerRuntime` creates both facet implementations and returns them from `signals()` and `interests()`.

**Alternatives:**
- Facets in domain packages (SignalSpace in api/model/signal) — mixes behavioral interfaces with value records
- New api/coordination package — more packages than necessary

**Rationale:** Follows codebase convention: `api/engine` = runtime interfaces, `api/spi` = domain SPIs, `api/model` = value types. Facet interfaces are runtime surfaces (same nature as WorkerRuntime). Each domain gets its own SPI package matching its WorkerRuntime facet: `api/spi/observation` (EnvironmentObserver, ObservationContext, Observation, ObservationConfig, ContextSnapshot), `api/spi/interest` (InterestDeclaration, InterestRegistration, InterestLandscape), `api/spi/neighbor` (Neighbor, NeighborRelation), `api/spi/rule` (LocalRule, RuleAction, RuleCondition, RuleContext, RuleFiring, RuleRegistration, RuleConfig). The import graph is meaningful — types imported from `spi.rule` are about rules, not "observation."

**Trade-offs:** `api/engine` package grows from 1 to 5 files (WorkerRuntime, SignalSpace, InterestSpace, NeighborSpace, RuleSpace). Four SPI packages instead of one. Better navigability for a 4-facet coordination layer.

**Sources:** `api/engine/WorkerRuntime.java`, `api/spi/observation/` package, `api/model/signal/` package, engine#1107
**Depends on:** D19 (faceted architecture), D20 (InterestDeclaration types)
**Exploration:** quick
**Status:** revised — R1-08: split api/spi/observation into per-domain packages (observation, interest, neighbor, rule) matching WorkerRuntime facets

## D24: InterestLandscape computation — on-demand from registry

**Choice:** `InterestLandscape` is computed on-demand from `ObservationRegistry` data. `InterestSpace.landscape()` (worker runtime) computes at each call — O(N) where N ≤ `maxObserversPerCase` (20). `ObservationContext.interestLandscape()` (observer evaluation) is computed once per evaluation cycle by the handler and passed as a pre-computed snapshot. No caching in the registry.

**Alternatives:**
- Cached per evaluation cycle in registry — cache invalidation complexity not justified for 20-entry aggregation
- Materialized on registration change — over-engineered for a read that's O(20)

**Rationale:** Cheap computation, bounded input size. Same pattern as `perceive()` on `SignalRegistry`.

**Trade-offs:** Repeated worker calls to `landscape()` within one execution recompute each time. Acceptable — bounded cost, no correctness issue.

**Sources:** `ObservationRegistry.java:89-101` (getObservers iteration pattern), engine#1107
**Depends on:** D19 (InterestSpace facet), D23 (placement)
**Exploration:** quick
**Status:** captured

## D25: ObservationContext gains interestLandscape() — 8th field

**Choice:** `ObservationContext` record gains `InterestLandscape interestLandscape` as the 8th field. Backward-compatible 7-arg and 6-arg constructors pass `InterestLandscape.EMPTY`. Handler computes the landscape once per evaluation cycle and passes into the `ObservationContext` constructor. Consistent with how `signals()` (7th field) was added in #1106.

**Alternatives:** None considered — direct extension of the established pattern.

**Rationale:** Observers can factor in collective attention during evaluation. Same extension pattern used twice before.

**Trade-offs:** Record grows wider. Acceptable — `ObservationContext` is constructed once per cycle, not per observer.

**Sources:** `ObservationContext.java:23-41` (existing 7-arg record), `CaseContextChangedEventHandler.java:1230-1238` (context construction site), engine#1106 (signals() precedent)
**Depends on:** D14 (observation integration), D24 (landscape computation)
**Exploration:** quick
**Status:** captured

## D26: Scope enforcement — same as registerObserver

**Choice:** `InterestSpace.register(InterestDeclaration)` enforces the same scope rule as `registerObserver()`: BINDING scope rejected with `IllegalStateException`, only COMPOUND or CASE scope allowed. Check is in `DefaultInterestSpace.register()` — looks up the binding's `LifecycleScope` from `CaseDefinition` via the same path as `DefaultWorkerRuntime.registerObserver()`.

**Alternatives:** None — interests create observers, same lifecycle constraints apply.

**Rationale:** Interests create observers under the hood. BINDING-scoped workers execute once and disappear — temporal patterns and sustained observation require COMPOUND or CASE scope. Same validation, same error message.

**Trade-offs:** None.

**Sources:** `DefaultWorkerRuntime.java:293-304` (existing scope check), engine#1105 D2 (scope rationale)
**Depends on:** D19 (InterestSpace facet), D20 (InterestDeclaration types)
**Exploration:** quick
**Status:** captured

## D27: Audit — INTEREST_REGISTERED and INTEREST_DEREGISTERED event types

**Choice:** Two new `CaseHubEventType` values: `INTEREST_REGISTERED` (metadata: interestId, interestType, agentId, declaration summary) and `INTEREST_DEREGISTERED` (metadata: interestId, agentId). EventLog publishing deferred to the same wiring pass as `PHEROMONE_DEPOSITED`/`PHEROMONE_EXPIRED` from #1106 — both need event bus plumbing into `DefaultWorkerRuntime` / the facet implementations.

**Alternatives:**
- Publish immediately via injected EventLogRepository — requires persistence dependency in the facet implementation, couples registration to storage
- No audit — loses traceability for compliance

**Rationale:** Consistent with #1106 deferral. The event types exist in the enum for schema completeness; publishing wiring is a cross-cutting concern for all runtime-side events.

**Trade-offs:** Events not published until wiring is added. EventLog metadata schema defined now for forward compat.

**Sources:** `CaseHubEventType.java`, `PHEROMONE_DEPOSITED`/`PHEROMONE_EXPIRED` (deferred audit from #1106), engine#1107
**Depends on:** D19 (InterestSpace facet)
**Exploration:** quick
**Status:** captured

## D28: YAML support — not in scope for v1

**Choice:** No YAML syntax for declaring interests. Interests are runtime-registered by agents via `InterestSpace.register()`. Static declaration of "what to observe" is already handled by `ContextChangeTrigger` on bindings. Dynamic interest registration (#1107) is inherently runtime — agents decide what to observe based on their own state and goals.

**Alternatives:**
- YAML `defaultInterests:` block on CaseDefinition — pre-registers interests at case start. Could be useful but conflates static triggers with dynamic interests
- YAML per-worker interests — pre-wires observation interests per worker definition. Overly prescriptive for self-organization.

**Rationale:** The entire point of #1107 is dynamic registration. YAML is static. If default interests become needed, it's a follow-up.

**Trade-offs:** Agents must programmatically register interests — no declarative shortcut. Acceptable for the self-organization use case.

**Sources:** engine#1107 issue description ("agents register observation interests at runtime"), `ContextChangeTrigger` (existing static trigger mechanism)
**Exploration:** quick
**Status:** captured

## D29: Coordination state persistence — in-memory only

**Choice:** All coordination state is in-memory only: `ObservationRegistry` (`ConcurrentHashMap`), `SignalRegistry` (`ConcurrentHashMap`), `ContextHistoryBuffer` (`ConcurrentHashMap`). On crash, process restart, or rolling deployment, all accumulated coordination intelligence is lost. No persistence layer, no recovery mechanism for coordination state.

**Alternatives:**
- Persist to event store — coordination state reconstructible from event replay; adds latency on writes, complexity in recovery
- Periodic snapshots to CaseContext (e.g., `_coordination.*` keys) — leverages existing persistence but risks feedback loops (per D7) and couples coordination to domain storage
- Dedicated coordination persistence (e.g., embedded RocksDB or Redis) — full recovery at the cost of operational complexity and an additional dependency

**Rationale:** Intentional "start in-memory" design consistent with the platform's current single-instance deployment model. All per-case evaluation state (goals, plan items, bindings) is similarly in-memory during the evaluation lifecycle. Coordination state follows the same model. The `Resettable` interface on all three registries supports demo/test replay. Persistence for coordination state should be addressed holistically when the platform addresses persistence for all in-memory evaluation state — not as a per-registry concern.

**Trade-offs:** Long-running cases (multi-day AML investigations, clinical trials) lose accumulated coordination intelligence on restart. Bounded by the observation that the current platform has no horizontal scaling — in-memory state is consistent because there's one instance. Multi-instance deployment would require a distributed backend for all three registries. The `CaseRecoveryStateRegistry` (which handles other recovery concerns) is also in-memory, confirming this is a platform-level constraint, not a coordination-specific one. **Documentation requirement:** The stigmergy YAML guide must include a prominent "Deployment Considerations" section warning that all coordination state (observations, signals, interests, rules, activity metrics, convergence state, role clusters, team affinities, progress scores) is lost on restart. Multi-day cases that accumulate coordination intelligence must be designed to tolerate this loss — either through rapid signal re-establishment on case resumption, or by persisting critical coordination outcomes as CaseContext keys (which ARE persisted via EventLog).

**Sources:** `ObservationRegistry.java`, `SignalRegistry.java`, `ContextHistoryBuffer.java`, `CaseRecoveryStateRegistry` (all in-memory), `Resettable` interface
**Depends on:** D7 (materialization), D10 (signal storage)
**Exploration:** quick (surfaced by review R1-09)
**Status:** revised — ADR-R1-16: added documentation requirement for deployment hazard warning in stigmergy YAML guide

## D30: Observer deduplication — registry-level replace on re-registration

**Choice:** `ObservationRegistry.registerObserver()` deduplicates by `(caseId, agentId, bindingName, observerType)`. When a registration matches an existing entry on all four keys, the existing observer is replaced (updated) rather than a new one added. This handles COMPOUND-scoped workers that re-register observers on each dispatch — the same agent/binding/type combination replaces rather than accumulates. Workers that need multiple distinct observers of the same type use distinct `observerType()` values.

**Alternatives:**
- No deduplication (current implementation) — COMPOUND workers accumulate observers on each dispatch, exhausting `maxObserversPerCase` after N dispatches. Workers must manually deregister before re-registering, which is undocumented and error-prone.
- Dedup by `(agentId, observerType)` only — too aggressive; different bindings for the same agent should be able to register observers of the same type independently
- Document manual deregistration responsibility — shifts lifecycle burden to the worker implementor; COMPOUND workers would need to call `unregisterByAgent()` at the start of each dispatch

**Rationale:** COMPOUND-scoped workers are the primary observer registrars — they run repeatedly and observe over time. Re-registration (replace semantics) is the natural model: each dispatch refreshes the observer with potentially updated parameters. The four-key dedup preserves the ability to have multiple observers per agent (via different binding names or observer types) while preventing accumulation from repeated dispatch. The `InterestDeclaration` hierarchy (D20) generates unique observer types from declaration parameters, so typed interests naturally dedup correctly.

**Trade-offs:** Workers that intentionally want two observers of the same type from the same binding must use distinct `observerType()` values. This is a constraint, but a well-motivated one — distinct observations should have distinct types for audit clarity.

**Sources:** `ObservationRegistry.java:38-62` (current registration without dedup), `ObservationRegistry.ObserverRegistration` (instanceId generation), D9 (maxObserversPerCase cap)
**Depends on:** D2 (registration mechanism), D9 (cardinality cap)
**Exploration:** quick (surfaced by review R1-10)
**Status:** captured

## D31: Observation replacement semantics — full replace per cycle

**Choice:** Observations produced by observers in cycle N are fully replaced by observations from cycle N+1. `ObservationRegistry.storeObservations()` uses `put()` which overwrites the previous list. There is no accumulation, trending, or confidence building across cycles within the registry. An observation that is present in cycle N and absent in cycle N+1 vanishes.

**Alternatives:**
- Accumulate with decay — observations persist across cycles with diminishing confidence, similar to signal decay. Requires observation-level decay model and increases registry memory proportionally to observation history depth.
- Sliding window — retain observations from the last K cycles. Rules can inspect temporal observation trends. Adds a K×N storage multiplier and complicates the query interface.
- Explicit expiry — observations persist until explicitly removed or a configurable TTL expires. Requires per-observation lifecycle management.

**Rationale:** Observations are instantaneous perception — what the observer detects RIGHT NOW. An observer that detects "suspicious pattern" in cycle N and doesn't detect it in cycle N+1 should not leave a residual trace in the observation registry. Temporal reasoning across cycles is the responsibility of the observer itself (which has access to the history buffer) and of local rules (#1109). The observation SPI produces point-in-time observations; higher-level reasoning (trending, confidence building) happens in the consumption layer, not the production layer. This separation keeps the registry simple and avoids the sub-decisions that accumulation would require (decay model, window size, eviction policy).

**Trade-offs:** Local rules (#1109) cannot react to observations that occurred in a previous cycle unless the observer re-detects them. Combined with the one-cycle delay (D5), this means a pattern must be present for at least two consecutive cycles to be both observed and acted upon. Acceptable — transient patterns that appear for exactly one cycle are noise, not signal.

**Sources:** `ObservationRegistry.storeObservations()` (put semantics), D5 (one-cycle delay), D7 (observation materialization)
**Depends on:** D7 (materialization), D5 (pipeline integration)
**Exploration:** quick (surfaced by review R1-15)
**Status:** captured

## D32: NeighborSpace architecture — query facade over existing engine data

**Choice:** `NeighborSpace` is a pure read-only query facade that computes neighbor awareness on-demand from existing engine registries (PlanItemStore, ScopedWorkerRegistry, ObservationRegistry, SignalRegistry). No new storage system. Proximity is emergent from shared activity: agents watching the same keys are observationally proximate, agents depositing the same signals are coordinationally proximate, agents executing on the same case are coactive, agents whose outputs feed another's inputs are complementary. Third WorkerRuntime facet: `default NeighborSpace neighbors() { return NeighborSpace.NOOP; }`. `DefaultNeighborSpace` (runtime-core) queries existing registries. Eidos can provide `EnhancedNeighborSpace` (capability-space proximity scoring) via the same facet interface when on the classpath.

**Alternatives:**
- Activity-only (no proximity) — agents only see who is on their case, no interest/signal overlap analysis
- Full in engine (including capability-space math) — duplicates eidos vector similarity computation
- SPI-only (no implementation) — delivers no working behavior until eidos provides implementation

**Rationale:** The engine already has all the data needed for activity-based neighbor discovery. A query facade avoids new storage while providing useful neighbor awareness immediately. Emergent proximity (from shared activity) is more actionable than abstract capability similarity — it captures what agents are actually doing, not what they're declared capable of. The facet pattern from D19 makes the upgrade path to eidos-enriched proximity transparent.

**Trade-offs:** No capability-space vector similarity until eidos integration. Emergent proximity may miss agents that are capability-similar but not yet active on the case. Acceptable — active agents are the ones you can coordinate with.

**Sources:** `PlanItemStore.java` (common-core), `ScopedWorkerRegistry.java` (common-core), `ObservationRegistry.java` (common-core), `SignalRegistry.java` (common-core), `WorkerRuntime.java` (api/engine), D19 (faceted architecture), engine#1108, arXiv:2504.00587 (AgentNet dynamic topology)
**Exploration:** quick
**Status:** captured

## D33: Neighbor data model — identity + capabilities + status + relations

**Choice:** `Neighbor(String agentId, Set<String> capabilities, TaskStatus currentStatus, String bindingName, Set<NeighborRelation> relations)` — bounded view of a neighboring agent. `NeighborRelation` enum: `COACTIVE` (active on same case), `SHARED_INTEREST` (watching ≥1 same key), `SHARED_SIGNAL` (depositing same signal), `COMPLEMENTARY` (my outputs feed their observations or vice versa). A neighbor can have multiple relations. `agentId` is the worker name — full identity exposed because agents need to know WHO to coordinate with (deposit targeted signals, register complementary interests, avoid duplicate work).

**Alternatives:**
- Anonymous with correlation ID — harder to use for direct coordination
- Configurable visibility (FULL/ANONYMOUS per case) — added complexity for an unclear use case at this stage

**Rationale:** InterestLandscape (#1107 D24) is anonymous because it answers "what is the collective watching?" (aggregate). NeighborSpace answers "who is near me?" (individual) — you can't coordinate with an anonymous aggregate. Identity is necessary for self-organization.

**Trade-offs:** Full identity exposure means agents can make decisions based on specific other agents' identities (identity coupling). Mitigated: agents should coordinate via signals and interests (stigmergy), not by hardcoding agent-specific logic.

**Sources:** `InterestLandscape.java` (anonymous aggregate precedent), `Worker.java` (worker-api, name field), D24 (InterestLandscape anonymity rationale), engine#1108
**Depends on:** D32 (NeighborSpace architecture)
**Exploration:** quick
**Status:** captured

## D34: NeighborSpace query API — four named methods

**Choice:** Four focused query methods on `NeighborSpace`: `active()` (coactive neighbors on same case), `withSharedInterests()` (agents watching ≥1 same key), `withSharedSignals()` (agents depositing same signals), `complementary()` (agents whose outputs feed my inputs or vice versa). Each returns `List<Neighbor>`. Named methods are discoverable by LLM agents. YAGNI — add query composition (NeighborQuery builder) when #1111/#1112 demand it.

**Alternatives:**
- Single `discover(NeighborQuery)` with builder — more flexible but adds complexity for v1
- Unified `all(NeighborRelation...)` with relation filter — composable but less self-documenting

**Rationale:** Pre-release means we can refactor freely when the swarm issues (#1111-#1112) reveal more complex query needs. Four named methods cover the four relation types cleanly. LLM agents work better with explicit method names than builder patterns.

**Trade-offs:** No ad-hoc query composition. If an agent wants "neighbors that are both coactive AND have shared interests," it must call both methods and intersect. Acceptable for v1.

**Sources:** `InterestSpace.java` (named method pattern), D32 (NeighborSpace architecture), engine#1108
**Depends on:** D32 (NeighborSpace architecture), D33 (Neighbor data model)
**Exploration:** quick
**Status:** captured

## D35: SignalRegistry source tracking — Set<String> sources on Signal

**Choice:** `Signal` gains `Set<String> sources` — immutable set of all agent IDs that have deposited or reinforced the signal. `deposit()` adds the depositor to the set (in addition to updating `lastSource` for backward compat). `sources` enables the "shared signal neighbors" query: find signals where `sources` contains both the calling agent and another agent. `Set.copyOf()` on read for immutability.

**Alternatives:**
- Use `lastSource` only — loses multi-depositor information, can't compute signal-based proximity
- Append-only depositor log with timestamps — more detailed but unbounded storage per signal

**Rationale:** Signals are the primary coordination mechanism in stigmergy. Knowing which agents are depositing the same signals is high-value proximity data. The set is bounded by the number of agents on a case (small). `lastSource` stays for audit (most recent depositor) while `sources` captures the full depositor set.

**Trade-offs:** Minor memory increase per Signal (Set<String> vs single String). Bounded by per-case agent count. Acceptable.

**Sources:** `Signal.java` (api/model/signal), `SignalRegistry.java` (common-core), D12 (signal identity), D32 (NeighborSpace data sources), engine#1108
**Depends on:** D32 (NeighborSpace architecture — signal-based proximity needs source tracking)
**Exploration:** quick
**Status:** captured

## D36: Module placement — NeighborSpace follows D23 pattern

**Choice:** `NeighborSpace` in `io.casehub.api.engine` (alongside `SignalSpace`, `InterestSpace`, `RuleSpace`). `Neighbor` and `NeighborRelation` in `io.casehub.api.spi.neighbor`. `DefaultNeighborSpace` in `runtime-core` at `io.casehub.engine.internal.neighbor`. Follows D23 per-domain package split.

**Alternatives:** None — follows revised D23 pattern.

**Rationale:** Facet interfaces are runtime surfaces. Neighbor types are their own domain — neighbor awareness is not observation. Each facet's SPI types live in a matching package.

**Trade-offs:** None significant.

**Sources:** D23 (module placement pattern), D19 (faceted architecture), engine#1108
**Depends on:** D32 (NeighborSpace architecture), D33 (Neighbor data model)
**Exploration:** quick
**Status:** revised — R1-08: Neighbor types moved to api/spi/neighbor (from api/spi/observation)

## D37: Rule action scope — coordination + context writes

**Choice:** Approach B — Rules can perform coordination-only actions (deposit signals, register/deregister interests, emit conclusions) AND write specific keys to the CaseContext working layer. Context writes bridge coordination state back into domain state, triggering `CONTEXT_CHANGED` and enabling binding dispatch. Coordination-only actions are best practice for most use cases; context writes are available when rules need to influence case progression directly.

**Alternatives:**
- Coordination only — rules stay entirely in the coordination layer; binding dispatch driven exclusively by external context changes. Simpler but breaks the stigmergy loop: rules can detect coordination patterns but can't cause the case to act without an external context change arriving.
- Full dispatch — rules can request worker scheduling directly, creating a second dispatch path alongside binding triggers. Over-powered and architecturally complex.

**Rationale:** The stigmergy loop requires perceive→decide→act→modify environment. If rules can only deposit signals (coordination-only), the "act" step never reaches domain state — signals are invisible to binding conditions. Context writes close the loop: rule detects a coordination pattern → writes a key to working layer → `CONTEXT_CHANGED` fires → binding with a matching `when` condition dispatches a worker. This is indirect coordination through environment modification — the definition of stigmergy. Context writes are applied after all rules have been evaluated (batched), with a single `CONTEXT_CHANGED` published post-evaluation to avoid re-entrant evaluation within the serializer gate.

**Trade-offs:** Context writes create a path from coordination state to domain state, which means rules can indirectly cause binding dispatch. This risks feedback loops (rule writes → binding fires → worker runs → context changes → rule fires → ...). Mitigated by: (1) one-shot evaluation per cycle (rules evaluate once, no intra-cycle chaining), (2) batched writes applied after all rules complete, (3) bindings can guard against re-triggering with `when` conditions that check for rule-written keys, (4) semantic identity check on batched writes — `CONTEXT_CHANGED` is suppressed when all writes produce values identical to what is already in the context (idempotent writes don't trigger re-evaluation), (5) D48 budget enforcement provides a hard backstop against unbounded cycles. Best practice guidance: use coordination-only actions by default, context writes only when the coordination pattern needs to influence case progression.

**Sources:** D7 (observation materialization — no CaseContext writes), D5 (pipeline integration), D10 (signal storage — not in CaseContext), engine#1109, engine#1111 (stigmergy requires environment modification), SwarmSys (arXiv:2510.10047)
**Depends on:** D5 (pipeline integration), D10 (signal storage), D19 (faceted architecture), D48 (budget enforcement — hard backstop against unbounded context-write cycles)
**Exploration:** quick
**Status:** revised — R1-03: added semantic identity check for batched writes to prevent idempotent-write feedback loops; added hard dependency on D48

## D38: Condition model — dual (expression + lambda)

**Choice:** Rule conditions support two models: `ExpressionEvaluator` conditions (JQ/MVEL against a combined JSON view of coordination state — observations, signals, neighbors, context) and `Predicate<RuleContext>` lambdas (Java DSL, full type safety). YAML-declared rules use expression-based conditions. Java DSL can use either. The combined JSON view for expression evaluation assembles observations, perceived signals, active neighbors, interest landscape, context snapshot, and changed keys into a single JSON document.

**Alternatives:**
- Expression-only — all conditions are ExpressionEvaluator instances. Lambda conditions wrapped via LambdaExpressionEvaluator lose type safety and become opaque.
- Predicate-only — all conditions are lambdas. No YAML support, no auditability.

**Rationale:** Dual model covers both needs: auditable expression conditions for compliance domains (AML, clinical) where you need to inspect exactly what a rule evaluates, and flexible lambdas for complex agent logic that doesn't fit expression syntax. Expression conditions are serializable (storable in EventLog, reconstructible from YAML). Lambda conditions are runtime-only. The existing `ExpressionEngineRegistry` infrastructure supports the expression path without new evaluation machinery.

**Trade-offs:** Two condition code paths. Mitigated: both converge to a boolean result; the evaluation pipeline dispatches based on condition type, like the existing binding trigger evaluation.

**Sources:** `ExpressionEngine.java:36`, `ExpressionEngineRegistry` (existing infrastructure), ADR-0009 (per-expression override), D20 (InterestDeclaration sealed hierarchy — precedent for typed + catch-all), engine#1109
**Depends on:** D37 (action scope — conditions must evaluate against coordination state that includes what actions can target)
**Exploration:** quick
**Status:** captured

## D39: Rule model and firing semantics — per-agent, all-fire, one-shot

**Choice:** Per-agent rule evaluation with all-matching-fire semantics. `LocalRule(String id, RuleCondition condition, List<RuleAction> actions, int priority)`. Each agent's rules are evaluated independently — no cross-agent rule interaction. ALL matching rules fire per cycle (priority determines execution order, not selection). One-shot per cycle — no intra-cycle chaining. Refraction is implicit via the one-cycle delay (D5). This differs from Drools' match-resolve-act model where conflict resolution selects one rule from the conflict set. In swarm systems, conflict resolution happens at the environment level (signal reinforcement/decay), not at the rule engine level.

**Alternatives:**
- Match-resolve-act (Drools model) — conflict resolution selects highest-priority matching rule, only one fires per cycle. More controlled but fights swarm semantics where multiple simultaneous behaviors are desirable.
- Rule chaining within a cycle — rules fire, modify state, re-evaluate. Powerful but risks infinite loops and violates the one-cycle delay principle from D5.

**Rationale:** Swarm agents follow multiple behavioral rules simultaneously (forage AND avoid danger AND follow pheromone gradient). All-fire matches this biological model. Priority determines execution order: coordination actions (signal deposits, interest changes) execute in priority order (highest first). Context writes are deduplicated per key — when multiple rules write the same key, only the highest-priority rule's write is applied (lower-priority writes to the same key are discarded). This is unambiguous and doesn't depend on write ordering semantics. Future Drools integration can provide a `RuleEvaluationStrategy` that replaces all-fire with match-resolve-act for domains that need it.

**Trade-offs:** Multiple rules writing the same context key within a single agent: highest-priority-wins via per-key dedup. Cross-agent write conflicts for the same key: last-writer-wins with undefined agent ordering (D42). Agents sharing keys must coordinate via signals to avoid conflicting writes.

**Sources:** SwarmSys (arXiv:2510.10047 — multiple simultaneous roles), D5 (one-cycle delay), engine#1109, engine#445 (Drools — different model for different purpose)
**Depends on:** D37 (action scope), D38 (condition model)
**Exploration:** quick
**Status:** revised — R1-07: context writes use priority-based ordering; ADR-R1-10: changed from ordering-dependent to per-key dedup — highest-priority rule's write per key wins, no ordering ambiguity

## D40: RuleContext — coordination fact space

**Choice:** `RuleContext` record carries the agent's full local perception: `observations` (List<Observation>, this agent's observations from current cycle), `signals` (Map<String, PerceivedSignal>, above threshold), `contextSnapshot` (JsonNode, working layer), `changedKeys` (Set<String>), `landscape` (InterestLandscape), `agentId`, `tenancyId`, `caseId`. For expression-based conditions (D38), assembled into a combined JSON document with top-level keys `observations`, `signals`, `context`, `changedKeys`, `landscape`. Neighbors deliberately excluded from the automatic context — the four NeighborSpace queries have different semantics and cost; agents needing neighbor data use lambda conditions with explicit calls.

**Alternatives:**
- Include all four neighbor views — expensive to compute for every rule evaluation cycle when most rules don't use neighbor data.
- Minimal context (observations + signals only) — insufficient for rules that need to condition on domain state (e.g., "if observation X AND context.status == 'active'").

**Rationale:** The RuleContext mirrors what an agent can perceive: observations (what patterns were detected), signals (what coordination state exists), context (what domain state exists), landscape (what others are watching), and changed keys (what just happened). This is exactly the swarm agent's local perception. Neighbors are a pull model (query when needed) not a push model (always computed).

**Trade-offs:** Lambda conditions needing neighbor data must inject NeighborSpace or use captured references. Acceptable — neighbor-dependent rules are a minority case.

**Sources:** `ObservationContext.java` (8-field record precedent), D14 (signals in ObservationContext), D25 (interestLandscape in ObservationContext), engine#1109
**Depends on:** D38 (condition model — determines how RuleContext is consumed), D32 (NeighborSpace — excluded from automatic context)
**Exploration:** quick
**Status:** captured

## D41: RuleAction sealed hierarchy — four permits

**Choice:** `RuleAction` is a sealed interface with four permits: `DepositSignal(String name, double strength, @Nullable Duration halfLife)`, `RegisterInterest(InterestDeclaration declaration)`, `DeregisterInterest(String interestId)`, `WriteContext(String key, JsonNode value)`. Each action type maps directly to an existing engine operation: signal deposit → `SignalRegistry.deposit()`, interest registration/deregistration → `ObservationRegistry`, context write → `WritableLayer.set()`. `RegisterInterest` and `DeregisterInterest` throw `UnsupportedOperationException` until interest changes from rule actions are wired into the handler context — explicit failure over silent no-op. `RuleCondition` is also sealed: `ExpressionCondition(ExpressionEvaluator evaluator)` | `PredicateCondition(Predicate<RuleContext> predicate)`. `ExpressionCondition` throws `UnsupportedOperationException` until expression wiring is complete — explicit failure over silent `false`.

**Alternatives:**
- Add EmitConclusion(type, data) as a fifth action — structured output for other agents. Overlaps with signals (inter-agent communication) and context writes (structured data). Deferred until a concrete use case distinguishes conclusions from signals.
- Add RequestDispatch(capabilityName) — directly request worker scheduling. Rejected in D37 — context writes bridge to dispatch indirectly via binding triggers.

**Rationale:** Four actions cover the stigmergy loop: deposit signals (announce findings), register interests (adapt perception), deregister interests (stop watching), write context (influence case progression). Each action is auditable (data record, not lambda). Each maps to an existing operation with established semantics. Extensible: new sealed permits can be added when new coordination primitives emerge.

**Trade-offs:** No "remove context key" action. An agent wanting to clear a key writes `NullNode.instance`. No "modify signal" action (e.g., reduce strength) — agents can only deposit (which reinforces). Signal reduction happens via natural decay. Both are intentional simplifications.

**Sources:** `SignalRegistry.deposit()`, `ObservationRegistry.registerObserver()`, `WritableLayerImpl.set()`, D37 (action scope), D20 (InterestDeclaration sealed hierarchy — precedent)
**Depends on:** D37 (action scope — WriteContext enabled by option 2), D38 (condition model — RuleCondition sealed)
**Exploration:** quick
**Status:** revised — R1-05: RegisterInterest/DeregisterInterest throw UnsupportedOperationException instead of silent no-op; ExpressionCondition throws instead of returning false

## D42: Pipeline integration — after observations, batched context writes

**Choice:** `localRules()` runs after `observations()` in `CaseContextChangedEventHandler.evaluateAndDispatch()`. Evaluation order: `rules()` → `goals()` → `observations()` → `localRules()`. Within `localRules()`: (1) build `RuleContext` per agent from registries, (2) evaluate all agents' rules concurrently — each agent's rules submitted to a virtual thread via `ExecutorService.submit(Callable)`, collected with `future.get(ruleEvaluationTimeoutMs, MILLISECONDS)`, and on `TimeoutException` cancelled via `future.cancel(true)` to interrupt the virtual thread (same pattern as D6 observer timeout — `Future.cancel(true)` over `CompletableFuture.orTimeout()` to ensure actual thread interruption), (3) collect all `RuleAction`s, (4) execute coordination actions immediately (signal deposits, interest changes), (5) batch all `WriteContext` actions, (6) apply batched writes in a single pass after all rules complete, with semantic identity check — each write is compared against the current context value and only applied if different, (7) if any writes actually changed a value, publish one `CONTEXT_CHANGED` event (queued by `CaseEvaluationSerializer.drainPending()` for the next evaluation cycle). Idempotent writes are suppressed — if no value actually changes, no `CONTEXT_CHANGED` fires.

**Alternatives:**
- Before observations — rules wouldn't have access to current-cycle observations. Breaks the observe→decide→act pipeline.
- Parallel with observations — race conditions between observation storage and rule reads.
- Context writes applied immediately per agent — ordering between agents becomes significant, harder to reason about.

**Rationale:** After observations ensures rules see current-cycle observations. Batched writes prevent inter-agent ordering effects and ensure a single clean `CONTEXT_CHANGED`. The serializer's `drainPending()` naturally handles the re-evaluation cycle — no special plumbing needed. Virtual thread dispatch with `Future.cancel(true)` timeout follows the established D6 observer pattern — `ExecutorService.submit()` with explicit cancellation on timeout to interrupt pathological rule lambdas.

**Trade-offs:** All context writes from all agents are batched. Cross-agent writes to the same key use deterministic priority-based dedup: the write from the highest-priority rule wins. When priorities are equal across agents, lexicographic agentId is the tiebreaker. This extends D39's intra-agent per-key dedup to the cross-agent case, making outcomes fully reproducible. When a cross-agent write conflict is detected (multiple agents writing the same key), a `CONTEXT_WRITE_CONFLICT` `CaseHubEventType` is emitted (log WARN + EventLog entry, metadata: `key`, `conflictingAgents`, `winningAgent`, `winningValue`). This makes conflicts visible for debugging without making them an error — conflicting writes are a signal to coordinate via signals, not a hard failure. The semantic identity check prevents idempotent-write feedback loops (a rule that always writes the same value won't cause infinite re-evaluation cycles).

**Sources:** `CaseContextChangedEventHandler.java:243-253` (existing pipeline), `CaseEvaluationSerializer.java:35-55` (drainPending), D5 (pipeline integration), D6 (observer thread model)
**Depends on:** D37 (context writes), D39 (one-shot semantics), D41 (action types)
**Exploration:** quick
**Status:** revised — R1-03/R1-04: parallel per-agent evaluation; semantic identity check on batched writes to prevent feedback loops; R1-01: deterministic cross-agent write dedup (highest-priority-wins, agentId tiebreaker) + CONTEXT_WRITE_CONFLICT event; R2-02: aligned timeout mechanism with D6 — Future.cancel(true) over CompletableFuture.orTimeout()

## D43: RuleSpace facet and RuleRegistry

**Choice:** `RuleSpace` is the 4th WorkerRuntime facet. Interface: `register(LocalRule) → RuleRegistration`, `deregister(String ruleId)`, `mine() → List<RuleRegistration>`, `lastFired() → List<RuleFiring>`. `RuleRegistration` record: `(String ruleId, LocalRule rule, Instant registeredAt)`. `RuleFiring` record: `(String ruleId, List<RuleAction> executedActions, Instant firedAt)`. `RuleRegistry` (`common-core`, `@ApplicationScoped`, `Resettable`) stores per-case, per-agent rules and per-cycle firing results. `ConcurrentHashMap<UUID, Map<String, List<LocalRule>>>` for rules (keyed by caseId → agentId → rules), `ConcurrentHashMap<UUID, Map<String, List<RuleFiring>>>` for firings (replaced per cycle, like `ObservationRegistry.storeObservations()`). Scope enforcement same as observers: BINDING rejected, COMPOUND/CASE only. Deduplication by `(caseId, agentId, bindingName, ruleId)`. `DefaultRuleSpace` in `runtime-core/internal/observation/`, wired via `WorkerRuntimeFactory`.

**Alternatives:** None — direct extension of the established facet pattern (D19, D23, D36).

**Rationale:** Follows the InterestSpace/SignalSpace/NeighborSpace pattern exactly. `lastFired()` gives agents visibility into their own rule behavior for adaptive decision-making. Per-cycle firing replacement matches `ObservationRegistry.storeObservations()` semantics.

**Sources:** `InterestSpace.java` (facet pattern), `ObservationRegistry.java` (registry pattern), `WorkerRuntimeFactory.java` (wiring pattern), D19 (faceted architecture), D23 (module placement)
**Depends on:** D39 (rule model), D41 (action types)
**Exploration:** quick
**Status:** captured

## D44: Configuration, audit, lifecycle, module placement

**Choice:** `RuleConfig` record on `CaseDefinition`: `maxRulesPerCase` (default 50), `maxActionsPerCycle` (default 100), `ruleEvaluationTimeoutMs` (default 100). YAML: `ruleConfig:` block under `spec:`. Audit: `CaseHubEventType.RULE_FIRED` (metadata: `agentId`, `ruleId`, `actions[]`, `priority`) and `RULE_REGISTERED` (metadata: `agentId`, `ruleId`, `conditionType`). Lifecycle: `CaseStatusChangedHandler` calls `ruleRegistry.evictByCase()` on terminal status. `ScopedWorkerTerminationHandler` calls `ruleRegistry.unregisterByBinding()` on `COMPOUND_COMPLETED`. Module placement follows revised D23: `RuleSpace` in `api/engine/`, rule types (`LocalRule`, `RuleAction`, `RuleCondition`, `RuleContext`, `RuleFiring`, `RuleConfig`, `RuleRegistration`) in `api/spi/rule/`, `RuleRegistry` in `common-core/internal/rule/`, `DefaultRuleSpace` in `runtime-core/internal/rule/`. No YAML rule declaration in v1 — rules are runtime-registered by agents via `RuleSpace.register()`. Static YAML rules are a natural extension for v2.

**Alternatives:** None — follows established patterns exactly.

**Rationale:** Configuration pattern matches `ObservationConfig` and `SignalConfig`. Audit pattern matches `OBSERVER_REGISTERED`/`OBSERVATION_DETECTED` and `PHEROMONE_DEPOSITED`. Lifecycle pattern matches all other per-case registries. Module placement follows per-domain package convention.

**Sources:** `ObservationConfig.java`, `SignalConfig.java`, `CaseHubEventType.java`, `CaseStatusChangedHandler.java`, D23 (module placement), D36 (NeighborSpace placement)
**Depends on:** D39-D43
**Exploration:** quick
**Status:** revised — R1-08: rule types moved to api/spi/rule (from api/spi/observation)

## D45: Pipeline integration — convergence detection as 5th phase

**Choice:** Add `convergenceDetection()` as a 5th phase in `CaseContextChangedEventHandler.evaluateAndDispatch()`, running after `localRules()`. Order: `rules()` → `goals()` → `observations()` → `localRules()` → `convergenceDetection()`. The phase has access to all coordination state from the current cycle: activity metrics, signal state, observation results, plan item progress. Runs inside the `CaseEvaluationSerializer` gate — serialized per case, consistent with all other phases.

**Alternatives:**
- Inside `observations()` as engine-registered EnvironmentObservers — conflates agent perception with system-level monitoring. Observations are per-agent; convergence is per-case.
- Separate event-driven path outside the serializer gate — loses per-cycle consistency guarantee. Convergence detection that races with evaluation can produce false positives.

**Rationale:** Convergence detection is a system-level concern that observes the *aggregate* behavior of all agents and coordination state. It needs a complete picture of the current cycle — signal deposits, rule firings, context mutations, plan item progress — before making a decision. Running last in the pipeline ensures this. The phase fires synthetic GOAL_REACHED events or BUDGET_EXHAUSTED events, which integrate with existing handlers without a new termination path.

**Trade-offs:** Adds latency to the evaluation cycle. Bounded — convergence detection is O(M) where M = number of tracked metrics (small, constant). No LLM calls, no external I/O.

**Sources:** `CaseContextChangedEventHandler.java:246-257` (evaluateAndDispatch), `CaseEvaluationSerializer.java:35` (per-case gate), D5 (pipeline integration pattern), D42 (localRules as 4th phase)
**Exploration:** quick
**Status:** captured

## D46: ActivityTracker — per-case cumulative metrics with sliding window rates

**Choice:** New `ActivityTracker` (`common-core`, `@ApplicationScoped`, `Resettable`) tracks per-case cumulative counts and sliding-window rates for four core metrics: `totalDispatches` (worker schedule events), `totalSignalDeposits`, `totalContextMutations` (context keys changed per cycle), `totalEvaluationCycles`. Each metric also has a sliding-window rate computed from event timestamps in a bounded circular buffer. Window size configurable via `ConvergenceConfig` on `CaseDefinition` (default 60 seconds). Storage: `ConcurrentHashMap<UUID, CaseActivityState>` where `CaseActivityState` holds four `AtomicLong` counters and four `SlidingWindowCounter` instances.

**Alternatives:**
- Cumulative counts only (no rates) — sufficient for budget enforcement but insufficient for convergence rate detection. Rates are needed to detect "activity has slowed down."
- Exponential moving average — O(1) memory but alpha tuning is non-intuitive and EMA reacts slowly to sudden changes. Sliding window is more precise for bursty swarm patterns.
- Per-cycle delta counting — simpler but couples rate to evaluation frequency. Wall-clock sliding window is more stable across varying evaluation rates.

**Rationale:** Four metrics cover the four resource dimensions where swarm pathology manifests: dispatches (agent thrashing), signals (coordination storms), context mutations (state thrashing), evaluations (evaluation re-entrant loops). Sliding-window rates give precise activity trends for convergence detection. Cumulative totals give hard budget enforcement. `SlidingWindowCounter` is a bounded circular buffer of timestamps — `record(Instant)` appends, `rate(windowDuration, now)` counts entries within the window and returns count/windowSeconds. Memory bound: `maxWindowEntries` is derived from `rateWindow` — cap = `rateWindow.toSeconds() * 10` (allows up to 10 events/second before oldest entries are evicted). For the default 60s window, cap = 600. Configurable via `ConvergenceConfig.maxWindowEntries` (nullable, null = derived). Eviction on case termination via `CaseStatusChangedHandler`, same pattern as all other per-case registries.

**Trade-offs:** Four sliding windows per case × 600 entries each (default) = 2400 timestamps per case. Bounded and manageable. At extreme event rates (>10/s), oldest entries are evicted and rate computation underestimates — acceptable since high event rates are by definition not converged. No persistence — lost on restart (consistent with D29, all coordination state is in-memory).

**Sources:** `QuiescenceTracker.java` (per-case atomic state pattern), `SignalRegistry.java` (per-case ConcurrentHashMap pattern), `Resettable` interface, D29 (in-memory only)
**Depends on:** D45 (pipeline integration — convergence phase reads rates)
**Exploration:** quick
**Status:** revised — made SlidingWindowCounter cap explicit (derived from rateWindow, configurable override) per review R2-01

## D47: Instrumentation points — where metrics are recorded

**Choice:** Each metric is recorded at its single canonical source:
- `totalDispatches` — incremented in `CaseContextChangedEventHandler` when a `WorkerScheduleEvent` is published (inside `publishWorkerSchedule()`, after successful dispatch)
- `totalSignalDeposits` — incremented inside `SignalRegistry.deposit()` itself (single source of truth for ALL signal deposits — worker-initiated via `DefaultSignalSpace`, rule-initiated via `LocalRuleEvaluator`, and any future deposit path). `ActivityTracker` injected into `SignalRegistry`.
- `totalContextMutations` — incremented in `CaseContextChangedEventHandler` at cycle start, counting `event.changedKeys().size()` (or a count of keys changed in the context diff)
- `totalEvaluationCycles` — incremented at the top of `evaluateAndDispatch()` (one per serialized evaluation)

All increments are fire-and-forget — no return values, no blocking. `ActivityTracker` is injected into `CaseContextChangedEventHandler` (for evaluations, dispatches, mutations) and `SignalRegistry` (for signal deposits).

**Alternatives:**
- EventLog-based counting (query EventLog for WORKER_SCHEDULED count) — accurate but O(N) query on each evaluation cycle. Too expensive for a per-cycle check.
- CDI event observers on existing events — decouples instrumentation from handlers but adds async overhead and loses per-cycle consistency.
- Multiple instrumentation points for signals (both `observations()` and `DefaultSignalSpace`) — risks double-counting. Single source at `SignalRegistry.deposit()` is correct.

**Rationale:** Direct instrumentation at the single canonical event source is the most accurate and lowest-overhead approach. For signals specifically, `SignalRegistry.deposit()` is the only method that actually creates or reinforces signals — instrumenting there catches all deposit paths without double-counting risk.

**Trade-offs:** Couples ActivityTracker to CaseContextChangedEventHandler and SignalRegistry. Acceptable — these are the canonical event sources for these metrics.

**Sources:** `CaseContextChangedEventHandler.java:publishWorkerSchedule()`, `CaseContextChangedEventHandler.java:evaluateAndDispatch()`, `SignalRegistry.java:deposit()`
**Depends on:** D46 (ActivityTracker defines what is tracked)
**Exploration:** quick
**Status:** revised — moved signal deposit instrumentation to SignalRegistry.deposit() as single source of truth per review R2-02

## D48: Budget enforcement — hard gate with case fault

**Choice:** Budget enforcement is a hard gate checked at two points: (1) at the top of `evaluateAndDispatch()` for evaluation cycle budget, and (2) inside dispatch/deposit operations for their respective budgets. When any cumulative count exceeds its configured budget cap (`ConvergenceConfig.maxDispatches`, `maxSignalDeposits`, `maxContextMutations`, `maxEvaluationCycles`), the engine: (a) fires a `BUDGET_EXHAUSTED` CaseHubEventType with metadata identifying which budget was exceeded, (b) dispatches `CaseStatusChanged(FAULTED)` with reason "Budget exhausted: <metric>". Budget caps are nullable on `ConvergenceConfig` — null means no limit (backward compatible, no enforcement for cases without convergence config).

**Alternatives:**
- Advisory monitoring + alert — doesn't prevent runaway. The whole point of budget caps is to be a fail-safe.
- Soft cap with escalation (warn at 80%, fault at 100%) — adds configuration complexity. Warning can be implemented separately as a convergence observation without coupling to the enforcement mechanism. **Update (ADR-R1-11):** A `BUDGET_WARNING` event is now included — fires when any metric crosses a configurable warning threshold (default 80% of budget cap). This is a simple event emission, not a dual-threshold enforcement mechanism. `BudgetConfig` gains `Double warningThreshold` (nullable, default 0.8). When non-null and a cumulative count exceeds `cap × warningThreshold`, a `BUDGET_WARNING` `CaseHubEventType` fires (once per metric per case, not repeated). Local rules can condition on this event to throttle behavior before the hard fault.

**Rationale:** Hard gate prevents unbounded resource consumption, which is the #1 production failure mode for multi-agent systems (40% of pilots fail from coordination overhead). Faulting the case is the correct response — it surfaces the problem clearly and triggers the existing failure handling pipeline (CaseOutcomeObserver, EventLog audit, etc.). Null caps preserve backward compatibility — existing cases without convergence config are unaffected.

**Trade-offs:** Hard fault is not graceful — running workers are not proactively cancelled (they complete naturally and find the case already terminal). Acceptable — `CaseStatusChangedHandler` handles cleanup. A case author who wants a warning gate can use a local rule that reads the activity metrics and reacts. Budget enforcement precision: context mutation budget is checked at cycle start, but `localRules()` can write new context keys later in the same cycle. This means the mutation count can overshoot the budget by one cycle's worth of rule writes before the next cycle's check catches it. Accepted imprecision — budget caps are order-of-magnitude safety nets (e.g. max 10,000 mutations), not precise limits. One cycle of overshoot is negligible.

**Sources:** `CaseStatusChanged` event, `CaseStatusChangedHandler.java` (terminal state handling), `maxConcurrentDispatches` (existing hard cap pattern), engine#1044 (WatchdogRecoveryBridge CANCEL_AFFECTED pattern)
**Depends on:** D46 (ActivityTracker provides counts), D47 (instrumentation provides the counts)
**Exploration:** quick
**Status:** revised — acknowledged one-cycle overshoot imprecision for context mutation budget per review R2-03; ADR-R1-11: added BUDGET_WARNING event at configurable threshold (default 80%)

## D49: ConvergenceDetector — activity quiescence with sustained stability

**Choice:** `ConvergenceDetector` (`runtime-core`, `@ApplicationScoped`) evaluates convergence during the 5th pipeline phase. Convergence condition: ALL four activity rates (dispatch, signal deposit, context mutation, evaluation) are below their respective thresholds simultaneously for a sustained duration (`stabilityWindow`). Per-case state tracks: `firstQuietCycle` (Instant when all rates first dropped below threshold, null when any rate exceeds), `consecutiveQuietCycles` (int). When `Duration.between(firstQuietCycle, now) >= stabilityWindow` → convergence detected. On detection: fires synthetic `GoalReachedEvent` with goal name `"_converged"` (engine-reserved, prefixed with `_`). Resets `firstQuietCycle` to prevent repeated firing (one convergence event per case lifetime).

**Alternatives:**
- Weighted composite score — harder to debug. "Which rate caused convergence?" is a common diagnostic question. Threshold-per-metric is directly inspectable.
- Configurable expression (JQ/predicate) — maximum flexibility but opaque. Convergence is a well-defined concept — thresholds + sustained duration cover it.

**Rationale:** "Everything has quieted down for long enough" is the clearest convergence signal. Each rate threshold is independently configurable — fast-changing cases (real-time monitoring) need different thresholds than slow cases (multi-day investigations). `stabilityWindow` prevents false positives from temporary lulls. Synthetic goal integration means no new termination path — the existing `GoalReachedEventHandler` handles case status transition if the CaseDefinition declares a convergence completion goal.

**Trade-offs:** Single convergence firing per case. If a case "de-converges" (activity resumes after convergence), the detector won't fire again. Acceptable — convergence is a terminal detection, not a toggle. Multi-phase cases (explore → exploit, breadth-first → depth-first) should use explicit phase tracking via goals and context keys, not convergence detection. Convergence detection is the "I don't know when it's done" escape hatch for the terminal state, not a phase-transition mechanism. Note: the synthetic `GoalReachedEvent` is published on the event bus and processed by `GoalReachedEventHandler` asynchronously — convergence detection and the resulting case termination are not atomic within one evaluation cycle. A worker could complete between detection and termination. This is safe: `GoalReachedEventHandler` checks `currentState.isTerminal()` and `CaseStatusChangedHandler` uses CAS for terminal transitions, preventing duplicate transitions.

**Sources:** `GoalReachedEventHandler.java:102-148` (goal evaluation), `QuiescenceTracker.java` (per-case state pattern), D45 (pipeline phase), D46 (activity rates)
**Depends on:** D45 (pipeline phase), D46 (ActivityTracker provides rates), D48 (budget enforcement runs before convergence)
**Exploration:** quick
**Status:** revised — R1-10: clarified terminal-only semantics; R2-04: noted async goal firing with existing CAS safety

## D50: Goal integration — convergence goal kind and CaseDefinition wiring

**Choice:** Convergence-triggered termination reuses `GoalBasedCompletion`. New reserved goal name `"_converged"` — the engine fires this when the `ConvergenceDetector` detects convergence. Case definitions that want convergence-based termination declare it in their completion block:
```yaml
completion:
  success:
    anyOf: [case-resolved, _converged]
```
The `_converged` goal is fired by the engine with `StandardGoalKind.SUCCESS` (terminal status: `COMPLETED`), not by any agent. The GoalKind determines the terminal CaseStatus if the completion block triggers on this goal. If a CaseDefinition does not include `_converged` in its completion goals, convergence detection still runs (for monitoring/audit) but does not trigger termination. A new `CaseHubEventType.CONVERGENCE_DETECTED` is always written to EventLog regardless of whether termination fires.

**Alternatives:**
- New CaseCompletion variant (`ConvergenceCompletion`) — requires unsealing `CaseCompletion` and adding a new code path in `GoalReachedEventHandler`. More invasive.
- Direct `CaseStatusChanged(COMPLETED)` dispatch — bypasses goal system, creates a second termination path. Fragile and harder to reason about.

**Rationale:** GoalBasedCompletion is already the extensible completion mechanism. `GoalKind` is an interface (not enum), so custom kinds work. Adding a convergence goal is purely declarative — no code changes to the completion system. The `_` prefix convention distinguishes engine-fired goals from agent-fired goals.

**Trade-offs:** Case authors must explicitly opt in to convergence termination by adding `_converged` to their completion goals. This is intentional — convergence detection without termination is useful for monitoring. Automatic termination on convergence would surprise case authors who don't expect it. `StandardGoalKind.SUCCESS` is the correct kind because convergence in the stigmergy model IS the expected terminal condition — the swarm has settled. Whether that settlement represents genuine completion or deadlock is a semantic judgment that belongs to the case author's completion block (e.g., require BOTH `case-resolved` AND `_converged` for true success). The `CONVERGENCE_DETECTED` event metadata includes diagnostic context: active agent count, unmet goal names, final activity rates, and total coordination metrics — enabling operators to distinguish genuine completion from deadlocked quiescence via the audit trail.

**Sources:** `GoalBasedCompletion.java:23-56` (GoalBasedCompletion builder), `GoalKind.java:17` (interface, not enum), `StandardGoalKind.java` (SUCCESS → COMPLETED), `GoalReachedEventHandler.java:102` (evaluateCompletion), D49 (fires synthetic goal)
**Depends on:** D49 (ConvergenceDetector fires the goal)
**Exploration:** quick
**Status:** revised — R1-16: specified GoalKind = StandardGoalKind.SUCCESS for _converged goal; ADR-R1-09: added diagnostic metadata to CONVERGENCE_DETECTED event (active agents, unmet goals, rates)

## D51: OutputConvergenceMonitor — output similarity tracking per binding

**Choice:** `OutputConvergenceMonitor` (`runtime-core`, `@ApplicationScoped`) tracks per-binding output similarity across agents. On each successful worker completion (`WorkflowExecutionCompletedHandler` success path), stores the output key set and a content hash per key. When `recentOutputCount >= convergenceMinSamples` (configurable, default 3), computes pairwise Jaccard similarity on key sets. When average Jaccard exceeds `convergenceThreshold` (configurable, default 0.9) AND value hashes match for overlapping keys, fires `OUTPUT_CONVERGENCE_DETECTED` CaseHubEventType as an informational event. Per-binding sliding window of last N outputs (default 10). No cross-binding comparison — convergence is evaluated within the same capability. This is NOT an anti-collusion mechanism — it detects structural convergence, which may indicate either independent consensus (correct behavior) or groupthink (a concern). The interpretation is semantic and belongs in blocks, not the engine.

**Alternatives:**
- Signal concentration monitoring — only catches collusion manifesting through signals, misses output-level convergence.
- LLM-based semantic analysis (deferred to blocks) — engine provides metrics, blocks provides intelligence. Future extension via observer SPI.

**Rationale:** Key-set Jaccard + value hash is classical, deterministic, and O(K×N²) where K = output keys and N = window size (small). Detects structurally identical outputs. Per-binding scoping makes the comparison meaningful — agents working on the same capability may or may not produce similar outputs depending on the domain. `convergenceMinSamples` prevents false positives when only 1-2 agents have run. The informational framing (OUTPUT_CONVERGENCE_DETECTED, not DIVERSITY_VIOLATION) correctly reflects what the engine can determine: structural similarity exists. Whether that similarity indicates consensus, collusion, or groupthink is a semantic judgment that belongs in blocks.

**Trade-offs:** Structural similarity only — semantically equivalent but structurally different outputs are not detected. Acceptable for v1 — LLM-backed semantic analysis is a natural blocks extension. Value hash comparison uses canonical JSON serialization (sorted keys, deterministic number formatting) to avoid serialization-order sensitivity — `{"a":1,"b":2}` and `{"b":2,"a":1}` produce identical hashes. For bindings where all agents use the same output schema (schema-defined key sets), Jaccard is always 1.0 and contributes no signal — the value-hash comparison is the entire convergence measure. This is acceptable: key-set Jaccard adds value for heterogeneous outputs (e.g., agents with no fixed output schema), while value-hash provides the actual convergence signal for schema-constrained bindings. **Scope:** This monitor targets traditional worker outputs (WorkerResult key-value pairs), not stigmergy coordination artifacts (signals, interests, rules). Stigmergy agents coordinate through the coordination layer — their convergence is detected by D49 activity quiescence and D67 coordination pattern detection (signal consensus, interest convergence). For mixed cases with both traditional workers and stigmergy agents, the monitor tracks only the traditional outputs.

**Sources:** `WorkflowExecutionCompletedHandler.java` (success path, output access), `ConflictResolver.java` (output key handling precedent), engine#1110 issue spec
**Depends on:** D45 (pipeline runs after outputs are recorded), D46 (ActivityTracker pattern for per-case state)
**Injection note:** `OutputConvergenceMonitor` is injected into `WorkflowExecutionCompletedHandler` via `Instance<OutputConvergenceMonitor>` with `isResolvable()` guard — transparent no-op when convergence module is absent. No circular dependency — both are `@ApplicationScoped` beans in `runtime-core`. The handler is large but adding an `Instance<>` injection follows the existing pattern (e.g., `Instance<StepOutcomeObserver>`, `Instance<CaseOutcomeObserver>`).
**Exploration:** quick
**Status:** revised — R1-06: reframed from anti-collusion/DIVERSITY_VIOLATION to informational; R2-05: clarified injection dependency and Instance<> guard pattern; R1-08: explicitly scoped to traditional worker outputs, not stigmergy coordination artifacts; ADR-R1-09: canonical JSON serialization for value hashing, acknowledged Jaccard limitation for schema-defined outputs

## D52: ConvergenceConfig — per-case configuration

**Choice:** Three independent config records in `engine-api` under `io.casehub.api.model.convergence`:

```java
BudgetConfig(
    Integer maxDispatches,           // null = no limit
    Integer maxSignalDeposits,
    Integer maxContextMutations,
    Integer maxEvaluationCycles
)

ConvergenceThresholdConfig(
    Double dispatchRateThreshold,       // default 0.1
    Double signalDepositRateThreshold,  // default 0.1
    Double contextMutationRateThreshold,// default 0.1
    Double evaluationRateThreshold,     // default 0.5
    Duration stabilityWindow,           // default 30 seconds
    Duration rateWindow                 // default 60 seconds
)

OutputConvergenceConfig(
    Double convergenceThreshold,     // default 0.9
    Integer convergenceMinSamples,   // default 3
    Integer convergenceWindowSize    // default 10
)
```

`CaseDefinition` gains three nullable fields: `budgetConfig`, `convergenceThresholdConfig`, `outputConvergenceConfig`. Each is independently nullable — presence activates the feature, absence disables it. No master switch needed. YAML: three separate blocks under `spec:`. A user who wants only budget enforcement configures only `budgetConfig:` — the convergence and output monitoring fields don't exist in their YAML.

**Alternatives:**
- Single monolithic record — crammed 14+ fields serving three independent concerns. User must understand all fields to configure any one concern. The "less surface area" argument is false — the surface area is identical, just less self-documenting.
- Config on individual bindings — convergence is a case-level concern, not per-binding.

**Rationale:** Each config record maps 1:1 to its concern: budget limits, activity quiescence detection, output convergence monitoring. A user enabling only budgets touches only budget fields. Presence-as-activation replaces the `enabled` master switch — more consistent with how `ObservationConfig`, `SignalConfig`, and `RuleConfig` work (they exist or they don't, no master switch on each).

**Trade-offs:** Three config surfaces instead of one. Acceptable — they ARE three independent concerns. YAML reads more clearly with three small blocks than one large one.

**Sources:** `ObservationConfig.java` (record pattern), `SignalConfig.java` (record pattern), `RuleConfig.java` (record pattern), `CaseDefinition` (config surface)
**Depends on:** D46 (defines what metrics exist), D49 (defines what thresholds mean), D51 (defines output convergence parameters)
**Exploration:** quick
**Status:** revised — R1-09: split monolithic ConvergenceConfig into BudgetConfig, ConvergenceThresholdConfig, OutputConvergenceConfig; presence-as-activation replaces master switch

## D53: Module placement — convergence types in api/model/convergence, infrastructure in common-core and runtime-core

**Choice:** `BudgetConfig`, `ConvergenceThresholdConfig`, `OutputConvergenceConfig` in `io.casehub.api.model.convergence`. `ActivityTracker` and `SlidingWindowCounter` in `io.casehub.engine.common.internal.convergence`. `ConvergenceDetector` and `OutputConvergenceMonitor` in `io.casehub.engine.internal.convergence` (runtime-core). Follows the established pattern: value types in api, mutable state management in common-core, handler/detection logic in runtime-core.

**Alternatives:** None — direct analog of D4 (observation), D17 (signals), D36 (neighbors), D44 (rules) placement.

**Rationale:** Consistent with every prior module placement decision in this epic.

**Sources:** D4, D17, D36, D44 (module placement precedents)
**Depends on:** D46, D49, D51, D52 (defines what types exist)
**Exploration:** quick
**Status:** revised — R1-06/R1-09: DiversityMonitor renamed to OutputConvergenceMonitor; ConvergenceConfig split into three records

## D54: Audit — CONVERGENCE_DETECTED, BUDGET_EXHAUSTED, DIVERSITY_VIOLATION event types

**Choice:** Three new `CaseHubEventType` values:
- `CONVERGENCE_DETECTED` — fired when all activity rates drop below threshold for the stability window. Metadata: `dispatchRate`, `signalDepositRate`, `contextMutationRate`, `evaluationRate`, `stabilityDuration`, `totalDispatches`, `totalSignalDeposits`, `totalContextMutations`, `totalEvaluationCycles`, `activeAgentCount`, `unmetGoalNames`.
- `BUDGET_EXHAUSTED` — fired when any cumulative budget cap is exceeded. Metadata: `exhaustedMetric`, `currentCount`, `budgetCap`.
- `OUTPUT_CONVERGENCE_DETECTED` — fired when output structural similarity exceeds threshold. Informational, not judgmental. Metadata: `bindingName`, `averageJaccard`, `matchingOutputCount`, `totalSamples`, `affectedAgents`.

All three are written to EventLog immediately when detected. `CONVERGENCE_DETECTED` is always written (even if the case does not have `_converged` in its completion goals — pure audit). `BUDGET_EXHAUSTED` is written before the case is faulted.

**Alternatives:** None — follows the established audit pattern from D16 (pheromone), D27 (interest), D44 (rules).

**Sources:** `CaseHubEventType.java`, D16 (audit pattern), D27 (audit pattern)
**Depends on:** D49, D48, D51 (define the detection events)
**Exploration:** quick
**Status:** revised — R1-06: renamed DIVERSITY_VIOLATION to OUTPUT_CONVERGENCE_DETECTED

## D55: Lifecycle — case termination eviction + Resettable

**Choice:** `CaseStatusChangedHandler` calls `activityTracker.evictByCase(caseId)` and `outputConvergenceMonitor.evictByCase(caseId)` on terminal case status. Both implement `Resettable` for demo/test replay. Same pattern as `SignalRegistry`, `ObservationRegistry`, `RuleRegistry`, `ContextHistoryBuffer`.

**Alternatives:** None — established lifecycle pattern.

**Sources:** `CaseStatusChangedHandler.java` (terminal eviction), `Resettable` interface, D18 (signal lifecycle), D44 (rule lifecycle)
**Depends on:** D46 (ActivityTracker), D51 (OutputConvergenceMonitor)
**Exploration:** quick
**Status:** revised — R1-06: DiversityMonitor → OutputConvergenceMonitor

## D56: WorkerRuntime surfacing — convergence metrics as read-only view

**Choice:** No new WorkerRuntime facet for convergence metrics. Agents should not directly read or influence convergence detection — it is a system-level concern. Convergence metrics are visible to agents indirectly: (1) through observations (a classical observer can watch for `CONVERGENCE_DETECTED` events), (2) through context signals (budget warnings can be written to context by local rules). The `ActivityTracker` is engine-internal infrastructure, not an agent-facing API. If future issues (#1111-#1115) need agent-visible metrics, a read-only `MetricsSpace` facet can be added without changing the tracker.

**Alternatives:**
- Add `MetricsSpace` facet now — provides `metrics() → CaseActivitySnapshot` with rate/count views. More transparent to agents but exposes system-level concern at the agent level.
- Add metrics to RuleContext — local rules could condition on activity rates. Useful but conflates coordination rules with system monitoring.

**Rationale:** Explicit v1 trade-off: agent-visible metrics are deferred, not rejected. Convergence detection is a system-level supervisory function. Agents coordinate via signals, observations, interests, neighbors, and rules — these are the agent-facing coordination primitives. The engine monitors aggregate behavior and intervenes when thresholds are breached. For v1, agents respond to the coordination primitives, not to system-level metrics. When swarm scenarios (#1112/#1113) demonstrate a concrete need for agent-level metric visibility, a read-only `MetricsSpace` facet with `activityRates() → Map<String, Double>` is the planned extension path — no tracker changes required.

**Trade-offs:** Agents cannot proactively respond to convergence metrics (e.g., voluntarily reduce activity when approaching a budget cap). They can only respond to the engine's interventions (faulted case, convergence goal). Acceptable — the engine is the authority on convergence, not the agents. If future issues (#1111-#1115) need agent-visible activity metrics, a read-only `MetricsSpace` facet can be added without changing the tracker infrastructure.

**Sources:** D19 (faceted architecture), D32 (NeighborSpace — read-only facade precedent), engine#1110 issue spec
**Depends on:** D46 (ActivityTracker is the infrastructure being surfaced or not)
**Exploration:** quick
**Status:** revised — R2-06: replaced weak gaming rationale with clearer separation-of-concerns argument; ADR-R1-12: reframed as explicit v1 trade-off with planned MetricsSpace extension path

## D57: Cross-case coordination scoping — per-case only

**Choice:** All coordination state — signals, observations, interests, neighbors, rules — is scoped to a single case. No cross-case coordination mechanisms. Agents working across multiple cases cannot use stigmergic coordination to share findings between cases.

**Alternatives:**
- Cross-case signal namespace (e.g., global signals visible across cases) — requires distributed signal registry, changes the consistency model
- Case-group coordination (cases in the same group share a signal namespace) — intermediate option, requires group concept
- External coordination layer (message bus, shared database) — exists at the application level, outside engine scope

**Rationale:** Per-case scoping is correct for v1 and consistent with the platform's per-case isolation model. All evaluation state (goals, plan items, bindings) is per-case. Cross-case coordination is an application-level concern that the engine should not own in v1. The per-case `ConcurrentHashMap` storage model (D29) is a data structure decision, not an architectural constraint — cross-case queries could be added over the same storage if needed. The architectural constraint is the `CaseEvaluationSerializer` gate, which serializes per-case.

**Trade-offs:** Agents investigating related cases (e.g., AML network analysis) must coordinate via external mechanisms (application-level context sharing, shared database). This is the right boundary for v1 — the engine provides per-case coordination primitives, the application provides cross-case orchestration.

**Sources:** D29 (in-memory only), `CaseEvaluationSerializer` (per-case gate), all coordination registries (keyed by caseId)
**Exploration:** quick (surfaced by R1-13)
**Status:** captured — made explicit from implicit per-case scoping

## D58: Quiescent case coordination asymmetry — accepted for v1

**Choice:** Signal decay is continuous (time-based), but perception only occurs during evaluation cycles. If a case goes quiescent (no context changes), signals decay mathematically but no agent perceives the decaying values. When activity resumes, accumulated coordination intelligence may have crossed the effective-zero threshold.

**Alternatives:**
- Periodic heartbeat evaluation — inject synthetic `CaseContextChangedEvent` on a timer to force perception even during quiescence. Adds operational complexity and fights the event-driven model.
- Perception-triggered decay — signals only decay when perceived (freeze decay during quiescence). Breaks the time-based semantics and makes signals dependent on evaluation frequency.
- Persistent coordination state — addresses the broader restart concern (D29) but doesn't fix the quiescence asymmetry.

**Rationale:** This is an inherent property of event-driven perception in a time-based decay model. The asymmetry is real but bounded: case authors should configure signal halfLife proportional to the expected case activity pattern. Fast-turnaround cases use minute-scale halfLife; multi-day investigations use hour/day-scale halfLife. D29 (in-memory only) means restart during quiescence loses everything regardless — the quiescence asymmetry is a lesser concern than the restart concern.

**Trade-offs:** Low-activity cases get less value from the coordination layer than high-activity cases. Acceptable for v1 — the coordination layer is most naturally useful for cases with sustained agent activity. **Documentation requirement:** YAML documentation for `signalConfig.defaultHalfLife` must explicitly warn: "Signals decay continuously even during periods of no case activity. Set halfLife to exceed the longest expected idle period, or accept that coordination state will be lost during quiescence."

**Sources:** D11 (exponential decay), D29 (in-memory only), `CaseContextChangedEvent` (evaluation trigger)
**Exploration:** quick (surfaced by R1-14)
**Status:** revised — R1-12: added explicit documentation requirement for halfLife quiescence warning

## D59: Observer evaluation order — non-deterministic across agents

**Choice:** Within a single evaluation cycle, the order in which agents' observers are evaluated is non-deterministic (depends on `ConcurrentHashMap` iteration order of `ObservationRegistry.getObservers()`). With parallel observer evaluation (D6 revision), all observers across all agents execute concurrently, making ordering irrelevant. Within an agent, observer registration order is preserved (List ordering).

**Alternatives:**
- Deterministic ordering (sorted by agentId) — adds overhead for no correctness benefit
- Priority-based ordering — adds complexity to the observer model

**Rationale:** Under parallel evaluation (D6), all observers run concurrently with a collective timeout. Ordering is moot — there is no "first" or "last." Each agent's observations are stored independently and do not affect other agents' observation results within the same cycle. The only interaction is through the shared ObservationContext, which is immutable.

**Trade-offs:** Non-reproducible execution traces (observer completion order varies between runs). Acceptable — observations are independent, so ordering doesn't affect correctness.

**Sources:** `ObservationRegistry.getObservers()` (ConcurrentHashMap iteration), D6 (parallel evaluation)
**Exploration:** quick (surfaced by R1-15)
**Status:** captured — made explicit from implicit non-determinism

## D60: Stigmergy execution model architecture — three-layer blend

**Choice:** The StigmergyExecutionModel is a composition of three layers, each addressing a different concern:

1. **StigmergyConfig** (configuration) — A unified config record on `CaseDefinition` that provides coordinated defaults for all coordination SPIs (signals, rules, convergence, budget, observation) and declares the agent population. Its presence activates stigmergy mode. YAML: `stigmergyConfig:` block.

2. **StigmergyStrategy** (dispatch) — A named `PlanningStrategy` (`"stigmergy"`) that manages agent population dispatch. On first cycle: dispatches all declared agents as COMPOUND-scoped workers. On subsequent cycles: monitors agent health via existing registry queries. Provides the lifecycle hook point (called every evaluation cycle by `PlanningStrategyLoopControl.select()`). Extends to dynamic population scaling in #1113.

3. **StigmergyCoordinator** (lifecycle) — An `@ApplicationScoped, Resettable` bean that tracks active agent state per case, publishes lifecycle events (`STIGMERGY_AGENT_JOINED`, etc.), and provides population-level queries. The strategy delegates to the coordinator for population state. Convergence detection can query it for agent-level quiescence.

Agents use existing WorkerRuntime facets (`signals()`, `interests()`, `neighbors()`, `rules()`) — no new facet. The existing 5-phase evaluation pipeline already drives the perceive→decide→act cycle — no new pipeline phase. Declarative YAML for individual interests/rules is deferred to a follow-up (per D28, D44).

**Alternatives:**
- PlanningStrategy only — captures dispatch but not configuration or lifecycle coordination. Case author must configure 5+ separate config blocks manually.
- Configuration pattern only — no new runtime type, minimal code. But no lifecycle tracking, no validation, no hook point for population management.
- Runtime lifecycle manager only — tracks agents but doesn't integrate with the strategy resolution system. No natural dispatch hook.
- Single class doing all three — violates SRP, harder to test, harder to extend independently.

**Rationale:** Stigmergy is a coordination model (fourth axis beyond the unified execution model's structure/dispatch/technique). It touches dispatch (agents need to be dispatched), configuration (SPIs need coherent defaults), and post-dispatch lifecycle (agents perceive, decide, act). The three-layer decomposition maps one layer per concern. Each layer is independently testable and extensible.

**Trade-offs:** Three new types vs. one. Mitigated: each is focused and small. The strategy layer is thin (trivial dispatch logic for v1) but provides the hook point for #1113 dynamic population management without architectural change.

**Sources:** Unified execution model spec §2.3 (composable strategies), §2.7 (orthogonal axes), §3.1 (stigmergy as choreographed dispatch), engine#604 (original stigmergy issue), engine#1111, arXiv:2608.26081 (SwarmWorld cognition/consequence split), D19 (faceted WorkerRuntime), D45-D56 (convergence decisions)
**Depends on:** D19 (faceted architecture), D45 (pipeline integration), D48 (budget enforcement), D49 (convergence detection)
**Exploration:** deep-analysis (first-principles derivation of coordination as fourth axis)
**Status:** captured

## D61: Module placement — StigmergyStrategy requires planning module

**Choice:** StigmergyStrategy lives in `planning-core` as a `NamedStrategy`, resolved by `StrategyResolver`. Cases using stigmergy need the planning module on the classpath. StigmergyConfig lives in `engine-api` (configuration type). StigmergyCoordinator lives in `runtime-core` (lifecycle management).

**Alternatives:**
- Standalone in runtime-core — would need a new LoopControl or hook in ChoreographyLoopControl. Breaks the existing pattern where all strategies live in the planning module.
- Split with fallback — coordinator and config available without planning, strategy requires it. Added complexity for unclear benefit — if you want stigmergy, you want the full model.

**Rationale:** Consistent with how sequential, HTN, and GOAP strategies work today — all require the planning module. `PlanningStrategyLoopControl` is the established mechanism for per-case strategy resolution. The planning module is already the natural home for strategy implementations.

**Trade-offs:** Cases without the planning module cannot use stigmergy. Acceptable — the planning module is lightweight and stigmergy is a coordination model that benefits from planning infrastructure (compound PlanItems, population tracking via PlanItemStore).

**Sources:** `PlanningStrategyLoopControl.java` (runtime), `ChoreographyLoopControl.java` (runtime), `DefaultPlanningStrategy.java` (planning-core), `StrategyResolver` (common-core)
**Depends on:** D60 (three-layer architecture — defines what goes where)
**Exploration:** quick
**Status:** captured

## D62: StigmergyConfig as coordinated defaults preset

**Choice:** `StigmergyConfig` provides default values that fill in missing per-SPI config fields on `CaseDefinition`. If a case declares both `stigmergyConfig` AND an explicit `signalConfig`, the explicit config wins. StigmergyConfig is a "preset" — sensible stigmergy defaults without configuring 6 blocks individually. Existing config fields (`signalConfig`, `ruleConfig`, `convergenceThresholdConfig`, `budgetConfig`, `outputConvergenceConfig`, `observationConfig`) are unchanged and retain their nullable semantics.

Resolution order: explicit per-SPI config > StigmergyConfig defaults > system defaults (null = disabled).

**Alternatives:**
- Wrapper that replaces individual configs — breaks backward compat for cases using individual configs alongside stigmergy
- Activator only, no defaults — doesn't reduce configuration burden, case author still configures 6 blocks

**Rationale:** Stigmergy requires all coordination SPIs working together with compatible settings. A preset gives the case author a working stigmergy setup with one config block. Power users override specific aspects without losing the rest. No breaking changes to existing config model.

**Trade-offs:** Two resolution paths (preset vs explicit). Mitigated: resolution is a simple null-check per field at case initialization time. Debug: EventLog entry at case start could log effective config source per block.

**Sources:** `CaseDefinition` (existing nullable config fields), `SignalConfig`, `RuleConfig`, `ConvergenceThresholdConfig`, `BudgetConfig`, `OutputConvergenceConfig`, `ObservationConfig`
**Depends on:** D60 (StigmergyConfig is Layer 1 of the three-layer architecture)
**Exploration:** quick
**Status:** captured

## D63: Agent model — all bindings are agents in stigmergy compound

**Choice:** When a compound PlanItem's `planningStrategy` is `"stigmergy"`, ALL bindings within that compound are treated as stigmergy agents. The `StigmergyStrategy` dispatches them at case start with `COMPOUND` scope. Trigger conditions (`on:`, `when:`) on bindings are ignored — the strategy handles all dispatch decisions. If a case needs non-stigmergic bindings, they go in a different compound with a different strategy.

**Alternatives:**
- Explicit agent list in StigmergyConfig — allows mixing stigmergy and non-stigmergy bindings in the same compound. More configuration surface, blurs compound's strategy semantics.
- Implicit via COMPOUND scope — uses existing field, but COMPOUND scope has meaning independent of stigmergy (worker persistence). Overloading it conflates two concerns.

**Rationale:** The unified execution model's key principle is that strategy scopes to compound PlanItems. A compound with `planningStrategy: stigmergy` is a stigmergy compound — all its children follow stigmergy semantics. Mixed dispatch within a single compound violates the per-compound strategy model. Nesting provides the composition path: a root compound (choreography) containing a stigmergy compound (stigmergy agents) alongside normal bindings.

**Trade-offs:** Explicit triggers (other than `ScopeActivatedTrigger`) on stigmergy bindings cause a validation failure at case definition initialization time. `IllegalStateException` with message: "Bindings in a stigmergy compound do not use triggers — the strategy manages dispatch. Remove the `on:` clause or move the binding to a non-stigmergy compound." This is a build-time error, not a runtime warning — silent behavior changes during migration from non-stigmergy to stigmergy compounds are correctness hazards. Forces the use of nested compounds for mixed-model cases — acceptable since the unified execution model already expects composition via nesting.

**Sources:** Unified execution model spec §2.3 (per-compound strategy), §2.1 (compound PlanItem contains children), `PlanningStrategyLoopControl.select()`, engine#1111
**Depends on:** D60 (three-layer architecture), D61 (planning module dependency)
**Exploration:** quick
**Status:** revised — R1-10: changed from runtime WARN to validation failure at case definition initialization

## D64: Agent lifecycle — three states with voluntary departure

**Choice:** `StigmergyCoordinator` tracks three lifecycle states per agent per case:

- `JOINING` — agent dispatched, worker execution in progress (setup: registering interests, rules, initial signals). Transition → ACTIVE when worker completes successfully.
- `ACTIVE` — agent participating in coordination. Its observers evaluate, rules fire, signals are deposited. Transition → DEPARTED on voluntary `leave()` or case termination.
- `DEPARTED` — agent has deregistered all state (observers, rules, signals) and is no longer participating. Terminal state.

`WorkerRuntime` gains a `leave()` method for voluntary departure. On `leave()`: deregisters all observers (via `ObservationRegistry.unregisterByAgent()`), deregisters all rules (via `RuleRegistry.unregisterByAgent()`), transitions agent to DEPARTED in coordinator, publishes `STIGMERGY_AGENT_DEPARTED` event. Signals deposited by the agent are NOT removed — they decay naturally (consistent with biological stigmergy: an ant that leaves doesn't erase its pheromone trail).

**Alternatives:**
- Two states (ACTIVE/DEPARTED) — can't distinguish setup-in-progress from fully active, loses diagnostic value
- Four states (adding QUIESCENT) — per-agent quiescence adds complexity. System-level convergence detection (D49) already tracks aggregate activity rates. Per-agent quiescence is a natural extension for #1112 (swarm) but premature for #1111.

**Rationale:** Three states capture the essential lifecycle without over-engineering. JOINING is diagnostic — if an agent stays JOINING for too long, the setup failed. ACTIVE is the steady state. DEPARTED enables voluntary exit, which reduces agent count and can accelerate convergence.

**Trade-offs:** `leave()` on `WorkerRuntime` is available to ALL workers, not just stigmergy agents. Calling `leave()` outside stigmergy mode is a no-op (coordinator is not tracking the agent). Alternatively, `leave()` could throw `IllegalStateException` — but no-op is safer and follows the `default` method pattern on WorkerRuntime.

**Sources:** `ObservationRegistry.unregisterByAgent()` (existing), `RuleRegistry` (needs `unregisterByAgent()`), D2 (registration mechanism), D18 (signal lifecycle — case-scoped eviction), D19 (faceted architecture)
**Depends on:** D60 (StigmergyCoordinator is Layer 3), D63 (all bindings are agents)
**Exploration:** quick
**Status:** captured

## D65: Strategy dispatch — first-cycle dispatch with health monitoring

**Choice:** `StigmergyStrategy.select()` tracks whether initial dispatch has happened (per case, via `StigmergyCoordinator`). On first `select()` call: returns all bindings for dispatch with `COMPOUND` lifecycle scope. On subsequent calls: returns empty list (no new dispatches). The strategy monitors agent health via coordinator queries each cycle — if an agent fails (worker execution error), it can re-dispatch that binding. No condition-gated or population-managed dispatch in v1.

**Alternatives:**
- Condition-gated dispatch — agents have activation conditions evaluated each cycle. Enables delayed joining. Blurs the line with choreography and adds complexity. Natural extension for v2 or #1112.
- Population-managed dispatch — strategy actively manages agent count. Foundation for #1113 (self-provisioning). Premature for v1.

**Rationale:** First-cycle dispatch is the simplest model that delivers working stigmergy. All agents join at case start, observe, decide, act. The pipeline drives the cycle. Health monitoring provides resilience (failed agents are re-dispatched) without adding complexity. Condition-gated and population-managed dispatch are natural extensions that can be added to `StigmergyStrategy.select()` without changing the architecture.

**Trade-offs:** All agents start simultaneously — no staggered or conditional joining. Acceptable for rule-based stigmergy where the agent population is known at case definition time. Dynamic joining is a #1112/#1113 concern.

**Sources:** `PlanningStrategy.select()` (existing SPI), `PlanningStrategyLoopControl.java` (evaluation cycle dispatch), D60 (strategy is Layer 2)
**Depends on:** D60 (StigmergyStrategy), D63 (all bindings are agents), D64 (lifecycle tracking)
**Exploration:** quick
**Status:** captured

## D66: StigmergyConfig structure — defaults + coordination thresholds

**Choice:** `StigmergyConfig` record in `engine-api` with two sub-records:

`StigmergyDefaults`: `signalHalfLife` (Duration, default PT5M), `effectiveZeroThreshold` (double, 0.01), `maxSignalsPerCase` (int, 100), `maxObserversPerCase` (int, 20), `maxRulesPerCase` (int, 50), `rateWindow` (Duration, PT60S), `stabilityWindow` (Duration, PT30S), `maxDispatches` (Integer, 10000), `maxEvaluationCycles` (Integer, 10000). All nullable — null means don't override the per-SPI default.

`CoordinationConfig`: `consensusThreshold` (int, default 2 — min reinforcement count for SIGNAL_CONSENSUS_DETECTED), `stormRateMultiplier` (double, default 10.0 — rates above convergenceThreshold × multiplier = storm), `interestHotspotThreshold` (double, default 0.6 — hotspot score for INTEREST_CONVERGENCE_DETECTED). All have sensible defaults — case author can use `stigmergyConfig: {}` with zero configuration to get working stigmergy.

YAML: `stigmergyConfig:` block with optional `defaults:` and `coordination:` sub-blocks. Empty `stigmergyConfig:` activates stigmergy mode with all defaults.

**Alternatives:**
- Flat record (all fields at top level) — loses semantic grouping between SPI defaults and coordination intelligence
- Per-detector config records — over-segmented for three threshold values

**Rationale:** Two sub-records map 1:1 to two concerns: SPI defaults (what values the coordination SPIs use) and coordination intelligence (what patterns the coordinator detects). Empty config with all defaults gives a zero-configuration entry point. Power users tune specific knobs.

**Trade-offs:** StigmergyDefaults overlaps with per-SPI config fields. Resolution is explicit: per-SPI config > StigmergyDefaults > system defaults. This is documented in the CaseDefinition initialization logic.

**Sources:** `SignalConfig`, `RuleConfig`, `ConvergenceThresholdConfig`, `BudgetConfig`, `OutputConvergenceConfig`, `ObservationConfig` (existing per-SPI configs), D62 (coordinated defaults)
**Depends on:** D60 (StigmergyConfig is Layer 1), D62 (defaults preset model)
**Exploration:** quick
**Status:** captured

## D67: Coordination pattern detection — three detectors in convergenceDetection phase

**Choice:** Three coordination pattern detectors run during the `convergenceDetection()` pipeline phase for stigmergy cases. Each is a method on `StigmergyCoordinator`, examining existing registry state with no new storage — pure computation over existing data.

1. **Signal consensus detection**: When a signal's `sources.size()` crosses `consensusThreshold`, emit `SIGNAL_CONSENSUS_DETECTED`. Tracks which signals have already fired consensus events (per-case `Set<String>`) to avoid repeated firing. Resets when signal decays below effective-zero. Computation: O(S) where S ≤ `maxSignalsPerCase`. Reads `SignalRegistry.perceiveAll(caseId)`.

2. **Coordination storm detection**: When any ActivityTracker rate exceeds `convergenceThreshold × stormRateMultiplier`, emit `COORDINATION_STORM_DETECTED`. Fires once when storm begins. Resets (can fire again) after all rates drop below storm threshold (hysteresis). Computation: O(1) — reads four rate values from ActivityTracker.

3. **Interest convergence detection**: When the interest landscape shows a hotspot score above `interestHotspotThreshold`, emit `INTEREST_CONVERGENCE_DETECTED`. Fires once per hotspot key. Resets when hotspot dissolves (agents deregister interests). Computation: O(N) where N ≤ `maxObserversPerCase`.

`convergenceDetection()` in `CaseContextChangedEventHandler` gains: `if coordinator.isStigmergyCase(caseId): coordinator.detectPatterns(...)` after existing ConvergenceDetector and BudgetEnforcer.

**Alternatives:**
- Lifecycle events only (no pattern detection) — misses the core value of stigmergy audit. Individual SPI events exist but don't tell the coordination story.
- Separate pipeline phase for coordination intelligence — adds a 6th phase. Over-engineered; convergenceDetection is the natural home since it already examines aggregate behavior.
- Pattern detection in a separate bean — unnecessary separation when patterns are simple threshold checks on existing data.

**Rationale:** The three patterns cover the essential stigmergy dynamics: consensus (the mechanism working), storms (the mechanism pathological), and attention convergence (the mechanism focusing). All are mechanically detectable — no LLM needed. They run alongside existing convergence checks with negligible overhead.

**Trade-offs:** Per-case tracking state for "already fired" (Set<String> for consensus signals, boolean for storm, Set<String> for hotspot keys) adds memory proportional to signals/keys. Bounded by `maxSignalsPerCase` and `maxObserversPerCase`. Evicted on case termination alongside all other coordinator state.

**Sources:** `SignalRegistry.perceiveAll()`, `ActivityTracker` (rate queries), `ObservationRegistry` (interest landscape), D35 (Signal.sources tracking), D46 (ActivityTracker rates), D49 (ConvergenceDetector — existing pattern), D54 (existing convergence events)
**Depends on:** D60 (StigmergyCoordinator is Layer 3), D66 (CoordinationConfig provides thresholds)
**Exploration:** quick
**Status:** captured

## D68: Seven new CaseHubEventTypes for stigmergy

**Choice:** Seven new `CaseHubEventType` values in two groups:

**Lifecycle events:**
- `STIGMERGY_CASE_INITIALIZED` — case starts with stigmergy mode. Metadata: `agentCount`, config summary.
- `STIGMERGY_AGENT_JOINED` — agent dispatched (JOINING state). Metadata: `agentId`, `bindingName`.
- `STIGMERGY_AGENT_ACTIVATED` — agent completed setup (ACTIVE state). Metadata: `agentId`, `interestCount`, `ruleCount`.
- `STIGMERGY_AGENT_DEPARTED` — agent voluntarily left. Metadata: `agentId`, `reason`, `activeTimeMs`.

**Coordination intelligence events:**
- `SIGNAL_CONSENSUS_DETECTED` — signal reinforced by N agents. Metadata: `signalName`, `reinforcementCount`, `sources` (agent IDs), `effectiveStrength`.
- `COORDINATION_STORM_DETECTED` — activity rates exceed storm thresholds. Metadata: `stormingMetrics` (which rates), `currentRates`, `stormThreshold`.
- `INTEREST_CONVERGENCE_DETECTED` — collective attention focusing. Metadata: `hotspotKeys`, `watchingAgentCount`, `hotspotScore`.

**Alternatives:** None significant — follows established CaseHubEventType patterns from D16 (pheromone), D27 (interest), D44 (rules), D54 (convergence).

**Rationale:** Two groups tell different parts of the coordination story. Lifecycle events track WHO is participating and WHEN. Coordination events track WHAT patterns are emerging. Together they give a complete audit trail of stigmergic coordination — readable from EventLog without manual correlation of hundreds of individual SPI events.

**Trade-offs:** Seven new event types is a significant addition to the enum. Justified: each captures a distinct, non-overlapping concern. All are conditional (only published for stigmergy cases or when thresholds are crossed).

**Sources:** `CaseHubEventType.java` (existing enum), D16 (pheromone events), D27 (interest events), D44 (rule events), D54 (convergence events)
**Depends on:** D64 (lifecycle states define when lifecycle events fire), D67 (detectors define when coordination events fire)
**Exploration:** quick
**Status:** captured

## D69: Trigger-less bindings in stigmergy compounds

**Choice:** Bindings within a stigmergy compound do not require trigger conditions (`on:`, `when:`). The `StigmergyStrategy` handles all dispatch decisions — triggers are the strategy's concern, not the binding's. Explicit triggers (other than `ScopeActivatedTrigger`) on stigmergy bindings cause a validation failure at case definition initialization time, consistent with D63. `IllegalStateException` with message: "Bindings in a stigmergy compound do not use triggers — the strategy manages dispatch. Remove the `on:` clause or move the binding to a non-stigmergy compound."

For YAML: `on:` is optional when `planningStrategy: stigmergy`. For Java: Binding.builder() allows `build()` without `on()` when the binding will be added to a stigmergy compound.

Implementation path: the case initializer automatically adds `ScopeActivatedTrigger` to bindings within a stigmergy compound when no trigger is specified. This is transparent to the YAML author — they omit `on:` and the initializer fills in the scope-activated trigger. `PlanningStrategyLoopControl.collectScopeActivatedBindings()` handles dispatch via its existing path when the compound activates. The `StigmergyStrategy.select()` focuses on health monitoring and re-dispatch, not initial dispatch. No new trigger type required.

**Alternatives:**
- Require always-true triggers — forces case authors to write `on: { contextChange: { filter: 'true' } }` on every binding. Boilerplate that contradicts the "strategy manages dispatch" principle.
- Remove triggers entirely for stigmergy — too aggressive, since future extensions (condition-gated dispatch per D65 alternatives) would want optional triggers back.

**Rationale:** The unified execution model's §2.2 establishes that ORCHESTRATED dispatch mode means "parent's strategy selects this item." The strategy IS the trigger. Requiring an explicit trigger alongside strategy-managed dispatch is redundant. Making triggers optional gives the cleanest YAML surface.

**Trade-offs:** Binding validation must be context-aware (needs to know parent compound's strategy to validate trigger requirement). Mitigated: validation happens at CaseDefinition initialization time when compound context is available, not at individual Binding construction time.

**Sources:** `Binding.Builder.build()`, unified execution model spec §2.2 (dispatch modes), D63 (all bindings are agents)
**Depends on:** D63 (all bindings are agents), D65 (strategy handles dispatch)
**Exploration:** quick
**Status:** revised — ADR-R1-08: specified ScopeActivatedTrigger as integration path; case initializer auto-adds trigger for trigger-less bindings in stigmergy compounds; ADR-R1-05: aligned trigger handling with D63 validation failure (was: runtime WARN + ignore)

## D70: Module placement — package structure

**Choice:** Detailed package placement:

| Component | Module | Package |
|-----------|--------|---------|
| `StigmergyConfig`, `StigmergyDefaults`, `CoordinationConfig` | engine-api | `io.casehub.api.model.stigmergy` |
| `AgentLifecycleState` (enum), `AgentState` (record) | engine-api | `io.casehub.api.model.stigmergy` |
| New `CaseHubEventType` values | engine-api | `io.casehub.api.event` (existing enum) |
| `StigmergyCoordinator` | runtime-core | `io.casehub.engine.internal.stigmergy` |
| `StigmergyStrategy` | planning-core | `io.casehub.engine.plan.strategy` (existing strategy package) |

`StigmergyCoordinator` is injected into `CaseContextChangedEventHandler` via `Instance<StigmergyCoordinator>` with `isResolvable()` guard — transparent no-op when runtime-core is present but no stigmergy case is active. No circular dependency — both are `@ApplicationScoped` beans in runtime-core. Follows the `Instance<OutputConvergenceMonitor>` pattern from D51.

`StigmergyStrategy` follows the `DefaultPlanningStrategy` pattern in planning-core — registered as a `NamedStrategy` with `id() = "stigmergy"`.

**Alternatives:** None significant — follows established patterns from D4, D17, D36, D44, D53.

**Rationale:** API types in `api/model/stigmergy` (new domain package matching signals, convergence). Infrastructure in `runtime-core/internal/stigmergy`. Strategy in `planning-core/strategy` alongside other strategies. Each follows the established tier: api → common-core → runtime-core → planning-core.

**Sources:** D4 (observation placement), D17 (signal placement), D53 (convergence placement), D61 (planning module dependency)
**Depends on:** D60 (three-layer architecture defines what exists), D61 (planning module houses the strategy)
**Exploration:** quick
**Status:** captured

## D71: Evaluation backpressure — per-case event coalescing in serializer

**Choice:** Explicit v1 trade-off: no backpressure mechanism beyond the existing `CaseEvaluationSerializer` per-case serialization and `BudgetConfig` cumulative caps. The serializer guarantees at most one evaluation per case at a time — concurrent context changes for the same case are queued and processed sequentially. Cross-case evaluation runs concurrently on virtual threads. `BudgetConfig` caps detect and terminate runaway cases after the fact. No bounded queue, no event dropping, no coalescing of duplicate context-change events.

**Alternatives:**
- Bounded queue per case with coalesced duplicate events — drops redundant `CONTEXT_CHANGED` events when the queue is full. Reduces evaluation pressure but risks losing meaningful context changes that appear identical to duplicates.
- Rate limiter on evaluation cycles — caps evaluation frequency per case (e.g., max 10 cycles/second). Adds latency to legitimate high-activity cases. Better suited for multi-tenant production hardening than v1 correctness.
- Cross-case evaluation thread pool with bounded queue — limits total concurrent evaluations. The virtual thread pool already provides this implicitly — virtual threads are cheap but the underlying carrier pool is bounded by CPU count.

**Rationale:** For v1, the existing mechanisms are sufficient: per-case serialization prevents concurrent evaluation, virtual threads handle cross-case concurrency efficiently, and budget enforcement provides the hard safety net. The primary risk scenario (swarm with many cases and frequent signal deposits) is bounded by `maxSignalsPerCase` × case count. Event coalescing is the natural next step when multi-tenant production workloads surface the need — the serializer's pending-event queue is the right coalescing point.

**Trade-offs:** A case can queue thousands of evaluation cycles before budget enforcement stops it. Each queued evaluation runs to completion (including observer evaluation, rule evaluation, convergence detection) — wasted work when the result would be identical. Acceptable for v1 because budget enforcement catches pathological cases, and the per-case serializer prevents fan-out.

**Sources:** `CaseEvaluationSerializer.java` (per-case gate), D48 (budget enforcement), D46 (ActivityTracker)
**Depends on:** D48 (budget enforcement as backstop)
**Exploration:** quick (surfaced by ADR R1-14)
**Status:** captured — made explicit from implicit v1 trade-off

## D72: Registry synchronization — ReentrantLock for virtual thread compatibility

**Choice:** All per-case registries (`ObservationRegistry`, `RuleRegistry`) that currently use `synchronized` blocks should use `java.util.concurrent.locks.ReentrantLock` instead. This aligns with D6's own guidance that implementations "should avoid `synchronized` blocks (which pin platform threads — a known virtual thread anti-pattern)." The registry's internal synchronization should follow the same rule it imposes on observer implementations.

**Alternatives:**
- Keep `synchronized` — the critical sections are short (list add/remove), so pinning duration is brief. Pragmatically acceptable but inconsistent with D6's guidance.
- `CopyOnWriteArrayList` for registrations — eliminates read-side locking entirely. Write-heavy scenarios (frequent registration changes in swarm mode) would degrade due to full-copy-on-write. Reads far exceed writes for observer evaluation, but registration churn in COMPOUND workers is write-heavy during setup.
- `ReadWriteReentrantLock` — separate read and write locks. Optimal for read-heavy access patterns but adds complexity for registries where the critical sections are already short.

**Rationale:** `ReentrantLock` is the minimal change that eliminates the virtual thread pinning anti-pattern. The critical sections remain short. The lock is per-case (each case has its own registration list), so contention is limited to concurrent registrations for the same case — rare in practice.

**Trade-offs:** Minor API change in registry internals (synchronized → lock/unlock). No externally visible change.

**Sources:** `ObservationRegistry.java` (synchronized blocks), D6 (virtual thread guidance), JEP 444 (Virtual Threads — synchronized pinning)
**Depends on:** D6 (thread model guidance)
**Exploration:** quick (surfaced by ADR R1-15)
**Status:** captured — made explicit from implicit inconsistency

## D73: Swarm architecture — extend stigmergy, not separate strategy

**Choice:** The swarm execution model extends `StigmergyStrategy` and `StigmergyCoordinator` with new tracking/detection registries. Same `planningStrategy: stigmergy`, no new strategy type. `StigmergyConfig` gains an optional `swarm` sub-record. Swarm behaviors (role tracking, team detection, progress monitoring, agent-visible metrics) layer onto the existing stigmergy infrastructure. Scope is tracking + detection only — no dynamic agent scaling (that's #1113), no LLM-driven role assignment (that's blocks).

**Alternatives:**
- Separate `planningStrategy: swarm` with `SwarmStrategy` — cleaner separation but more types, another strategy to resolve, and duplicates stigmergy dispatch logic
- Layered composition (SwarmExecutionModel wraps stigmergy) — most modular but most indirection, and D60 already anticipated swarm as a stigmergy extension

**Rationale:** D65 explicitly notes that `StigmergyStrategy.select()` gains additional logic for #1112 without architectural change. Swarm IS stigmergy plus role awareness and collective intelligence. Keeping it as one strategy avoids the composition overhead and aligns with the "coordination as fourth axis" framing from D60.

**Trade-offs:** StigmergyConfig grows wider. Mitigated: SwarmConfig is a nullable sub-record — pure stigmergy cases don't see swarm fields.

**Sources:** D60 (three-layer architecture), D65 (strategy extension note), engine#1112
**Exploration:** quick
**Status:** captured

## D74: Role emergence — multi-dimensional behavioral fingerprinting

**Choice:** A role is the emergent behavioral pattern of an agent, computed from four domain-specific sub-vectors:

| Domain | Features | Source | Type |
|--------|----------|--------|------|
| Perception | interest keys | ObservationRegistry | binary (1.0) |
| Communication | signal names deposited | SignalRegistry (sources tracking) | normalized counts |
| Decision | rule IDs that fired | RoleTracker accumulator | normalized counts |
| Effect | context keys written | RoleTracker accumulator | normalized counts |

`BehavioralFingerprint` record: four `Map<String, Double>` sub-vectors. Similarity = weighted average of per-domain cosine similarities. Default weights: equal (0.25 each), configurable via `SwarmConfig.domainWeights`. Case authors can emphasize specific domains (e.g., "roles are defined by what agents communicate").

Role clusters detected via pairwise similarity + connected components with internal average similarity check. Cluster matching across cycles via majority member overlap (>50%). RoleTracker maintains per-agent sliding window of dynamic feature counts (configurable window size, default 20 cycles).

**Alternatives:**
- Set intersection (Jaccard) — simplest but loses all intensity information. Can't distinguish specialists from generalists.
- Flat feature vector + single cosine — captures intensity but mixes domains with different semantics. Domains with more features dominate similarity.

**Rationale:** Multi-dimensional fingerprint captures all four behavioral dimensions of perceive→decide→act independently. Domain weights give case authors control over what "role" means. Per-domain analysis gives richer audit data ("identical perception, divergent effects"). Built entirely from existing registry queries — no new data collection.

**Trade-offs:** More complex than Jaccard or flat vector. Four cosine computations per pair instead of one. Mitigated: N ≤ 20 agents, each domain vector is very sparse (≤20-100 features), total cost is negligible. **Scaling note:** The O(N²×K) pairwise cost (190 pairs × 4 domains × ≤100 features = ~76,000 ops for N=20) grows quadratically with agent count. If future issues (#1113 self-provisioning) raise `maxSwarmSize` above 20, the detection cost must be re-evaluated. At N=50: 1,225 pairs → ~490,000 ops. At N=100: 4,950 pairs → ~1.98M ops. The `detectionInterval` (default 10 cycles) amortizes this, but scaling `maxSwarmSize` requires explicit cost analysis.

**Sources:** arXiv:2603.28990 ("Drop the Hierarchy and Roles" — 5,006 emergent roles), SwarmSys (arXiv:2510.10047 — Explorer/Worker/Validator cycle), D73 deep exploration analysis
**Depends on:** D73 (swarm extends stigmergy)
**Exploration:** deep-analysis (first-principles derivation of four behavioral domains)
**Status:** captured

## D75: Team model — signal-based affinity clusters

**Choice:** Teams are emergent coordination clusters, not declared structures. Team affinity between two agents is a weighted function of three NeighborSpace relations: `SHARED_INTEREST` (attention alignment), `SHARED_SIGNAL` (communication alignment), `COMPLEMENTARY` (workflow alignment — one agent's outputs feed another's observations). Affinity = weighted Jaccard-like score across these three dimensions. Teams detected using same clustering algorithm as roles (connected components + internal average check) but on affinity matrix instead of behavioral similarity. Events: `SWARM_TEAM_FORMED`, `SWARM_TEAM_DISSOLVED`.

**Alternatives:**
- Explicit TeamRegistry with JoinTeam/LeaveTeam rule actions — more structured but adds a new coordination primitive, defeats emergence
- Compound-based teams (dynamic sub-compounds) — leverages existing compound lifecycle but requires dynamic PlanItem creation, a significant engine extension

**Rationale:** Teams are defined by WHO coordinates together, not WHAT they do (that's roles). The engine already tracks all inter-agent relations via NeighborSpace (D32-D34). Team detection is a clustering query over existing relation data. Ephemeral teams that form and dissolve naturally match biological swarm behavior — ant foraging parties form and dissolve without explicit group management.

**Trade-offs:** No explicit team identity — agents can't say "I'm in team X." Agents detect team membership indirectly via NeighborSpace queries. Acceptable for v1 — explicit team identity is a natural extension if needed.

**Sources:** D32-D34 (NeighborSpace architecture and relations), D35 (Signal.sources tracking), engine#1108, engine#1112
**Depends on:** D73 (swarm extends stigmergy), D74 (role clusters use same algorithm)
**Exploration:** quick
**Status:** captured

## D76: Work redistribution — departure event + rule reaction

**Choice:** No engine-orchestrated redistribution. When an agent fails or departs, the existing `STIGMERGY_AGENT_DEPARTED` event (D68) is published. Neighboring agents detect this via their local rules (rule condition checks neighbor state via NeighborSpace queries in lambda predicates) and adapt — register additional interests, deposit compensating signals, pick up the departed agent's work. Self-healing through the existing perceive→decide→act cycle. MetricsSpace (D78) enables agents to see the departure and adjust.

**Alternatives:**
- Coordinator-managed redistribution — StigmergyCoordinator actively deposits "help-needed" signals or modifies remaining agents' rules. More reliable but makes the coordinator an orchestrator, fighting the self-organization principle.
- Capability handoff protocol — failed agent's capabilities published as available, others claim. More structured but requires a new claiming mechanism.

**Rationale:** The entire point of self-organization is that the system adapts without central control. The engine provides the observation and notification infrastructure; agents provide the adaptive logic. This is exactly how biological swarms handle worker loss — neighboring ants detect the gap in pheromone refreshment and compensate by adjusting their response thresholds.

**Trade-offs:** Self-healing quality depends entirely on agent rule quality. Poorly-written rules won't compensate for departed agents. Acceptable — the engine provides the infrastructure, blocks (#1112 blocks issue #10) provides the LLM intelligence for sophisticated adaptation.

**Sources:** D64 (agent lifecycle — DEPARTED state), D68 (STIGMERGY_AGENT_DEPARTED event), D37 (rule action scope), engine#1112
**Depends on:** D73 (swarm extends stigmergy), D64 (departure lifecycle)
**Exploration:** quick
**Status:** captured

## D77: Swarm progress tracking — exploration, consensus, stability metrics

**Choice:** `SwarmProgressTracker` (`runtime-core/internal/stigmergy/`, `@ApplicationScoped`, `Resettable`) tracks three generic progress dimensions:

1. **Exploration breadth** — ratio of unique features explored (unique signal names deposited + unique context keys written) vs. a sliding maximum. Higher = more exploration.
2. **Consensus formation** — ratio of signals with consensus (sources.size() ≥ threshold) vs. total active signals. Higher = more agreement.
3. **Stability score** — role cluster stability over last N detection cycles (% of agents that stayed in the same cluster). Higher = swarm has settled.

`SwarmProgress` record: `(double explorationScore, double consensusScore, double stabilityScore, Instant computedAt)`. Each score is [0.0, 1.0]. Progress is computed during `convergenceDetection()` phase. `SWARM_PROGRESS` event fired when any score changes significantly (delta > 0.1) or at configurable intervals.

Mission completion: standard goal in completion block. Progress scores are proxies — the engine can't measure domain-specific progress. Agents use progress scores via MetricsSpace to adapt strategy (e.g., shift from exploration to exploitation when exploration score is high).

**Alternatives:**
- Mission as configuration string only — no runtime tracking, progress inferred from convergence detection. Simpler but agents can't self-regulate.
- Mission decomposition into sub-goals — most ambitious but requires goal generation logic (LLM concern, blocks scope).

**Rationale:** Three dimensions capture the generic dynamics of any swarm: explore (have we covered the space?), agree (do agents see the same things?), settle (have roles stabilized?). Domain-specific progress belongs in goal conditions, not the generic tracker. The tracker gives agents enough information to adapt their exploration/exploitation balance.

**Trade-offs:** Generic metrics are proxies, not direct progress measures. A high exploration score doesn't mean the right things were explored. Acceptable — domain-specific intelligence is blocks' concern.

**Sources:** D46 (ActivityTracker pattern), D67 (coordination pattern detection), D74 (role clusters for stability), engine#1112
**Depends on:** D73 (swarm extends stigmergy), D74 (role clusters for stability score)
**Exploration:** quick
**Status:** captured

## D78: MetricsSpace — 5th WorkerRuntime facet

**Choice:** `MetricsSpace` is a read-only WorkerRuntime facet (the 5th, alongside signals, interests, neighbors, rules):

```java
interface MetricsSpace {
    Map<String, Double> activityRates();
    Map<String, Long> budgetUsage();
    BehavioralFingerprint myFingerprint();
    SwarmProgress swarmProgress();
    List<DetectedRole> detectedRoles();
}
```

`DefaultMetricsSpace` (`runtime-core/internal/stigmergy/`) delegates to `ActivityTracker`, `RoleTracker`, `SwarmProgressTracker`. All methods return immutable snapshots. No mutations. `default MetricsSpace metrics() { return MetricsSpace.NOOP; }` on `WorkerRuntime`.

D56 planned this extension path — swarm is the concrete use case. Agents use metrics to adapt: see their own fingerprint and compare with role clusters, see swarm progress to decide explore vs. exploit, see budget usage to voluntarily throttle.

**Alternatives:**
- No MetricsSpace (keep deferred) — agents coordinate only through signals/interests/rules. Limits self-regulation.
- Partial (progress only) — less useful without activity rates and fingerprint.

**Rationale:** D56 explicitly deferred MetricsSpace for #1112. Swarm agents that can see system-level state make better adaptation decisions. Read-only ensures agents observe but don't manipulate system metrics. The facet pattern (D19) makes this a clean extension.

**Trade-offs:** Agents can game metrics (e.g., artificially inflate exploration by depositing diverse signals). Mitigated: budget enforcement caps total activity regardless of diversity.

**Sources:** D19 (faceted architecture), D56 (MetricsSpace deferral), D46 (ActivityTracker), engine#1112
**Depends on:** D73 (swarm extends stigmergy), D74 (fingerprints), D77 (progress tracker)
**Exploration:** quick
**Status:** captured

## D79: SwarmConfig — inside StigmergyConfig, presence-activated

**Choice:** `StigmergyConfig` gains an optional `SwarmConfig swarm` field. Presence activates swarm features (role tracking, team detection, progress monitoring, MetricsSpace). Absence means pure stigmergy without swarm tracking.

`SwarmConfig` record:
```java
SwarmConfig(
    Double roleSimilarityThreshold,     // default 0.7
    Integer roleMinClusterSize,         // default 2
    Integer roleDetectionWindow,        // default 20 (cycles for sliding window)
    Integer detectionInterval,          // default 10 (periodic re-cluster every N cycles)
    RoleDomainWeights domainWeights,    // default equal (0.25 each)
    Double teamAffinityThreshold,       // default 0.5
    Integer teamMinSize,                // default 2
    Double progressChangeThreshold      // default 0.1 (fire event when score delta > this)
)
```

All fields nullable — null means use default. YAML: `stigmergyConfig: { swarm: { ... } }`. Empty `swarm: {}` activates swarm with all defaults.

**Alternatives:**
- Separate top-level `swarmConfig` on CaseDefinition — more independent but two coordination configs to manage, and swarm without stigmergy is architecturally impossible.

**Rationale:** Swarm is an extension of stigmergy, not an independent concern. Nesting SwarmConfig inside StigmergyConfig makes the dependency explicit. Presence-as-activation follows the established pattern (StigmergyConfig itself, ObservationConfig, SignalConfig, etc.).

**Trade-offs:** StigmergyConfig grows wider. Mitigated: SwarmConfig is a nullable sub-record — pure stigmergy cases don't see it.

**Sources:** D62 (StigmergyConfig as preset), D66 (StigmergyConfig structure), engine#1112
**Depends on:** D73 (swarm extends stigmergy)
**Exploration:** quick
**Status:** captured

## D80: Detection frequency — periodic + event-triggered hybrid

**Choice:** Role and team detection use dual triggers:

1. **Periodic** — re-cluster every `detectionInterval` evaluation cycles (default 10). Catches gradual behavioral drift in dynamic features (rule firing patterns, signal deposit frequency). Fingerprint data accumulates every cycle (cheap counter increments); clustering runs only at periodic intervals.

2. **Event-triggered** — immediate re-cluster on structural changes: agent departure (`STIGMERGY_AGENT_DEPARTED`), new interest/rule registration, interest/rule deregistration. These are significant topology changes that can invalidate current clusters immediately.

Both triggers feed the same clustering pipeline. Periodic detection uses a cycle counter in RoleTracker. Event-triggered detection sets a "dirty" flag checked at the next `convergenceDetection()` phase — detection still runs within the pipeline, not inline with the registration event.

SwarmProgressTracker always runs on the periodic interval (progress is a slow metric).

**Alternatives:**
- Periodic only — misses immediate structural changes (agent departure leaves stale cluster data until next periodic tick)
- Event-triggered only — misses gradual behavioral drift in dynamic features
- Every cycle — wasteful; O(N²) clustering on every cycle when patterns change slowly

**Rationale:** Periodic and event-triggered are complementary, not alternatives. Periodic catches slow drift. Event-triggered catches sharp topology changes. The dirty-flag approach avoids running clustering inline with registration events (which happen inside the serializer gate) while ensuring the next evaluation cycle picks up the change.

**Trade-offs:** Slightly more complex trigger logic. Mitigated: dirty flag is a single boolean per case.

**Sources:** D67 (coordination pattern detection runs in convergenceDetection phase), D46 (ActivityTracker periodic pattern)
**Depends on:** D74 (role detection), D75 (team detection), D79 (detectionInterval config)
**Exploration:** quick
**Status:** captured

## D81: Swarm event types — 6 new CaseHubEventType values

**Choice:** Six new `CaseHubEventType` values in three groups:

**Role events:**
- `SWARM_ROLE_EMERGED` — a new role cluster detected. Metadata: `roleId` (generated), `memberAgents`, `dominantFeatures` (top-3 features from centroid), `clusterSize`, `avgSimilarity`.
- `SWARM_ROLE_DISSOLVED` — a role cluster no longer meets minimum membership. Metadata: `roleId`, `previousMembers`, `lifetimeCycles`.
- `SWARM_ROLE_SHIFT` — an agent moved from one role cluster to another. Metadata: `agentId`, `fromRoleId`, `toRoleId`, `similarityToNewRole`.

**Team events:**
- `SWARM_TEAM_FORMED` — a team affinity cluster detected. Metadata: `teamId` (generated), `memberAgents`, `dominantRelations` (which relation types dominate), `avgAffinity`.
- `SWARM_TEAM_DISSOLVED` — a team cluster no longer exists. Metadata: `teamId`, `previousMembers`, `lifetimeCycles`.

**Progress events:**
- `SWARM_PROGRESS` — progress scores changed significantly or periodic report. Metadata: `explorationScore`, `consensusScore`, `stabilityScore`, `cycle`.

Convergence/specialization (two clusters merging/splitting) are derivable from EMERGED/DISSOLVED pairs in the event stream — no dedicated event types.

**Alternatives:**
- 8 events (add ROLE_CONVERGED, ROLE_SPECIALIZED) — derivable from the 6-event set
- 4 events (drop ROLE_SHIFT, TEAM_DISSOLVED) — loses per-agent tracking and team lifecycle

**Sources:** D68 (stigmergy event patterns), D54 (convergence events), engine#1112
**Depends on:** D74 (role detection), D75 (team detection), D77 (progress tracking)
**Exploration:** quick
**Status:** captured

## D82: Module placement — extend existing stigmergy packages

**Choice:** Swarm types extend the existing stigmergy packages, not new packages:

| Component | Module | Package |
|-----------|--------|---------|
| `SwarmConfig`, `RoleDomainWeights` | api | `io.casehub.api.model.stigmergy` |
| `BehavioralFingerprint`, `DetectedRole`, `SwarmProgress` | api | `io.casehub.api.model.stigmergy` |
| `MetricsSpace` | api | `io.casehub.api.engine` |
| `RoleTracker` | runtime-core | `io.casehub.engine.internal.stigmergy` |
| `TeamDetector` | runtime-core | `io.casehub.engine.internal.stigmergy` |
| `SwarmProgressTracker` | runtime-core | `io.casehub.engine.internal.stigmergy` |
| `DefaultMetricsSpace` | runtime-core | `io.casehub.engine.internal.stigmergy` |
| New `CaseHubEventType` values | api | `io.casehub.api.event` (existing enum) |

No new packages. Swarm is an extension of stigmergy — the package structure reflects this. MetricsSpace follows the facet pattern in `api/engine` alongside SignalSpace, InterestSpace, NeighborSpace, RuleSpace.

**Alternatives:**
- New `api/model/swarm` + `runtime-core/internal/swarm` packages — clearer separation but swarm IS stigmergy, creating new packages implies a distinction that doesn't exist architecturally.

**Sources:** D23 (per-domain package pattern), D70 (stigmergy package structure)
**Depends on:** D73 (swarm extends stigmergy)
**Exploration:** quick
**Status:** captured

## D83: Pipeline decomposition — CaseEvaluationPipeline with phase handlers

**Choice:** Extract the five evaluation phases from `CaseContextChangedEventHandler` into a `CaseEvaluationPipeline` composed of phase handlers. Each phase implements a common interface receiving a `CaseEvaluationContext(CaseInstance, CaseContext, CaseDefinition)` and returning phase-specific results. Phase handlers: `BindingDispatchPhase` (existing `rules()` method), `GoalEvaluationPhase` (existing `goals()` method), `ObservationPhase` (existing `observations()` method), `LocalRulePhase` (existing `localRules()` method), `ConvergenceDetectionPhase` (existing `convergenceDetection()` method). The handler delegates to the pipeline. Each phase class owns only the dependencies it needs — BindingDispatchPhase takes the dispatch-related dependencies, ObservationPhase takes the observation registries, etc. The handler's constructor shrinks from 38 parameters to the pipeline + a few handler-level concerns (eventDispatcher, evaluationSerializer, quiescenceTracker).

**Alternatives:**
- Keep monolithic handler — current state with 38 constructor parameters, all 5 phases in one class. Difficult to test individual phases in isolation, hard to reason about which dependencies serve which concern.
- Partial extraction (only new phases) — extract ObservationPhase, LocalRulePhase, ConvergenceDetectionPhase from hive-mind; keep existing rules() and goals() in the handler. Inconsistent — two phases in the handler, three extracted. No clear boundary.

**Rationale:** The handler has 38 constructor parameters and 5 sequential phases with distinct dependency sets. The `evaluateAndDispatch()` method already calls five named methods sequentially — these ARE the phases. Extracting them reduces per-class complexity and improves testability. Each phase is testable in isolation with only its relevant dependencies. The pipeline structure makes the evaluation order explicit in the type system rather than implicit in method call order.

**Trade-offs:** Additional indirection — `evaluateAndDispatch()` delegates to a pipeline instead of calling methods directly. Acceptable — the abstraction boundary is already implicit in the five named methods. One shared CaseEvaluationContext object instead of repeating parameters across method signatures.

**Sources:** `CaseContextChangedEventHandler.java:97-235` (constructor with 38 parameters), `CaseContextChangedEventHandler.java:257-308` (evaluateAndDispatch calling 5 phases)
**Depends on:** D5 (observation phase), D42 (local rule phase), D45 (convergence detection phase)
**Exploration:** quick (surfaced by R1-03)
**Status:** revised — ADR-R1-08: corrected constructor parameter count from 34 to 38 (26 regular fields + 5 Instance<> fields + 7 additional)

## D84: Provisioning trigger — Signal-based consensus

**Choice:** Agents deposit provisioning signals (e.g., `swarm:need-capacity`) through the existing pheromone model when they detect workload pressure. When the signal reaches consensus (multiple agents reinforcing the same signal), the engine triggers provisioning. Signal metadata carries capability tags and preferred model.

**Alternatives:**
- Rule-based (explicit `RuleAction.Provision`) — more explicit control but couples provisioning logic into per-agent rule definitions, less emergent
- Metric-threshold (SwarmProgressTracker auto-triggers on exploration pace drop) — fully automated but less agent-driven; the swarm doesn't "decide", the engine infers

**Rationale:** Uses the existing signal/pheromone infrastructure — decay prevents stale requests, reinforcement confirms real need, consensus prevents single-agent noise from triggering expensive operations. No new trigger mechanism needed, just a semantic convention on signal names and metadata.

**Trade-offs:** Consensus-based triggering is slower than direct rule actions — there's inherent latency between "agent detects pressure" and "enough agents agree." Acceptable for provisioning which is a heavyweight operation that shouldn't fire on transient spikes.

**Sources:** `SignalRegistry.java:37` (deposit/reinforce/decay), `SignalRegistry.consensusSignals()` (multi-source detection), `SwarmConfig.java` (configuration), issue #1113
**Exploration:** quick
**Status:** captured

## D85: Capability resolution — Capability-tagged signals via eidos

**Choice:** The provisioning signal carries metadata (capability tags, preferred model query). The engine resolves these against eidos `AgentDescriptor` registry to find or compose a matching agent. The swarm says "I need an analyst" not "provision agent-config-xyz."

**Alternatives:**
- Explicit agent type (signal metadata names a specific AgentDescriptor ID) — tighter coupling but guarantees exact agent configuration
- Engine-inferred (engine analyzes role/team gaps and decides) — maximum engine autonomy but least agent control

**Rationale:** Capability-tagged signals let the swarm express intent without coupling to specific agent configurations. Eidos already has the registry and matching infrastructure (`AgentCapability`, `CapabilityHealth`, `ModelQuery`). The platform resolves the "what kind of agent" question — the swarm only needs to express "what capabilities are needed."

**Trade-offs:** Indirect resolution means the provisioned agent might not exactly match what the swarm expected. The eidos registry might not have a matching descriptor. Both are acceptable — the tunable integration model (D86) handles mismatches gracefully.

**Sources:** `AgentCapability` (eidos-api), `CapabilityHealth` (eidos-api), `ModelQuery` (platform-api), `AgentDescriptor` (eidos-api), issue #1113
**Depends on:** D84 (signal carries metadata for resolution)
**Exploration:** quick
**Status:** captured

## D86: Agent membership — Three-axis tunable integration model with CBR learning

**Choice:** Newly provisioned agents integrate through a tunable model with three independently configurable axes: (1) **Bootstrap richness** — how much swarm state context the engine provides (0.0=none → 1.0=full state transfer), (2) **Integration delay** — how many cycles before the agent's fingerprint influences role/team detection (0=immediate → N=extended observation), (3) **Self-determination** — how much the agent decides its own interests/signals/rules vs. having them pre-configured (0.0=engine-assigned → 1.0=fully autonomous discovery). Optional guardrails on each axis. CBR records provisioning outcomes and adjusts tuning over time. Agents can also propose tuning adjustments based on their own experience (feed-forward for #1114).

**Alternatives:**
- Coordinator-managed join only — engine slots agent in, treats it as interchangeable capacity; kills emergence
- Staged onboarding only — engine-imposed PROBATIONARY state; readiness is about understanding, not time
- Self-registration only — purest autonomy but cold-start is fatal in fast-moving swarms

**Rationale:** No single approach is correct for all situations. Emergency swarms need high bootstrap + zero delay. Research swarms need high autonomy + some delay. The right settings are contextual and situational — something we won't get right on day one. Building tunable infrastructure with CBR learning lets the system evolve toward optimal settings for each context. Agent self-tuning enables #1114 self-improvement.

**Trade-offs:** More configuration surface than any single approach. The tuning axes add complexity to the provisioning config model. Acceptable — the alternative is a rigid system that works well in one scenario and poorly in others. The complexity is in configuration, not in runtime logic — each axis maps to a simple behavioral change.

**Defaults:** Bootstrap richness: 0.7 (high — new agents get most swarm context by default, reducing cold-start latency). Integration delay: 3 cycles (brief observation period before fingerprint influences detection — enough to avoid noise without excessive delay). Self-determination: 0.5 (balanced — engine pre-configures basic interests/rules from the provisioning signal's capability metadata, agent can override). These defaults prioritize operational safety (fast integration with moderate autonomy) over emergence (high autonomy with extended observation). CBR will tune from here.

**Sources:** `StigmergyCoordinator.agentJoined()` (membership lifecycle), `RoleTracker` (fingerprinting), `TeamDetector` (affinity), `SwarmProgressTracker` (stability impact), `CbrRetrievalService` (outcome learning), issue #1113
**Depends on:** D84 (signal triggers provisioning), D85 (capability resolution determines what agent), D73 (swarm extends stigmergy)
**Exploration:** deep-analysis (first-principles exploration of three approaches and hybrid)
**Status:** revised — ADR-R1-14: specified explicit defaults for three axes (bootstrap=0.7, delay=3 cycles, self-determination=0.5)

## D87: Budget enforcement — Layered caps

**Choice:** Three enforcement layers, all must agree before provisioning proceeds: (1) `SwarmConfig.maxSwarmSize` — per-case hard cap on total active agents (declared + provisioned combined), (2) `ProvisionBudget` nested in SwarmConfig with `maxProvisions` (total lifetime provisions), `maxConcurrent` (simultaneous active provisioned agents — distinct from `maxSwarmSize` which caps the total including declared agents), `cooldownCycles` (minimum cycles between provisions), (3) external `DispatchBudget` SPI for cross-case capacity coordination (e.g., claudony session pool limits). Clarification: `maxSwarmSize` on SwarmConfig is the overall population cap (all agents, whether declared in YAML or dynamically provisioned). `ProvisionBudget.maxConcurrent` is a sub-cap on dynamically provisioned agents specifically. Both are enforced — a provisioning request must satisfy `activeAgents < maxSwarmSize` AND `activeProvisionedAgents < maxConcurrent`.

**Alternatives:**
- Single cap (just maxSwarmSize) — simple but no rate limiting or cross-case coordination; aggressive swarm exhausts all capacity in one burst
- Token-based (regenerating provisioning tokens) — elegant but adds a new resource tracking concept

**Rationale:** Provisioning is expensive (real compute, real cost). Single cap doesn't prevent burst provisioning. Token-based is novel complexity. Layered caps use existing patterns: maxSwarmSize is already in SwarmConfig, DispatchBudget SPI already exists for external capacity. ProvisionBudget adds rate limiting (cooldown) and lifetime caps without new concepts.

**Trade-offs:** Three-layer check on every provisioning decision adds latency. Negligible — provisioning itself is orders of magnitude slower than the budget check.

**Sources:** `SwarmConfig.maxSwarmSize` (existing field), `BudgetConfig` (existing pattern), `DispatchBudget` (existing SPI), `BudgetEnforcer` (existing enforcement), issue #1113
**Depends on:** D84 (budget checked after signal consensus triggers provisioning)
**Exploration:** quick
**Status:** revised — ADR-R1-14: clarified maxSwarmSize vs ProvisionBudget.maxConcurrent relationship

## D88: De-provisioning — Idle detection + signal decay

**Choice:** Track agent activity via ActivityTracker. When an agent's activity rate drops below a configurable threshold AND no provisioning signals reinforce its role, the engine terminates it via `WorkerProvisioner.terminate()`. Pheromone decay naturally clears stale capacity requests — if no one reinforces the signal that provisioned this agent, the justification decays away. De-provisioning events emitted for audit and CBR.

**Alternatives:**
- Departure rules (agents fire RuleAction.Leave when they detect they're no longer needed) — requires agents to have good self-awareness; unreliable for rule-based agents
- Swarm vote (agents deposit 'reduce-capacity' signals targeting idle agents) — democratic but slower; adds delay to scale-down

**Rationale:** Idle detection is observable and objective — activity rates are already tracked. Signal decay handles the "why was this agent provisioned" question naturally — if the original need has decayed, the agent's justification has too. Combining both prevents premature termination (agent is idle but still needed) and delayed termination (agent is active but original need is gone).

**Trade-offs:** Idle threshold is a tuning parameter that varies by workload type. An agent processing rare high-value events might appear idle most of the time. Mitigated by making the threshold configurable per SwarmConfig and learnable via CBR.

**Sources:** `ActivityTracker` (rate computation), `SignalRegistry` (decay mechanics), `WorkerProvisioner.terminate()` (existing termination SPI), issue #1113
**Depends on:** D84 (provisioning signals whose decay informs de-provisioning), D87 (de-provisioning frees budget capacity)
**Exploration:** quick
**Status:** captured

## D89: Provisioning orchestration — SwarmProvisioner bean

**Choice:** New `@ApplicationScoped` bean `SwarmProvisioner` in `runtime-core/stigmergy` that coordinates the full provisioning flow: validate budget → resolve capabilities via eidos → build bootstrap context (per D86 tuning) → call `WorkerProvisioner.provision()` → `StigmergyCoordinator.agentJoined()` → emit ledger-integrated audit events. Clean extension points for blocks-side LLM reasoning (SPI hook where blocks can inject reasoning into provisioning decisions). Full CBR loop: query past outcomes for similar swarm states, record current outcome.

**Alternatives:**
- CaseContextChangedEventHandler inline — keeps everything in one place but makes the already-large handler even larger (34+ constructor params, per D83)
- RuleAction extension (RuleAction.Provision) — reuses rule infrastructure but means provisioning is always rule-triggered, limiting signal-based approaches

**Rationale:** Provisioning is a distinct responsibility with its own dependencies (WorkerProvisioner, CapabilityHealth, CbrRetrievalService, budget enforcement). Inlining it into the handler violates single responsibility and increases constructor size. The SwarmProvisioner bean is injected via `Instance<>` (same pattern as StigmergyCoordinator) — available when stigmergy is active, absent otherwise.

**Trade-offs:** New bean adds to the CDI graph. Acceptable — the alternative is a 38+ parameter handler constructor. SwarmProvisioner has clear boundaries: it's called from the convergence detection phase and delegates to existing SPIs.

**Sources:** `StigmergyCoordinator` (pattern for `Instance<>` injection), `WorkerProvisioner` (existing provisioning SPI), `CbrRetrievalService` (outcome learning), `RuntimeBeans.java` (CDI wiring pattern), issue #1113
**Depends on:** D84 (signal consensus triggers SwarmProvisioner), D85 (capability resolution), D86 (integration model), D87 (budget enforcement), D88 (de-provisioning), D83 (handler extraction — SwarmProvisioner avoids further handler bloat)
**Exploration:** quick
**Status:** captured

## D90: Audit trail — Ledger-integrated provisioning events

**Choice:** Provisioning events flow through the existing `LedgerTraceIdProvider` / `causedByEntryId` chain. Each provision/terminate gets a ledger entry with causal linkage back to the signal consensus that triggered it. Engine-internal `CaseHubEventType` events also emitted for real-time swarm awareness (SWARM_PROVISION_REQUESTED, SWARM_PROVISION_COMPLETED, SWARM_AGENT_TERMINATED). CBR records include the full provisioning context (tuning axes, swarm state at time of decision, outcome).

**Alternatives:**
- Engine-internal events only — cheaper, no ledger dependency, sufficient for CBR but no compliance/audit trail
- Both (ledger + events) — redundant channels for the same information

**Rationale:** Provisioning creates real compute resources with real cost. Audit trail is non-negotiable for compliance and cost attribution. The existing ledger infrastructure already handles causal linkage via `ProvisionResult.causedByEntryId()`. Engine events provide the real-time swarm awareness needed for detection algorithms. Both channels serve different consumers.

**Trade-offs:** Ledger writes add latency to the provisioning flow. Acceptable — provisioning is already a heavyweight operation (spinning up compute). The ledger write is negligible relative to the actual provisioning time.

**Sources:** `LedgerTraceIdProvider` (existing trace infrastructure), `ProvisionResult.causedByEntryId()` (existing causal linkage), `CaseHubEventType` (existing event enum), issue #1113
**Depends on:** D89 (SwarmProvisioner emits the events)
**Exploration:** quick
**Status:** captured

## D91: Scope — Engine-complete with blocks hooks and full CBR loop

**Choice:** Full provisioning mechanism works in-engine with rule-based agents. Clean SPI extension points where blocks can later inject LLM reasoning into provisioning decisions (e.g., LLM evaluates bootstrap context, LLM reasons about capability gaps). Full CBR loop included: query past provisioning outcomes for similar swarm states during provisioning decisions, record current provisioning context + outcome for future retrieval. No blocks repo dependency in this issue.

**Alternatives:**
- Engine-only (no blocks hooks, no CBR) — simplest scope but requires retrofit
- Include blocks integration — builds blocks-side LLM reasoning; significantly larger scope, cross-repo

**Rationale:** The CBR loop is essential because provisioning tuning (D86) is explicitly designed to be learned, not hardcoded. Without CBR from day one, the tuning axes have no feedback mechanism. Blocks hooks are low-cost extension points (SPI interfaces) that prevent API-breaking changes when blocks integration arrives.

**Trade-offs:** Larger scope than mechanism-only. CBR integration requires understanding CbrRetrievalService's query model and adapting it for provisioning scenarios. Acceptable — the alternative is building a tunable system with no way to learn.

**Sources:** `CbrRetrievalService` (existing CBR infrastructure), `blocks#285` (future LLM coordination), issue #1113
**Depends on:** D86 (CBR learns tuning axes), D89 (SwarmProvisioner orchestrates CBR queries)
**Exploration:** quick
**Status:** captured

## D92: Improvement identification model — signals trigger goals

**Choice:** Two-layer architecture. Signals are sensors (collective consensus), goals are actuators (execution lifecycle). Agents deposit `improvement:*` signals when they observe issues. Signal consensus validates the observation (same pattern as `swarm:need-capacity` in #1113). Once consensus is reached, GoalFormationService.propose() creates a SELF_IMPROVEMENT goal with the improvement context. The goal is dispatched through the normal goal lifecycle.

**Alternatives:**
- Signals alone — no prioritisation, tracking, lifecycle, or learning mechanism; would reinvent half the goal system
- Goals alone — no consensus mechanism; single agent's opinion drives improvement instead of collective intelligence

**Rationale:** Sensing and acting are fundamentally different capabilities. An agent good at noticing code smells might not be good at fixing them. Signals let any agent contribute observations; goals let the best-suited agent execute. Each layer evolves independently — blocks can enhance sensing (LLM-powered observation, blocks#284) or execution (LLM-powered implementation, blocks#285) separately. Neocortex makes both layers smarter: sensing remembers "we tried X before, it was rejected," execution remembers "this approach worked for module Y." Directly feeds into goal creation/management epic (#800) — improvement goals ARE goals, so all of Sub-epic C (formation, revision, abandonment, priority evolution) applies.

**Trade-offs:** More complex than either layer alone. Signal→goal bridge is new infrastructure. Acceptable — the alternative is building parallel systems or leaving gaps that #1115 and #800 would need to fill.

**Sources:** SwarmProvisioner.java:71 (signal consensus pattern), GoalFormationService.java:18, GoalFormationStrategy.java:20, issue #1114, issue #800 (goal lifecycle epic), arXiv:2603.28990 (role emergence), SwarmWorld cognition/consequence split
**Exploration:** deep-analysis
**Status:** captured

## D93: Execution model — case-as-improvement

**Choice:** Each improvement spawns a child case via SubCaseBinding. The improvement IS a case — bindings for introspect, implement, submit-pr, review, integrate. Each step is a worker with its own capability. Full audit trail via EventLog. Workers are independently replaceable — blocks can substitute any step with an LLM-powered agent.

**Alternatives:**
- Direct worker dispatch — single monolithic 'self-improver' worker handles entire lifecycle; can't evolve steps independently, no per-step audit trail, can't parallelise
- Workflow orchestration via WorkOrchestrator.submitAndWait() — explicit orchestration rather than choreography; doesn't use case model strengths, less extensible via bindings

**Rationale:** The case model already provides everything needed: bindings with triggers, per-step worker dispatch, EventLog audit, SubCaseBinding for spawning child cases, lifecycle management (RUNNING/WAITING/COMPLETED). Each step is a capability — the routing strategy selects the best worker. Trust model applies per step. New steps can be added as new bindings without code changes.

**Trade-offs:** More complex than a single worker. Requires a case definition template. Acceptable — the complexity is managed by existing infrastructure, and independent step evolution is a hard requirement for extensibility.

**Sources:** SubCaseBinding (DESIGN.md:157-177), CaseContextChangedEventHandler, WorkerScheduleEventHandler, issue #1114
**Depends on:** D92 (goals trigger the improvement case)
**Exploration:** quick
**Status:** captured

## D94: Engine implementation depth — full working implementations

**Choice:** Engine provides working implementations of each improvement step. Rule-based workers use REST/GraphQL/MCP APIs to interact with repos and DevTown. Same pattern as every other hive mind issue: engine = working rule-based foundation, blocks = LLM enhancement. Workers handle structured, API-mediated, rule-based improvement categories.

**Alternatives:**
- SPI stubs + blocks implements — breaks the established pattern; "engine-complete" means it works, not "has interfaces"; self-improvement system does nothing without blocks
- Thin engine + callback hooks — fewer new SPIs but callbacks are less typed than dedicated workers; same fundamental problem as stubs

**Rationale:** Every issue in this epic provides working engine implementations: SignalRegistry, RoleTracker, SwarmProvisioner all work standalone. Self-improvement follows the same rule. REST/GraphQL/MCP APIs (coming to all repos) make interaction clean and proper, not shell hacking. WorkerProvisioner already calls external systems — this is the same kind of integration.

**Trade-offs:** Larger implementation scope than stubs. Workers need real API integrations. Acceptable — rule-based improvements (dependency bumps, lint fixes, coverage analysis) are useful standalone and prove the architecture before blocks adds LLM intelligence.

**Sources:** WorkerProvisioner.java:34 (external system integration pattern), SwarmProvisioner.java:37 (working engine implementation pattern), issue #1114
**Depends on:** D93 (case model defines the steps workers implement)
**Exploration:** deep-analysis
**Status:** captured

## D95: Improvement taxonomy — operational AND capability improvements with research

**Choice:** Two-dimensional improvement taxonomy with a research-driven capability growth loop.

**Operational improvements** target code hygiene and infrastructure: dependency updates, lint/checkstyle fixes, test coverage gaps, CI failure triage, OpenRewrite-style code recipes. Detected via rule-based metrics (coverage reports, lint reports, dependency staleness, CI logs). Executed by engine rule-based workers via REST/GraphQL/MCP APIs.

**Capability improvements** target the swarm's cognitive and execution abilities across multiple dimensions: cognitive (reasoning, problem-solving, inference), execution control (planning strategies, coordination patterns, convergence detection), perception (observation patterns, signal interpretation), social (team formation, role emergence, agent coordination), knowledge (domain understanding, CBR enrichment, memory structures), and tools (new MCP tools, API integrations, expanded capabilities).

Capability improvements have a **research-driven growth loop**: the swarm doesn't just look inward at its own metrics — it actively studies the outside world. It searches the internet, Google Scholar, arXiv for relevant techniques, algorithms, and approaches. It reads papers, evaluates applicability to the CaseHub architecture, synthesises findings, and builds implementation plans based on what it discovers. This is what human engineers do — read papers, attend conferences, study new techniques — but at swarm scale and continuously.

The full capability improvement cycle: (1) identify opportunity — internal metrics OR proactive "what would make me better at X?"; (2) research — search internet, Google Scholar, arXiv for techniques; (3) analyse — evaluate applicability, synthesise findings; (4) plan — design the architectural improvement with delivery plan; (5) implement; (6) submit-pr; (7) review via DevTown; (8) integrate; (9) monitor outcome.

The case template is category-agnostic with conditional bindings. The research+analyse phase fires for capability improvements (trigger: `improvementType == 'capability'`) and skips for operational improvements (dependency bumps don't need literature review). This is standard case binding behaviour — trigger conditions control which steps execute.

Engine provides: (a) search infrastructure — API calls to search engines, paper repositories, structured data fetching (rule-based); (b) detection metrics for both dimensions; (c) the full case lifecycle. Blocks provides: (d) paper understanding and synthesis (LLM); (e) architectural analysis and design (LLM); (f) capability implementation intelligence (LLM).

Five initial engine rule-based worker categories for operational improvements: (1) DependencyUpdateWorker, (2) LintFixWorker, (3) CoverageGapWorker, (4) CITriageWorker, (5) RecipeWorker. Capability improvement workers are blocks-provided with engine search infrastructure support.

**Alternatives:**
- Operational only — the swarm fixes lint but never gets smarter; misses the core self-improvement promise
- Capability without research — the swarm only looks inward; never discovers techniques it doesn't already know
- Research without structured taxonomy — no rule-based engine story; everything requires LLM

**Rationale:** Self-improvement that doesn't include self-directed growth is just maintenance. The swarm should do what engineers do: identify weaknesses, study the field, find better approaches, design improvements, build them, verify they work. The research loop is what separates a self-improving system from a self-maintaining one.

The three-axis integration model (D86) — specifically the self-determination axis — is about growing the swarm's autonomy. Capability improvement with research is growing along that axis: the swarm decides WHAT to get better at, researches HOW, and executes the growth plan.

Connection to goal epic (#800): goal formation discovers capability gaps AND research opportunities ("this paper describes a technique that would improve our coordination"). Goal revision adjusts growth direction based on outcomes. Goal abandonment drops research directions that don't pan out. The goal lifecycle IS the growth lifecycle.

**Trade-offs:** Research-driven capability improvement is the most ambitious scope. The research infrastructure (search APIs) is engine-complete; the understanding (paper synthesis, architectural design) needs blocks. In engine-only mode: operational improvements work fully, capability improvement research is available as data but synthesis waits for blocks. The architecture supports graceful degradation.

**Sources:** Issue #1114, issue #800 (goal lifecycle), D86 (three-axis integration model), Darwin Gödel Machine (SWE-bench 20%→50%, autonomously discovered better tools), SICA (17%→53%), AlphaEvolve (0.7% of Google's worldwide compute recovered), Self-Evolving Agents Survey (arXiv:2507.21046)
**Depends on:** D94 (workers are full implementations), D92 (signals detect both operational and capability gaps), D93 (case template supports conditional research bindings)
**Exploration:** deep-analysis
**Status:** revised — expanded to include research-driven capability growth loop

## D96: DevTown review gate — standard code-review capability

**Choice:** DevTown is modelled as a standard worker with `code-review` capability. No dedicated CodeReviewGate SPI. Routing selects DevTown the same way it selects any worker. Human reviewers or alternative review systems register with the same capability. Trust model applies to the reviewer. The improvement case binding triggers on `pr-submitted` signal.

**Alternatives:**
- Dedicated CodeReviewGate SPI with sealed ReviewOutcome — more explicit safety guarantee but creates a special case outside the worker model; doesn't compose with routing or trust

**Rationale:** DevTown is operational with an API. It's a worker that does code review — treating it as such reuses the entire routing, trust, and capability infrastructure. No special SPI needed. Other review mechanisms (human, alternative review systems) slot in identically by registering the same capability. The mandatory review constraint is enforced by the improvement case lifecycle — the integrate binding cannot fire without a review outcome in the case context.

**Trade-offs:** The review requirement is enforced at two levels: (1) case lifecycle — the integrate binding's `when` condition requires a review outcome in the case context, and (2) defensive hard gate — the integration worker itself performs a pre-flight check against the case's EventLog, confirming a `REVIEW_COMPLETED` event exists with a passing verdict before proceeding. The hard gate is belt-and-suspenders: even if a lifecycle bug allows the integrate binding to fire without a review, the worker refuses to execute. `IllegalStateException` with message: "Integration blocked: no passing review record found for improvement case <caseId>." This is not an SPI — it's an internal check in the integration worker.

**Sources:** DevTown (operational API), AgentRoutingStrategy, TrustWeightedAgentStrategy, ComposableAgentRoutingStrategy, issue #1114
**Depends on:** D93 (review is a step in the improvement case)
**Exploration:** quick
**Status:** revised — ADR-R1-15: added defensive hard gate at integration step (EventLog pre-flight check) alongside lifecycle enforcement

## D97: Safety model — layered budget enforcement

**Choice:** ImprovementBudget record with layered enforcement, following the same pattern as ProvisionBudget + DispatchBudget in SwarmProvisioner. Fields: maxConcurrent, maxPerDay, cooldownMinutes, allowedRepos, deniedPaths, requireReview (default true), requireGreenCI (default true), maxPRSize (lines changed). Enforced by ImprovementBudgetEnforcer before the improvement case is spawned. **Structural self-modification denial:** `deniedPaths` includes by default (not just as configuration): any path matching `**/ImprovementBudget*`, `**/ImprovementBudgetEnforcer*`, `**/SafetyConfig*`, and the improvement case template definition. These defaults are hardcoded in `ImprovementBudgetEnforcer` and cannot be overridden by configuration — they are structural constraints, not policy choices. User-configured `deniedPaths` are additive to these structural denials. This ensures the self-improvement system cannot modify its own safety constraints regardless of configuration.

**Alternatives:**
- Trust-gated only — existing trust maturity model restricts scope by earned trust; organic but no hard limits on volume or blast radius
- Scope manifests — static per-worker-type scope declarations; simpler but not adaptive to runtime conditions

**Rationale:** Layered budget enforcement is a proven pattern in this codebase (ProvisionBudget, DispatchBudget). Multiple independent limits prevent unbounded behaviour even when individual checks pass. Path restrictions and repo scoping provide defence-in-depth alongside DevTown review. The budget record is configurable per case via SwarmConfig, following the same structure as self-provisioning.

**Trade-offs:** Configuration surface grows. Defaults must be conservative — improvements are opt-in and constrained by default. The budget can be relaxed as the system proves itself, which connects to the trust model naturally.

**Sources:** ProvisionBudget (api/model/stigmergy), SwarmProvisioner.java:86-159 (budget enforcement pattern), DispatchBudget, issue #1114
**Depends on:** D92 (budget gates goal formation), D93 (budget checked before case spawn)
**Exploration:** quick
**Status:** revised — ADR-R1-15: added structural self-modification denial (hardcoded deniedPaths for safety infrastructure, non-overridable)

## D98: Outcome tracking — three-layer event-sourced model

**Choice:** All three layers, composed via event sourcing. (1) Structured EventLog record is the source of truth — typed ImprovementOutcome with PR status, CI delta, coverage delta, performance delta. (2) Signals projected from the record — `improvement:outcome:positive`, `improvement:outcome:regression`, etc. — provide real-time swarm notification that decays naturally. (3) CBR trace projected from the record — stored in neocortex for historical learning, retrievable by future improvement cycles.

**Alternatives:**
- Any single layer alone — each serves a different consumer at a different timescale; omitting one leaves a gap

**Rationale:** Each layer serves a different consumer at a different timescale. Structured records: case lifecycle (GoalRevisionEvaluator, GoalAbandonmentEvaluator, ImprovementBudget tracking). Signals: evaluation cycle (swarm adjusts behaviour immediately). CBR: historical (future improvements retrieve similar past outcomes). The pattern mirrors CaseLedgerEventCapture — structured event is the source, projections serve different consumers. All three layers are load-bearing for #1115 (continuous evolution loop) and #800 (goal lifecycle management).

**Trade-offs:** Three projections from one event is the most complex approach. Mitigated by the fact that each projection uses existing infrastructure (EventLog, SignalRegistry, CbrRetrievalService) — no new data stores.

**Sources:** CaseLedgerEventCapture (event-sourcing pattern), SignalRegistry, CbrRetrievalService, issue #1114, issue #1115, issue #800
**Depends on:** D92 (signals are the real-time layer), D93 (case lifecycle produces the structured record)
**Exploration:** deep-analysis
**Status:** captured

## D99: Goal integration — GoalKind.SELF_IMPROVEMENT

**Choice:** New GoalKind.SELF_IMPROVEMENT value. GoalFormationEvaluator applies improvement-specific logic: improvement signals trigger proposals, scoped by ImprovementBudget. GoalRevisionEvaluator uses structured outcome records to adjust priority: repeated failures deprioritise category. GoalAbandonmentEvaluator detects futility: 3 rejected PRs → abandon direction; regression detected → abandon + rollback. Existing goal-capability mapping (#860) applies: improvement goals map to improvement capabilities.

**Alternatives:**
- Standard goals with tagged metadata — simpler but evaluators can't distinguish improvement goals from regular agent goals without inspecting metadata; loses type safety

**Rationale:** GoalKind is the discriminator the existing goal evaluators use. A dedicated kind lets the evaluators apply improvement-specific logic without metadata inspection. The goal lifecycle machinery (formation, revision, abandonment, priority evolution from #800 Sub-epic C) applies directly. This is the integration point between self-improvement and the goal management epic.

**Trade-offs:** Enum extension — straightforward. The improvement-specific evaluator logic is new code but follows existing patterns.

**Sources:** GoalFormationEvaluator.java:47, GoalRevisionEvaluator.java:51, GoalRevisionAction.java:18, AgentGoalCompletionMarker.java:28, issue #860, issue #800
**Depends on:** D92 (goals are the execution layer), D98 (outcome records feed goal evaluators)
**Exploration:** quick
**Status:** captured

## D100: Scope boundary — #1114 full single-shot, #1115 continuous loop

**Choice:** #1114 delivers the complete single-shot cycle: improvement signal types, signal→goal formation bridge, ImprovementBudget + enforcer, improvement case template (hybrid Java/YAML), five rule-based worker categories, DevTown as code-review capability worker, outcome tracking (all three layers), GoalKind.SELF_IMPROVEMENT. #1115 adds: standing directive / continuous trigger, outcome→detection feedback loop, prioritisation across concurrent improvements, data autophagy prevention, growth direction (swarm decides what to improve), rollback on regression.

**Alternatives:**
- #1114 mechanism only / #1115 all lifecycle — thinner #1114 but #1115 becomes very large and tightly coupled
- #1114 everything except growth direction — #1115 becomes too thin to justify as a separate issue

**Rationale:** The single-shot cycle is the natural unit of completeness — it can be tested end-to-end (signal → goal → case → workers → review → outcome). The continuous loop adds autonomous agency (the swarm decides WHEN and WHAT) which is a qualitatively different concern. Monitoring/outcome recording belongs to #1114 because it's part of verifying the single cycle worked. The feedback loop that feeds outcomes back into detection belongs to #1115 because it's the continuous aspect.

**Trade-offs:** #1114 is a large issue. Mitigated by batched implementation — workers can be implemented incrementally, architecture works with any subset.

**Sources:** Issue #1114, issue #1115, issue #1104 (epic structure)
**Depends on:** D92-D99 (all prior decisions scope #1114)
**Exploration:** quick
**Status:** captured

## D101: Case template format — hybrid Java/YAML (existing convention)

**Choice:** Hybrid — core improvement case lifecycle in Java, YAML as peer representation per the DSL parity principle. Not a novel decision — this is the established CaseHub convention. Java provides the typed API surface, YAML provides the user-configurable case definition. DSL extensions where necessary.

**Alternatives:** None — this is the existing convention, not a design choice.

**Rationale:** CLAUDE.md: "DSL parity: YAML and Java are peer representations." Every case definition in the platform follows this pattern.

**Trade-offs:** None beyond the usual hybrid approach maintenance.

**Sources:** CLAUDE.md (DSL parity principle), CaseDefinition.yaml, yaml-record-mappings.yaml
**Depends on:** D93 (case-as-improvement defines what needs Java/YAML representation)
**Exploration:** quick (existing convention)
**Status:** captured

## D102: Cross-agent observation isolation — per-agent observations, shared signals

**Choice:** Observations are per-agent: `observationRegistry.getObservations(caseId, agentId)` returns only the calling agent's observations. Signals are shared: all agents perceive the same signal set. InterestLandscape is shared: all agents see the collective interest aggregate. Neighbors are per-agent queries: each agent sees its own proximity context.

This creates an architectural requirement: signals are the inter-agent communication channel for observation-derived findings. If Agent A detects "suspicious pattern" via an observer, Agent B can only learn about it if Agent A deposits a signal. Observations are local perception (what I detected); signals are shared announcement (what I want others to know).

**Alternatives:**
- Shared observations (all agents see all observations) — breaks per-agent specialization. Agents with different interests would be overwhelmed by observations from domains they don't care about. The observation registry would need cross-agent filtering, which is what the signal layer already provides.
- Observation forwarding (agent A can explicitly share specific observations with agent B) — useful but adds a targeted communication primitive outside the stigmergy model. Stigmergy is indirect coordination through environment modification, not direct agent-to-agent messaging.
- Per-agent with opt-in visibility (agents can mark observations as "public") — hybrid, but collapses the observation/signal distinction. A "public observation" IS a signal.

**Rationale:** The asymmetry is intentional and architecturally correct. Observations are the output of the perceive step — they represent what an individual agent's observers detected. Rules are the decide step — they process the agent's own observations alongside shared coordination state (signals, landscape). Signals are the act step — they announce findings to the environment. The perceive→decide→act cycle requires this asymmetry: if observations were shared, there would be no need for signals as a communication mechanism, and the stigmergy model collapses into a shared-memory model.

**Trade-offs:** Inter-agent communication requires an explicit signal deposit. An observation that should influence other agents must be "promoted" to a signal by the detecting agent's rules. This is additional rule complexity but matches the biological model — an ant that finds food must lay a pheromone trail (signal) to share the finding; other ants don't telepathically see the finding.

**Sources:** `ObservationRegistry.getObservations(caseId, agentId)` (per-agent), `SignalRegistry.perceive(caseId)` (shared), `ObservationContext.interestLandscape()` (shared), `NeighborSpace` queries (per-agent), D1 (observer SPI), D7 (observation materialization), D10 (signal storage), D14 (observation integration)
**Depends on:** D1, D7, D10, D14, D31
**Exploration:** quick (surfaced by ADR-R1-17 — made explicit from implicit per-agent isolation)
**Status:** captured

## D103: Improvement prioritisation — Drive system integration, not rigid hierarchy

**Choice:** Self-improvement prioritisation uses the existing Drive system (`blocks-core`, `io.casehub.blocks.agentic.social.drive`) rather than a rigid tier hierarchy. The Drive system models four competing axes (CURIOSITY, COMPETENCE, AFFILIATION, AUTONOMY) as balanced needs that compete dynamically, modulated by mood and personality. The **dominant drive** emerges from context — no axis has hardcoded priority over another.

Self-improvement maps onto the Drive axes:
- **COMPETENCE drive** — rises when quality/stability metrics degrade (CI failures, test regressions, coverage gaps, lint violations). The swarm feels "I need to get my house in order."
- **CURIOSITY drive** — rises when research opportunities appear, when success rates plateau, when new techniques are discovered. The swarm feels "I want to learn and grow."
- **AUTONOMY drive** — rises when the swarm's self-determination is constrained, when it encounters problems it can't handle. The swarm feels "I need to expand my capabilities."
- **AFFILIATION drive** — rises when coordination quality drops, when team coherence degrades. The swarm feels "I need to work better together."

`DriveOrchestrator.tick()` evaluates all four axes each cycle, `DriveComposer` modulates by mood and personality, and the `DriveProfile.dominantDrive()` determines what the swarm focuses on. Budget allocation is proportional to drive intensity — a high-intensity COMPETENCE drive (stability is failing) naturally draws more budget than a low-intensity CURIOSITY drive (everything's fine, let's explore). But neither is suppressed — the swarm can research while fixing stability, just with proportionally less allocation.

The engine provides `ImprovementDriveSource` implementations for each axis — rule-based evaluators that feed drive intensity from metrics:
- COMPETENCE source: CI status, test pass rate, lint violation count, coverage percentage → drive intensity
- CURIOSITY source: time since last research cycle, number of unexamined external signals, success rate trajectory → drive intensity
- AUTONOMY source: count of problem classes with repeated failures, trust score plateaus, capability gaps → drive intensity
- AFFILIATION source: team coherence score (from TeamDetector), coordination failure rate → drive intensity

These sources implement `DriveSource` (existing blocks SPI). The engine provides rule-based defaults; blocks can enhance with LLM-powered assessment.

`DriveGoalFormationStrategy` already exists and proposes goals from drive context. Self-improvement goals flow through this existing machinery: when a drive intensity exceeds `DriveConfig.changeThreshold()`, `DriveGoalFormationStrategy.propose()` is called with the drive context, and it proposes a SELF_IMPROVEMENT goal scoped to the drive axis. The budget enforcer (D97) checks `ImprovementBudget` limits before approving.

The neocortex goal cognition epic (#345) adds further sophistication: goal dependency graphs, affective valuation, multi-signal priority (urgency × importance × feasibility × affective valence), opportunity cost awareness, and goal-conditioned retrieval. As these capabilities land, self-improvement goals automatically benefit from them.

**Alternatives:**
- Rigid tier hierarchy (Maslow model) — stability always wins; capability growth only when base is healthy. Too rigid — a single flaky test shouldn't suppress all research. Doesn't match how the Drive system works.
- Flat priority — all improvements compete without any weighting. Misses the real signal that stability degradation should increase urgency.

**Rationale:** The Drive system already solves this problem. It models balanced competing needs where the dominant one emerges from context rather than being prescribed. Using it for self-improvement means: (1) no new prioritisation infrastructure, (2) improvement priorities respond to mood and personality (a cautious agent naturally prioritises stability; an exploratory agent naturally prioritises research), (3) as neocortex goal cognition (#345) lands, self-improvement goals get affective valuation, dependency tracking, and sophisticated prioritisation for free.

The key insight: the Drive system found that "one doesn't take priority over the other" — needs balance dynamically based on intensity, modulation, and context. A swarm with failing CI has high COMPETENCE drive intensity, which naturally dominates budget allocation. But if stability is fine, CURIOSITY and AUTONOMY drives compete for what the swarm works on next. This is more realistic and more extensible than rigid tiers.

**Trade-offs:** Depends on blocks-core for the Drive system. In engine-only mode (no blocks), `ImprovementBudgetEnforcer` falls back to proportional allocation based on raw signal counts — simpler but still responsive. The Drive system adds personality, mood, and narrative modulation that pure signal counting cannot provide.

**Sources:** DriveAxis.java, DriveComposer.java, DriveOrchestrator.java, DriveConfig.java, DriveProfile.java, DriveGoalFormationStrategy.java, DriveGoalFormationContext.java, DriveGoalProposal.java (all in blocks-core `io.casehub.blocks.agentic.social.drive`/`.goal`), neocortex#345 (goal cognition epic), issue #1114, issue #1115
**Depends on:** D92 (improvement signals feed drive sources), D97 (budget enforcer respects drive profile), D99 (GoalKind.SELF_IMPROVEMENT goals proposed by DriveGoalFormationStrategy)
**Exploration:** deep-analysis
**Status:** revised — replaced rigid tier hierarchy with Drive system integration; balanced competing needs instead of strict priority ordering. Further revised to include centralised cognitive agent model (D104).

## D104: Self-improvement as a cognitive agent — full CognitionCore stack

**Choice:** The self-improvement system is a cognitive agent running the full blocks `CognitionCore` stack. It is not a mechanical budgeting system — it is an entity with personality, drives, emotions, inner narrative, memory, strategy learning, and goals. Both centralised AND distributed:

**Centralised:** A dedicated self-improvement agent with an `AgentDescriptor` configured for improvement-oriented cognition. Its `CognitionCore` orchestrates:
- **MoodOrchestrator** (PAD) — it feels. Pleasure rises when improvements land, drops when PRs get rejected. Arousal rises when it discovers promising research or faces a crisis. Dominance rises when it expands capabilities, drops when it hits walls.
- **DriveOrchestrator** — its needs. COMPETENCE for stability/quality, CURIOSITY for research/growth, AUTONOMY for capability expansion, AFFILIATION for coordination quality. Needs balance dynamically — no rigid hierarchy.
- **NarrativeOrchestrator** — its inner monologue. "I noticed test coverage dropped in the planning module. That makes me uneasy. I should look into it."
- **StrategyLearningOrchestrator** — it learns what improvement approaches work. "Dependency bumps in module X always go smoothly. Code refactors in module Y often get rejected."
- **MentalModelOrchestrator** — it models the platform it's improving. Tracks what's stable, what's fragile, what's well-tested.
- **GoalProposalOrchestrator** — it proposes improvement goals from its cognitive state (via `DriveGoalFormationStrategy`).
- **MemoryHygieneOrchestrator** — it manages its own knowledge freshness.

**Distributed:** Every swarm agent's drive profile includes improvement-relevant signals. When any agent encounters a failure pattern, test regression, or capability gap, it deposits improvement signals that feed the centralised agent's drive sources. The swarm collectively senses; the cognitive agent decides and acts.

**Personality configuration** for the self-improvement agent (via `AgentDescriptor.disposition()`):
- High curiosity / risk appetite — explores new techniques
- High competence focus — sensitive to quality degradation
- Moderate autonomy — expands capabilities within safety bounds
- Moderate social orientation — coordinates with other agents when improvements affect them

The mood-drive feedback loop: PAD emotional state modulates drives (already implemented in `DriveComposer.applyMoodModulation()`). A frustrated agent (low pleasure from rejected PRs) naturally shifts toward safer stability work. A satisfied agent (high pleasure from recent successes) gets bolder and invests more in research. An excited agent (high arousal from discovering a paper) prioritises capability growth. This is not hardcoded logic — it emerges from the cognitive architecture.

The engine provides the case lifecycle, signal infrastructure, budget enforcement, and rule-based drive sources. Blocks provides the cognitive stack (`CognitionCore`) and LLM-powered workers. As neocortex goal cognition (#345) lands — goal dependency graphs, affective valuation, multi-signal priority, opportunity cost awareness, goal-conditioned retrieval — the self-improvement agent automatically benefits.

**Alternatives:**
- Mechanical budgeting system — no personality, no emotion, no narrative. Works but misses the cognitive dimension. Doesn't leverage the existing avatar infrastructure.
- Distributed only — no centralised agent; improvement emerges purely from swarm consensus. Clean but no focused execution capability; improvements would compete with regular case work for agent attention.
- Centralised only — dedicated agent, no distributed sensing. Misses the swarm's collective observation capability; the centralised agent can't see what it hasn't observed itself.

**Rationale:** The platform already has a complete cognitive agent architecture — CognitionCore with mood, drives, narrative, strategy, mental model, goals. Using it for self-improvement means the agent that improves the platform is a first-class cognitive entity, not a mechanical process. Its improvement decisions are influenced by how it feels (PAD), what it needs (drives), what it's learned (strategy), and what it's thinking about (narrative). This makes the self-improvement system a demonstration of the platform's own capabilities — it eats its own cooking.

The centralised+distributed model follows the same pattern as biological self-regulation: every cell monitors its own health (distributed sensing), while the brain coordinates systemic response (centralised decision-making). Neither works alone; together they create adaptive self-regulation.

**Trade-offs:** Full CognitionCore integration is a blocks-level capability. In engine-only mode, the self-improvement system operates with rule-based drive sources and budget allocation — functional but without the cognitive depth (no mood modulation, no narrative, no strategy learning). This is the same graceful degradation as the rest of the hive mind: engine works, blocks enhances.

**Sources:** CognitionCore.java, CognitionSnapshot.java, MoodOrchestrator (PAD), DriveOrchestrator, NarrativeOrchestrator, StrategyLearningOrchestrator, MentalModelOrchestrator, GoalProposalOrchestrator, DriveGoalFormationStrategy.java, neocortex#345 (goal cognition epic), issue #1114
**Depends on:** D103 (Drive system integration), D92 (improvement signals), D93 (improvement case), D95 (improvement taxonomy), D99 (GoalKind.SELF_IMPROVEMENT)
**Exploration:** deep-analysis
**Status:** captured

## D105: Full cognitive memory — MindMap as the agent's lived experience

**Choice:** The self-improvement agent uses the full neocortex memory architecture as its long-term memory of EVERYTHING it experiences — not just research findings, but build attempts, CI events, human interactions, agent coordination, improvement outcomes, platform state changes, and sessions. Sessions disappear; memory persists. The MindMap is the agent's lived experience as a semantic knowledge graph with typed edges, emotional associations, relationship memory, and consolidation lifecycle.

**Everything becomes memory:**

*Improvement attempts:*
- "I bumped hibernate-core from 6.6 to 6.7 last week. Tests broke in persistence module — a transitive pulled in an incompatible validator. That was stressful. I learned to always check transitives first." *(episodic → consolidated knowledge with negative valence + causal relationship)*

*Build and CI events:*
- "The build broke at 3am on Tuesday. Root cause was a flaky test in SwarmProvisionerTest that only fails under parallel execution. Took 40 minutes to diagnose. Frustrating." *(event memory with temporal context, negative arousal, linked to the specific test class)*

*Human interactions:*
- "The reviewer on PR #847 gave detailed feedback on edge cases in convergence detection. They care about correctness. I respect their judgment." *(relationship memory, per-agent-pair interaction, positive affiliation)*
- "The user asked me to pause capability research and focus on stability. They seemed concerned about production reliability." *(interaction memory, belief about user priorities)*

*Agent coordination:*
- "Agent-7 and I work well together on coverage improvements — it finds gaps in the call graph, I generate test scaffolding. We complement each other." *(relationship memory, team cohesion, positive affiliation)*
- "Agent-3 keeps proposing dependency bumps without checking transitives. I've had to revert two of its PRs." *(relationship memory, frustration, competence concern)*

*Research and techniques:*
- "That paper on adaptive signal decay looked promising. Three months ago I tried a similar approach and it failed because the half-life was too aggressive. But the paper suggests per-signal adaptive rates. I'm excited to revisit this." *(cross-referencing failure memory with new research, curiosity rising)*

*Platform understanding:*
- "The planning module is the most fragile part of the codebase — 40% of CI failures originate there. The routing module is rock-solid — no failures in 6 weeks." *(mental model of platform health, built from accumulated experience)*

**The full neocortex memory stack:**

| Layer | What it stores | Timescale | Self-improvement role |
|-------|---------------|-----------|----------------------|
| **Episodic buffer** | Raw events — build results, PR outcomes, research sessions, human feedback | Minutes–hours | Immediate experience, input to mood shifts |
| **Experience events** | Structured timestamped events per agent | Hours–days | Feeds consolidation, drives emotional associations |
| **Relationship memory** | Per-agent-pair interaction graph | Persistent | Who reviews well, who breaks things, who to coordinate with |
| **Reflective diary** | Periodic synthesis of raw experience into insights | Days–weeks | "I notice I'm better at coverage work than refactoring" |
| **MindMap nodes** | Consolidated semantic knowledge with emotional valence | Persistent | Durable understanding of platform, techniques, relationships |
| **Consolidation** | Promotes ephemeral → episodic → semantic | Continuous | Unreinforced memories decay; successful patterns become permanent knowledge |
| **CBR traces** | Structured improvement outcomes | Persistent | "Find similar past improvements" — quantitative retrieval |

**Emotional associations are first-class:**
Every memory node carries emotional valence from its PAD state at formation and subsequent interactions. A technique that worked carries satisfaction. A build failure carries frustration. A promising paper carries excitement. These emotions aren't metadata — they modulate drives (via `DriveComposer.applyMoodModulation()`), influence goal priority (via affective valuation in neocortex#345), and guide what the agent finds worth pursuing. The emotional landscape IS the prioritisation mechanism.

**Curiosity from knowledge gaps:**
`CuriositySignalGenerator` identifies holes in the knowledge graph — referenced but unexplored techniques, known problems without known solutions, successful approaches in one domain not yet tried in another. These gaps become CURIOSITY drive intensity, naturally directing exploration.

**Goal-conditioned retrieval:**
"What do I know that's relevant to THIS improvement goal?" uses `ModulationFactor` to bias retrieval toward actionable knowledge in the context of the current goal. Not just topic similarity — emotional valence and past outcomes weight retrieval.

**CBR traces complement MindMap:**
D98's CBR traces are structured outcome records (what was tried, metrics delta, PR status). MindMap is relational (how concepts connect, what they mean, how the agent feels). Both project the same experience — CBR for "find similar," MindMap for "understand and navigate."

**Alternatives:**
- CBR-only — flat traces, no semantic relationships, no emotional associations. Can't navigate "what connects to what" or explore via curiosity.
- Session-scoped memory — knowledge dies with the session. No learning across sessions. Each session starts from scratch.
- Research-only memory — too narrow. The agent's experience of build failures, human feedback, and agent coordination is just as important as research findings.

**Rationale:** The MindMap is the agent's mind. For a self-improving agent, that mind encompasses everything: the platform's architecture, the research landscape, the history of improvements, the relationships with humans and other agents, the emotional weight of past successes and failures. Sessions are ephemeral; the agent's understanding is permanent. Using the full neocortex memory stack means the self-improvement agent grows wiser over time — not just smarter (more techniques) but more experienced (better judgment about what works, who to trust, when to be cautious, when to be bold).

**Trade-offs:** Full memory integration requires neocortex. In engine-only mode, memory falls back to EventLog + CBR traces — functional but without semantic navigation, emotional associations, relationship memory, consolidation lifecycle, or curiosity-driven exploration. The full cognitive agent requires blocks + neocortex.

**Sources:** MindMap, CognitiveProfile, CognitiveDerivationEngine, CuriositySignalGenerator, ModulationFactor, ExperienceConsolidationPhase (#336), cognitive node type classification (#322), relationship memory (neocortex#184, #186), reflective diary (neocortex#186), neocortex#345 (goal cognition — affective valuation, goal-conditioned retrieval), D98 (CBR outcome traces), D104 (cognitive agent model)
**Depends on:** D104 (cognitive agent uses CognitionCore), D98 (CBR traces complement MindMap), D95 (improvement taxonomy — research is one category of experience)
**Exploration:** deep-analysis
**Status:** revised — expanded from research-only memory to full lived experience across all interaction types
