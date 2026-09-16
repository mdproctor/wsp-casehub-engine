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
**Depends on:** D1 (SPI design determines evaluation contract), D4 (placement determines where handler lives), D6 (thread model)
**Exploration:** quick
**Status:** revised — added bounded execution constraint; clarified LLM-backed observation is out of scope for serializer gate; removed circular D7 dependency (D5 is independent of materialization target)

## D6: Observer thread model — Synchronous, bounded execution

**Choice:** All observers execute within the evaluation cycle (inside the serializer gate per D5), blocking-synchronous from the evaluator's perspective. Each observer is dispatched to a virtual thread via `CompletableFuture.supplyAsync(observer::observe, virtualThreads)` with a 100ms `orTimeout()`, then `.join()`ed back to the evaluation thread. The pipeline processes observers sequentially — each completes (or times out) before the next is evaluated. Observer code runs on the virtual thread pool: implementations must be thread-safe and should avoid `synchronized` blocks (which pin platform threads — a known virtual thread anti-pattern). The SPI contract (D1) is a synchronous functional interface; the engine calls `observe()` and uses the returned `List<Observation>` immediately. LLM-backed observation (blocks #284) does not call LLM inside the observer — the engine observer detects a fast trigger pattern, and the LLM call is dispatched as a separate worker via normal binding dispatch.

**Alternatives:**
- Asynchronous evaluation — decouples from evaluation cycle, loses deterministic ordering guarantee, indeterminate availability of results in cycle N+1
- Hybrid sync/async with marker interface — complexity of two execution models, interaction semantics undefined, harder to reason about

**Rationale:** Engine-level observation is mechanical pattern detection — multi-key correlation, threshold crossings, temporal sequence detection. These are fast, bounded computations. The "engine mechanics, blocks intelligence" split from epic #1104 means the engine provides the detection mechanism; blocks provides the LLM intelligence that acts on detections. The observer detects; a worker dispatch handles the LLM call.

**Trade-offs:** Slow observers degrade evaluation throughput. Timeout enforcement is the safety net — observers exceeding the bound are interrupted and their contribution lost for that cycle.

**Sources:** `CaseEvaluationSerializer.java:23`, issue #1104 ("Engine mechanics, blocks intelligence")
**Depends on:** D1 (SPI design), D5 (pipeline integration)
**Exploration:** quick (surfaced by review R1-03, R1-07)
**Status:** revised — clarified virtual thread dispatch mechanism and thread-safety requirements for observer implementations

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

**Choice:** A signal is a single value per `(caseId, signalName)`. When multiple agents write the same signal name, the write is a reinforcement: strength is set to `max(currentEffective, newStrength)`, timestamp resets to now, and `reinforcementCount` increments. `lastSource` (agent ID) is tracked for audit. This makes heavily-trafficked signals persist longer — exactly the ACO behavior.

**Alternatives:**
- Per-agent signal instances `(caseId, signalName, agentId)` with aggregation — more faithful to multi-ant pheromone, but aggregation strategy becomes a sub-decision, N entries per signal per agent
- Append-only signal log — maximally faithful but unbounded storage, O(N) reads

**Rationale:** `max()` preserves two orthogonal dimensions of signal quality: `effectiveStrength` represents the peak confidence of any single endorsement; `reinforcementCount` represents the breadth of consensus. Observers can weigh these independently — e.g., `effectiveStrength * log(reinforcementCount)` for consensus-weighted strength. With additive-and-clamp (`min(1.0, current + deposit)`), these dimensions collapse: 3 deposits of 0.4 saturate to 1.0, making strength meaningless and losing individual signal quality to clamping. The `max()` semantics also mean reinforcement resets the decay timestamp, so frequently-reinforced signals persist longer — consensus manifests through temporal persistence, not strength amplification. Note: this DIFFERS from classical ACO where pheromone deposit is additive (`τ ← τ + Σ Δτ`). The platform's signal model is stigmergy-inspired but not an ACO implementation — agents have varying confidence levels and the strongest endorsement should dominate strength, while consensus is captured separately via `reinforcementCount`.

**Trade-offs:** Loses individual agent contribution history. If two agents reinforce and then one "retracts," there's no mechanism to reduce strength other than natural decay. Acceptable — pheromone trails don't support retraction in the biological model either.

**Sources:** engine#1106 issue spec (reinforcement model), ACO literature (pheromone deposit/evaporation), `DispositionSignalStore` (eidos uses per-agent signals — different use case, personality is inherently per-agent)
**Depends on:** D10 (registry storage), D11 (read-time decay)
**Exploration:** quick
**Status:** revised — corrected inaccurate ACO claim in rationale; max() semantics defended with orthogonal-dimensions argument

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

**Rationale:** Five types give full expressiveness. The four typed permits are fully auditable — you can inspect exactly what an agent watches, what keys, what thresholds. The JQ catchall provides flexibility for unregulated domains. The config gate makes it a per-case-definition policy decision, consistent with `ObservationConfig`'s existing role as the observation policy surface. `registerObserver()` escape hatch is ungated (it's the engine-internal API, not agent-facing interest registration).

**Trade-offs:** JQ gate enforcement is at registration time only — if the config changes after registration, existing JQ observers continue running. Acceptable — config changes don't retroactively invalidate live case behavior. The five-type vocabulary may need extension for future interest patterns (e.g. rate-of-change) — the sealed hierarchy would need a new permit, which is a source-compatible addition.

**Sources:** `ThresholdObserver.java`, `CorrelationObserver.java`, `TemporalSequenceObserver.java`, `SignalStrengthObserver.java` (runtime-core), `ObservationConfig.java` (engine-api), engine#1107
**Depends on:** D19 (faceted architecture — interests live on InterestSpace)
**Exploration:** quick
**Status:** captured

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

**Choice:** Pre-release clean design. Facet interfaces (`SignalSpace`, `InterestSpace`) in `io.casehub.api.engine` alongside `WorkerRuntime`. Interest types (`InterestDeclaration`, `InterestRegistration`, `InterestLandscape`) in `io.casehub.api.spi.observation` alongside `EnvironmentObserver`. Signal types unchanged in `io.casehub.api.model.signal`. Implementations: `DefaultSignalSpace` in `runtime-core/internal/signal/`, `DefaultInterestSpace` in `runtime-core/internal/observation/`. `DefaultWorkerRuntime` creates both facet implementations and returns them from `signals()` and `interests()`.

**Alternatives:**
- Facets in domain packages (SignalSpace in api/model/signal) — mixes behavioral interfaces with value records
- New api/coordination package — more packages than necessary

**Rationale:** Follows codebase convention: `api/engine` = runtime interfaces, `api/spi` = domain SPIs, `api/model` = value types. Facet interfaces are runtime surfaces (same nature as WorkerRuntime). Interest types are observation domain artifacts. `DefaultWorkerRuntime` gets slimmer — delegates coordination to focused facet implementations.

**Trade-offs:** `api/engine` package grows from 1 to 3 files (WorkerRuntime, SignalSpace, InterestSpace). Future facets (#1108 NeighborSpace, #1109 RuleSpace) grow it to 5. Still manageable.

**Sources:** `api/engine/WorkerRuntime.java`, `api/spi/observation/` package, `api/model/signal/` package, engine#1107
**Depends on:** D19 (faceted architecture), D20 (InterestDeclaration types)
**Exploration:** quick
**Status:** captured

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

**Trade-offs:** Long-running cases (multi-day AML investigations, clinical trials) lose accumulated coordination intelligence on restart. Bounded by the observation that the current platform has no horizontal scaling — in-memory state is consistent because there's one instance. Multi-instance deployment would require a distributed backend for all three registries. The `CaseRecoveryStateRegistry` (which handles other recovery concerns) is also in-memory, confirming this is a platform-level constraint, not a coordination-specific one.

**Sources:** `ObservationRegistry.java`, `SignalRegistry.java`, `ContextHistoryBuffer.java`, `CaseRecoveryStateRegistry` (all in-memory), `Resettable` interface
**Depends on:** D7 (materialization), D10 (signal storage)
**Exploration:** quick (surfaced by review R1-09)
**Status:** captured

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

**Choice:** `NeighborSpace` in `io.casehub.api.engine` (alongside `SignalSpace`, `InterestSpace`). `Neighbor` and `NeighborRelation` in `io.casehub.api.spi.observation` (neighbor awareness is part of the observation domain — agents observing their social environment). `DefaultNeighborSpace` in `runtime-core` at `io.casehub.engine.internal.observation`. Follows D23 exactly.

**Alternatives:** None considered — established pattern.

**Rationale:** Facet interfaces are runtime surfaces (same nature as WorkerRuntime). Neighbor types are observation-domain artifacts (agents observing their social environment). Implementations in runtime-core.

**Trade-offs:** `api/spi/observation` package grows wider. Acceptable — all observation-related types belong together.

**Sources:** D23 (module placement pattern), D19 (faceted architecture), engine#1108
**Depends on:** D32 (NeighborSpace architecture), D33 (Neighbor data model)
**Exploration:** quick
**Status:** captured

## D37: Rule action scope — coordination + context writes

**Choice:** Approach B — Rules can perform coordination-only actions (deposit signals, register/deregister interests, emit conclusions) AND write specific keys to the CaseContext working layer. Context writes bridge coordination state back into domain state, triggering `CONTEXT_CHANGED` and enabling binding dispatch. Coordination-only actions are best practice for most use cases; context writes are available when rules need to influence case progression directly.

**Alternatives:**
- Coordination only — rules stay entirely in the coordination layer; binding dispatch driven exclusively by external context changes. Simpler but breaks the stigmergy loop: rules can detect coordination patterns but can't cause the case to act without an external context change arriving.
- Full dispatch — rules can request worker scheduling directly, creating a second dispatch path alongside binding triggers. Over-powered and architecturally complex.

**Rationale:** The stigmergy loop requires perceive→decide→act→modify environment. If rules can only deposit signals (coordination-only), the "act" step never reaches domain state — signals are invisible to binding conditions. Context writes close the loop: rule detects a coordination pattern → writes a key to working layer → `CONTEXT_CHANGED` fires → binding with a matching `when` condition dispatches a worker. This is indirect coordination through environment modification — the definition of stigmergy. Context writes are applied after all rules have been evaluated (batched), with a single `CONTEXT_CHANGED` published post-evaluation to avoid re-entrant evaluation within the serializer gate.

**Trade-offs:** Context writes create a path from coordination state to domain state, which means rules can indirectly cause binding dispatch. This risks feedback loops (rule writes → binding fires → worker runs → context changes → rule fires → ...). Mitigated by: (1) one-shot evaluation per cycle (rules evaluate once, no intra-cycle chaining), (2) batched writes applied after all rules complete, (3) bindings can guard against re-triggering with `when` conditions that check for rule-written keys. Best practice guidance: use coordination-only actions by default, context writes only when the coordination pattern needs to influence case progression.

**Sources:** D7 (observation materialization — no CaseContext writes), D5 (pipeline integration), D10 (signal storage — not in CaseContext), engine#1109, engine#1111 (stigmergy requires environment modification), SwarmSys (arXiv:2510.10047)
**Depends on:** D5 (pipeline integration), D10 (signal storage), D19 (faceted architecture)
**Exploration:** quick
**Status:** captured

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

**Rationale:** Swarm agents follow multiple behavioral rules simultaneously (forage AND avoid danger AND follow pheromone gradient). All-fire matches this biological model. Priority-as-ordering (not selection) means context writes from higher-priority rules are overwritten by lower-priority rules on the same key — last-writer-wins, which is consistent with `ConflictResolver.LAST_WRITER_WINS`. Future Drools integration can provide a `RuleEvaluationStrategy` that replaces all-fire with match-resolve-act for domains that need it.

**Trade-offs:** Multiple rules writing the same context key: last-priority-wins. Agents must manage their own rule sets to avoid conflicting actions. Acceptable — per-agent isolation means conflicts are within one agent's rule set, not across agents.

**Sources:** SwarmSys (arXiv:2510.10047 — multiple simultaneous roles), D5 (one-cycle delay), `ConflictResolver.LAST_WRITER_WINS` (existing conflict model), engine#1109, engine#445 (Drools — different model for different purpose)
**Depends on:** D37 (action scope), D38 (condition model)
**Exploration:** quick
**Status:** captured

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

**Choice:** `RuleAction` is a sealed interface with four permits: `DepositSignal(String name, double strength, @Nullable Duration halfLife)`, `RegisterInterest(InterestDeclaration declaration)`, `DeregisterInterest(String interestId)`, `WriteContext(String key, JsonNode value)`. Each action type maps directly to an existing engine operation: signal deposit → `SignalRegistry.deposit()`, interest registration/deregistration → `ObservationRegistry`, context write → `WritableLayer.set()` via `ConflictResolver.LAST_WRITER_WINS`. `RuleCondition` is also sealed: `ExpressionCondition(ExpressionEvaluator evaluator)` | `PredicateCondition(Predicate<RuleContext> predicate)`.

**Alternatives:**
- Add EmitConclusion(type, data) as a fifth action — structured output for other agents. Overlaps with signals (inter-agent communication) and context writes (structured data). Deferred until a concrete use case distinguishes conclusions from signals.
- Add RequestDispatch(capabilityName) — directly request worker scheduling. Rejected in D37 — context writes bridge to dispatch indirectly via binding triggers.

**Rationale:** Four actions cover the stigmergy loop: deposit signals (announce findings), register interests (adapt perception), deregister interests (stop watching), write context (influence case progression). Each action is auditable (data record, not lambda). Each maps to an existing operation with established semantics. Extensible: new sealed permits can be added when new coordination primitives emerge.

**Trade-offs:** No "remove context key" action. An agent wanting to clear a key writes `NullNode.instance`. No "modify signal" action (e.g., reduce strength) — agents can only deposit (which reinforces). Signal reduction happens via natural decay. Both are intentional simplifications.

**Sources:** `SignalRegistry.deposit()`, `ObservationRegistry.registerObserver()`, `WritableLayerImpl.set()`, D37 (action scope), D20 (InterestDeclaration sealed hierarchy — precedent)
**Depends on:** D37 (action scope — WriteContext enabled by option 2), D38 (condition model — RuleCondition sealed)
**Exploration:** quick
**Status:** captured

## D42: Pipeline integration — after observations, batched context writes

**Choice:** `localRules()` runs after `observations()` in `CaseContextChangedEventHandler.evaluateAndDispatch()`. Evaluation order: `rules()` → `goals()` → `observations()` → `localRules()`. Within `localRules()`: (1) build `RuleContext` per agent from registries, (2) evaluate all agents' rules independently, (3) collect all `RuleAction`s, (4) execute coordination actions immediately (signal deposits, interest changes), (5) batch all `WriteContext` actions, (6) apply batched writes in a single pass after all rules complete, (7) if any writes occurred, publish one `CONTEXT_CHANGED` event (queued by `CaseEvaluationSerializer.drainPending()` for the next evaluation cycle). Per-agent rule evaluation timeout: 100ms via `CompletableFuture.orTimeout()` on virtual threads — same pattern as observer evaluation (D6).

**Alternatives:**
- Before observations — rules wouldn't have access to current-cycle observations. Breaks the observe→decide→act pipeline.
- Parallel with observations — race conditions between observation storage and rule reads.
- Context writes applied immediately per agent — ordering between agents becomes significant, harder to reason about.

**Rationale:** After observations ensures rules see current-cycle observations. Batched writes prevent inter-agent ordering effects and ensure a single clean `CONTEXT_CHANGED`. The serializer's `drainPending()` naturally handles the re-evaluation cycle — no special plumbing needed. Virtual thread dispatch with timeout follows the established observer pattern.

**Trade-offs:** All context writes from all agents are batched — if two agents write the same key, last-writer-wins (agent ordering within the batch is undefined). Acceptable — per-agent isolation means agents should write to different keys. Agents sharing keys must coordinate via signals.

**Sources:** `CaseContextChangedEventHandler.java:243-253` (existing pipeline), `CaseEvaluationSerializer.java:35-55` (drainPending), D5 (pipeline integration), D6 (observer thread model)
**Depends on:** D37 (context writes), D39 (one-shot semantics), D41 (action types)
**Exploration:** quick
**Status:** captured

## D43: RuleSpace facet and RuleRegistry

**Choice:** `RuleSpace` is the 4th WorkerRuntime facet. Interface: `register(LocalRule) → RuleRegistration`, `deregister(String ruleId)`, `mine() → List<RuleRegistration>`, `lastFired() → List<RuleFiring>`. `RuleRegistration` record: `(String ruleId, LocalRule rule, Instant registeredAt)`. `RuleFiring` record: `(String ruleId, List<RuleAction> executedActions, Instant firedAt)`. `RuleRegistry` (`common-core`, `@ApplicationScoped`, `Resettable`) stores per-case, per-agent rules and per-cycle firing results. `ConcurrentHashMap<UUID, Map<String, List<LocalRule>>>` for rules (keyed by caseId → agentId → rules), `ConcurrentHashMap<UUID, Map<String, List<RuleFiring>>>` for firings (replaced per cycle, like `ObservationRegistry.storeObservations()`). Scope enforcement same as observers: BINDING rejected, COMPOUND/CASE only. Deduplication by `(caseId, agentId, bindingName, ruleId)`. `DefaultRuleSpace` in `runtime-core/internal/observation/`, wired via `WorkerRuntimeFactory`.

**Alternatives:** None — direct extension of the established facet pattern (D19, D23, D36).

**Rationale:** Follows the InterestSpace/SignalSpace/NeighborSpace pattern exactly. `lastFired()` gives agents visibility into their own rule behavior for adaptive decision-making. Per-cycle firing replacement matches `ObservationRegistry.storeObservations()` semantics.

**Sources:** `InterestSpace.java` (facet pattern), `ObservationRegistry.java` (registry pattern), `WorkerRuntimeFactory.java` (wiring pattern), D19 (faceted architecture), D23 (module placement)
**Depends on:** D39 (rule model), D41 (action types)
**Exploration:** quick
**Status:** captured

## D44: Configuration, audit, lifecycle, module placement

**Choice:** `RuleConfig` record on `CaseDefinition`: `maxRulesPerCase` (default 50), `maxActionsPerCycle` (default 100), `ruleEvaluationTimeoutMs` (default 100). YAML: `ruleConfig:` block under `spec:`. Audit: `CaseHubEventType.RULE_FIRED` (metadata: `agentId`, `ruleId`, `actions[]`, `priority`) and `RULE_REGISTERED` (metadata: `agentId`, `ruleId`, `conditionType`). Lifecycle: `CaseStatusChangedHandler` calls `ruleRegistry.evictByCase()` on terminal status. `ScopedWorkerTerminationHandler` calls `ruleRegistry.unregisterByBinding()` on `COMPOUND_COMPLETED`. Module placement follows D23/D36: `RuleSpace` in `api/engine/`, rule types (`LocalRule`, `RuleAction`, `RuleCondition`, `RuleContext`, `RuleFiring`, `RuleConfig`, `RuleRegistration`) in `api/spi/observation/`, `RuleRegistry` + `DefaultRuleSpace` in `common-core/internal/observation/` and `runtime-core/internal/observation/`. No YAML rule declaration in v1 — rules are runtime-registered by agents via `RuleSpace.register()`. Static YAML rules are a natural extension for v2.

**Alternatives:** None — follows established patterns exactly.

**Rationale:** Configuration pattern matches `ObservationConfig` and `SignalConfig`. Audit pattern matches `OBSERVER_REGISTERED`/`OBSERVATION_DETECTED` and `PHEROMONE_DEPOSITED`. Lifecycle pattern matches all other per-case registries. Module placement follows the observation domain grouping.

**Sources:** `ObservationConfig.java`, `SignalConfig.java`, `CaseHubEventType.java`, `CaseStatusChangedHandler.java`, D23 (module placement), D36 (NeighborSpace placement)
**Depends on:** D39-D43
**Exploration:** quick
**Status:** captured

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

**Rationale:** Four metrics cover the four resource dimensions where swarm pathology manifests: dispatches (agent thrashing), signals (coordination storms), context mutations (state thrashing), evaluations (evaluation re-entrant loops). Sliding-window rates give precise activity trends for convergence detection. Cumulative totals give hard budget enforcement. `SlidingWindowCounter` is a bounded circular buffer of timestamps — `record(Instant)` appends, `rate(windowDuration, now)` counts entries within the window and returns count/windowSeconds. Memory: O(maxWindowEntries) per metric per case, capped at e.g. 1000 entries. Eviction on case termination via `CaseStatusChangedHandler`, same pattern as all other per-case registries.

**Trade-offs:** Four sliding windows per case × 1000 entries each = 4000 timestamps per case. Bounded and manageable. No persistence — lost on restart (consistent with D29, all coordination state is in-memory).

**Sources:** `QuiescenceTracker.java` (per-case atomic state pattern), `SignalRegistry.java` (per-case ConcurrentHashMap pattern), `Resettable` interface, D29 (in-memory only)
**Depends on:** D45 (pipeline integration — convergence phase reads rates)
**Exploration:** quick
**Status:** captured

## D47: Instrumentation points — where metrics are recorded

**Choice:** Each metric is recorded at its natural event source, inside the existing handlers:
- `totalDispatches` — incremented in `CaseContextChangedEventHandler` when a `WorkerScheduleEvent` is published (inside `publishWorkerSchedule()`, after successful dispatch)
- `totalSignalDeposits` — incremented in `CaseContextChangedEventHandler.observations()` when signal expiry detection runs (after deposits from `localRules()` phase), AND in `DefaultSignalSpace.deposit()` for worker-initiated deposits
- `totalContextMutations` — incremented in `CaseContextChangedEventHandler` at cycle start, counting `event.changedKeys().size()` (or a count of keys changed in the context diff)
- `totalEvaluationCycles` — incremented at the top of `evaluateAndDispatch()` (one per serialized evaluation)

All increments are fire-and-forget — no return values, no blocking. `ActivityTracker` is injected into `CaseContextChangedEventHandler` (for evaluations, dispatches, mutations) and `DefaultSignalSpace` (for signal deposits via WorkerRuntime).

**Alternatives:**
- EventLog-based counting (query EventLog for WORKER_SCHEDULED count) — accurate but O(N) query on each evaluation cycle. Too expensive for a per-cycle check.
- CDI event observers on existing events — decouples instrumentation from handlers but adds async overhead and loses per-cycle consistency.

**Rationale:** Direct instrumentation at the event source is the most accurate and lowest-overhead approach. Each handler already knows what it's doing — adding an `activityTracker.recordDispatch(caseId)` call is a single-line addition. The tracker's sliding window handles timing; the handler just signals "this happened."

**Trade-offs:** Couples ActivityTracker to CaseContextChangedEventHandler and DefaultSignalSpace. Acceptable — these are the canonical event sources for these metrics.

**Sources:** `CaseContextChangedEventHandler.java:publishWorkerSchedule()`, `CaseContextChangedEventHandler.java:evaluateAndDispatch()`, `DefaultSignalSpace.java:deposit()`
**Depends on:** D46 (ActivityTracker defines what is tracked)
**Exploration:** quick
**Status:** captured

## D48: Budget enforcement — hard gate with case fault

**Choice:** Budget enforcement is a hard gate checked at two points: (1) at the top of `evaluateAndDispatch()` for evaluation cycle budget, and (2) inside dispatch/deposit operations for their respective budgets. When any cumulative count exceeds its configured budget cap (`ConvergenceConfig.maxDispatches`, `maxSignalDeposits`, `maxContextMutations`, `maxEvaluationCycles`), the engine: (a) fires a `BUDGET_EXHAUSTED` CaseHubEventType with metadata identifying which budget was exceeded, (b) dispatches `CaseStatusChanged(FAULTED)` with reason "Budget exhausted: <metric>". Budget caps are nullable on `ConvergenceConfig` — null means no limit (backward compatible, no enforcement for cases without convergence config).

**Alternatives:**
- Advisory monitoring + alert — doesn't prevent runaway. The whole point of budget caps is to be a fail-safe.
- Soft cap with escalation (warn at 80%, fault at 100%) — adds configuration complexity. Warning can be implemented separately as a convergence observation without coupling to the enforcement mechanism.

**Rationale:** Hard gate prevents unbounded resource consumption, which is the #1 production failure mode for multi-agent systems (40% of pilots fail from coordination overhead). Faulting the case is the correct response — it surfaces the problem clearly and triggers the existing failure handling pipeline (CaseOutcomeObserver, EventLog audit, etc.). Null caps preserve backward compatibility — existing cases without convergence config are unaffected.

**Trade-offs:** Hard fault is not graceful — running workers are not proactively cancelled (they complete naturally and find the case already terminal). Acceptable — `CaseStatusChangedHandler` handles cleanup. A case author who wants a warning gate can use a local rule that reads the activity metrics and reacts.

**Sources:** `CaseStatusChanged` event, `CaseStatusChangedHandler.java` (terminal state handling), `maxConcurrentDispatches` (existing hard cap pattern), engine#1044 (WatchdogRecoveryBridge CANCEL_AFFECTED pattern)
**Depends on:** D46 (ActivityTracker provides counts), D47 (instrumentation provides the counts)
**Exploration:** quick
**Status:** captured

## D49: ConvergenceDetector — activity quiescence with sustained stability

**Choice:** `ConvergenceDetector` (`runtime-core`, `@ApplicationScoped`) evaluates convergence during the 5th pipeline phase. Convergence condition: ALL four activity rates (dispatch, signal deposit, context mutation, evaluation) are below their respective thresholds simultaneously for a sustained duration (`stabilityWindow`). Per-case state tracks: `firstQuietCycle` (Instant when all rates first dropped below threshold, null when any rate exceeds), `consecutiveQuietCycles` (int). When `Duration.between(firstQuietCycle, now) >= stabilityWindow` → convergence detected. On detection: fires synthetic `GoalReachedEvent` with goal name `"_converged"` (engine-reserved, prefixed with `_`). Resets `firstQuietCycle` to prevent repeated firing (one convergence event per case lifetime).

**Alternatives:**
- Weighted composite score — harder to debug. "Which rate caused convergence?" is a common diagnostic question. Threshold-per-metric is directly inspectable.
- Configurable expression (JQ/predicate) — maximum flexibility but opaque. Convergence is a well-defined concept — thresholds + sustained duration cover it.

**Rationale:** "Everything has quieted down for long enough" is the clearest convergence signal. Each rate threshold is independently configurable — fast-changing cases (real-time monitoring) need different thresholds than slow cases (multi-day investigations). `stabilityWindow` prevents false positives from temporary lulls. Synthetic goal integration means no new termination path — the existing `GoalReachedEventHandler` handles case status transition if the CaseDefinition declares a convergence completion goal.

**Trade-offs:** Single convergence firing per case. If a case "de-converges" (activity resumes after convergence), the detector won't fire again. Acceptable — convergence is a terminal detection, not a toggle. Cases that need re-evaluation should use a local rule that monitors activity rates directly.

**Sources:** `GoalReachedEventHandler.java:102-148` (goal evaluation), `QuiescenceTracker.java` (per-case state pattern), D45 (pipeline phase), D46 (activity rates)
**Depends on:** D45 (pipeline phase), D46 (ActivityTracker provides rates), D48 (budget enforcement runs before convergence)
**Exploration:** quick
**Status:** captured

## D50: Goal integration — convergence goal kind and CaseDefinition wiring

**Choice:** Convergence-triggered termination reuses `GoalBasedCompletion`. New reserved goal name `"_converged"` — the engine fires this when the `ConvergenceDetector` detects convergence. Case definitions that want convergence-based termination declare it in their completion block:
```yaml
completion:
  success:
    anyOf: [case-resolved, _converged]
```
The `_converged` goal is fired by the engine, not by any agent. If a CaseDefinition does not include `_converged` in its completion goals, convergence detection still runs (for monitoring/audit) but does not trigger termination. A new `CaseHubEventType.CONVERGENCE_DETECTED` is always written to EventLog regardless of whether termination fires.

**Alternatives:**
- New CaseCompletion variant (`ConvergenceCompletion`) — requires unsealing `CaseCompletion` and adding a new code path in `GoalReachedEventHandler`. More invasive.
- Direct `CaseStatusChanged(COMPLETED)` dispatch — bypasses goal system, creates a second termination path. Fragile and harder to reason about.

**Rationale:** GoalBasedCompletion is already the extensible completion mechanism. `GoalKind` is an interface (not enum), so custom kinds work. Adding a convergence goal is purely declarative — no code changes to the completion system. The `_` prefix convention distinguishes engine-fired goals from agent-fired goals.

**Trade-offs:** Case authors must explicitly opt in to convergence termination by adding `_converged` to their completion goals. This is intentional — convergence detection without termination is useful for monitoring. Automatic termination on convergence would surprise case authors who don't expect it.

**Sources:** `GoalBasedCompletion.java:23-56` (GoalBasedCompletion builder), `GoalKind.java:17` (interface, not enum), `GoalReachedEventHandler.java:102` (evaluateCompletion), D49 (fires synthetic goal)
**Depends on:** D49 (ConvergenceDetector fires the goal)
**Exploration:** quick
**Status:** captured

## D51: DiversityMonitor — output similarity tracking per binding

**Choice:** `DiversityMonitor` (`runtime-core`, `@ApplicationScoped`) tracks per-binding output similarity across agents. On each successful worker completion (`WorkflowExecutionCompletedHandler` success path), stores the output key set and a content hash per key. When `recentOutputCount >= diversityMinSamples` (configurable, default 3), computes pairwise Jaccard similarity on key sets. When average Jaccard exceeds `diversityThreshold` (configurable, default 0.9) AND value hashes match for overlapping keys, fires `DIVERSITY_VIOLATION` CaseHubEventType. Per-binding sliding window of last N outputs (default 10). No cross-binding comparison — diversity is evaluated within the same capability.

**Alternatives:**
- Signal concentration monitoring — only catches collusion manifesting through signals, misses output-level convergence.
- LLM-based semantic analysis (deferred to blocks) — engine provides metrics, blocks provides intelligence. Future extension via observer SPI.

**Rationale:** Key-set Jaccard + value hash is classical, deterministic, and O(K×N²) where K = output keys and N = window size (small). Detects structurally identical outputs — the price-fixing equivalent where all agents produce the same answer without coordinating. Per-binding scoping makes the comparison meaningful — agents working on the same capability should produce diverse approaches. `diversityMinSamples` prevents false positives when only 1-2 agents have run.

**Trade-offs:** Structural similarity only — semantically equivalent but structurally different outputs are not detected. Acceptable for v1 — LLM-backed semantic analysis is a natural blocks extension. Value hash comparison is exact-match — near-duplicates with minor field variations pass. Mitigated by the Jaccard threshold on key sets catching most near-duplicates.

**Sources:** `WorkflowExecutionCompletedHandler.java` (success path, output access), `ConflictResolver.java` (output key handling precedent), engine#1110 issue spec (anti-collusion requirements)
**Depends on:** D45 (pipeline runs after outputs are recorded), D46 (ActivityTracker pattern for per-case state)
**Exploration:** quick
**Status:** captured

## D52: ConvergenceConfig — per-case configuration

**Choice:** `ConvergenceConfig` record in `engine-api` under `io.casehub.api.model.convergence`:
```java
ConvergenceConfig(
    // Budget caps (null = no limit)
    Integer maxDispatches,
    Integer maxSignalDeposits,
    Integer maxContextMutations,
    Integer maxEvaluationCycles,
    // Convergence thresholds (rates per second)
    Double dispatchRateThreshold,       // default 0.1
    Double signalDepositRateThreshold,  // default 0.1
    Double contextMutationRateThreshold,// default 0.1
    Double evaluationRateThreshold,     // default 0.5
    // Timing
    Duration stabilityWindow,           // default 30 seconds
    Duration rateWindow,                // default 60 seconds (sliding window size)
    // Diversity
    Double diversityThreshold,          // default 0.9
    Integer diversityMinSamples,        // default 3
    Integer diversityWindowSize,        // default 10
    // Master switch
    boolean enabled                     // default false
)
```
`CaseDefinition` gains `convergenceConfig` (nullable, null = disabled). Builder: `.convergenceConfig(ConvergenceConfig)`. YAML: `convergenceConfig:` block under `spec:`. `enabled: false` default means existing cases are completely unaffected — opt-in only.

**Alternatives:**
- Separate config records per concern (BudgetConfig, ConvergenceThresholdsConfig, DiversityConfig) — more granular but more configuration surface area for the user. One record is simpler to declare in YAML.
- Config on individual bindings — convergence is a case-level concern, not per-binding.

**Rationale:** Single configuration record follows the `ObservationConfig`, `SignalConfig`, `RuleConfig` pattern. All convergence-related settings in one place. `enabled: false` default preserves backward compatibility — zero impact on existing cases. Individual null caps mean each budget dimension can be independently enabled.

**Trade-offs:** One large record. Acceptable — the fields group naturally (budgets, rates, timing, diversity, switch). YAML nesting keeps it readable.

**Sources:** `ObservationConfig.java` (record pattern), `SignalConfig.java` (record pattern), `RuleConfig.java` (record pattern), `CaseDefinition` (config surface)
**Depends on:** D46 (defines what metrics exist), D49 (defines what thresholds mean), D51 (defines diversity parameters)
**Exploration:** quick
**Status:** captured

## D53: Module placement — convergence types in api/model/convergence, infrastructure in common-core and runtime-core

**Choice:** `ConvergenceConfig` in `io.casehub.api.model.convergence`. `ActivityTracker` and `SlidingWindowCounter` in `io.casehub.engine.common.internal.convergence`. `ConvergenceDetector` and `DiversityMonitor` in `io.casehub.engine.internal.convergence` (runtime-core). Follows the established pattern: value types in api, mutable state management in common-core, handler/detection logic in runtime-core.

**Alternatives:** None — direct analog of D4 (observation), D17 (signals), D36 (neighbors), D44 (rules) placement.

**Rationale:** Consistent with every prior module placement decision in this epic.

**Sources:** D4, D17, D36, D44 (module placement precedents)
**Depends on:** D46, D49, D51, D52 (defines what types exist)
**Exploration:** quick
**Status:** captured

## D54: Audit — CONVERGENCE_DETECTED, BUDGET_EXHAUSTED, DIVERSITY_VIOLATION event types

**Choice:** Three new `CaseHubEventType` values:
- `CONVERGENCE_DETECTED` — fired when all activity rates drop below threshold for the stability window. Metadata: `dispatchRate`, `signalDepositRate`, `contextMutationRate`, `evaluationRate`, `stabilityDuration`, `totalDispatches`, `totalSignalDeposits`, `totalContextMutations`, `totalEvaluationCycles`.
- `BUDGET_EXHAUSTED` — fired when any cumulative budget cap is exceeded. Metadata: `exhaustedMetric`, `currentCount`, `budgetCap`.
- `DIVERSITY_VIOLATION` — fired when output diversity drops below threshold. Metadata: `bindingName`, `averageJaccard`, `matchingOutputCount`, `totalSamples`, `affectedAgents`.

All three are written to EventLog immediately when detected. `CONVERGENCE_DETECTED` is always written (even if the case does not have `_converged` in its completion goals — pure audit). `BUDGET_EXHAUSTED` is written before the case is faulted.

**Alternatives:** None — follows the established audit pattern from D16 (pheromone), D27 (interest), D44 (rules).

**Sources:** `CaseHubEventType.java`, D16 (audit pattern), D27 (audit pattern)
**Depends on:** D49, D48, D51 (define the detection events)
**Exploration:** quick
**Status:** captured

## D55: Lifecycle — case termination eviction + Resettable

**Choice:** `CaseStatusChangedHandler` calls `activityTracker.evictByCase(caseId)` and `diversityMonitor.evictByCase(caseId)` on terminal case status. Both implement `Resettable` for demo/test replay. Same pattern as `SignalRegistry`, `ObservationRegistry`, `RuleRegistry`, `ContextHistoryBuffer`.

**Alternatives:** None — established lifecycle pattern.

**Sources:** `CaseStatusChangedHandler.java` (terminal eviction), `Resettable` interface, D18 (signal lifecycle), D44 (rule lifecycle)
**Depends on:** D46 (ActivityTracker), D51 (DiversityMonitor)
**Exploration:** quick
**Status:** captured

## D56: WorkerRuntime surfacing — convergence metrics as read-only view

**Choice:** No new WorkerRuntime facet for convergence metrics. Agents should not directly read or influence convergence detection — it is a system-level concern. Convergence metrics are visible to agents indirectly: (1) through observations (a classical observer can watch for `CONVERGENCE_DETECTED` events), (2) through context signals (budget warnings can be written to context by local rules). The `ActivityTracker` is engine-internal infrastructure, not an agent-facing API. If future issues (#1111-#1115) need agent-visible metrics, a read-only `MetricsSpace` facet can be added without changing the tracker.

**Alternatives:**
- Add `MetricsSpace` facet now — provides `metrics() → CaseActivitySnapshot` with rate/count views. More transparent to agents but exposes system-level concern at the agent level.
- Add metrics to RuleContext — local rules could condition on activity rates. Useful but conflates coordination rules with system monitoring.

**Rationale:** Convergence detection is orthogonal to agent coordination. Agents coordinate via signals, observations, interests, neighbors, and rules. The engine monitors the collective behavior and intervenes when necessary. Exposing metrics to agents creates a feedback loop where agents could game the convergence detector (e.g., depositing a signal to prevent convergence detection).

**Trade-offs:** Agents cannot proactively respond to convergence metrics. They can only respond to the engine's interventions (faulted case, convergence goal). Acceptable — the engine is the authority on convergence, not the agents.

**Sources:** D19 (faceted architecture), D32 (NeighborSpace — read-only facade precedent), engine#1110 issue spec
**Depends on:** D46 (ActivityTracker is the infrastructure being surfaced or not)
**Exploration:** quick
**Status:** captured
