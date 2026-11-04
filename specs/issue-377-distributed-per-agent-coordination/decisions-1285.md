# Decisions — engine#1285 AgentWorkLifecycleListener + AgentWorkState

## D1: AgentWorkState package placement

**Choice:** `api/model/stigmergy/` alongside AgentLifecycleState and AgentState
**Alternatives:**
- `api/model/coordination/` — work state governs what agents do within a coordination topology, but it's runtime state not structural relationship
- New `api/model/agent/` subpackage — separates per-agent concerns but fragments the existing stigmergy cohesion
**Rationale:** Work state is per-agent runtime state, same domain as AgentLifecycleState. The stigmergy package tracks agent existence and behavior — work state is the next layer of that. Coordination package is about structural relationships (roles, topology), not runtime state.
**Trade-offs:** Stigmergy package grows larger. Acceptable — the package is cohesive around per-agent runtime state.
**Sources:** AgentLifecycleState.java (api/model/stigmergy/), AgentState.java (api/model/stigmergy/)
**Exploration:** quick
**Status:** captured

## D2: SPI shape — single interface

**Choice:** Single `AgentWorkLifecycleListener` interface with five default methods covering both work-state transitions and role lifecycle
**Alternatives:**
- Split into `AgentWorkStateListener` + `AgentRoleLifecycleListener` — cleaner separation but over-decomposition for 5 methods with defaults; two injection points; role events are coupled to work execution context
**Rationale:** Matches issue spec directly. All methods are default (no-op), so consumers pick what they care about. One injection point. Follows SwarmCoordinationListener precedent which also mixes concerns (roles + progress + provisioning).
**Trade-offs:** Single interface mixes work-state and role-lifecycle concerns. Acceptable — consumers cherry-pick via defaults, and splitting 5 methods across 2 interfaces adds wiring cost for no practical benefit.
**Sources:** SwarmCoordinationListener.java (api/spi/stigmergy/)
**Exploration:** quick
**Status:** captured

## D3: AgentWorkLifecycleListener package placement

**Choice:** `api/spi/stigmergy/` alongside SwarmCoordinationListener
**Alternatives:**
- `api/spi/coordination/` — role lifecycle events connect to coordination model, but the primary owner (StigmergyCoordinator) is a stigmergy component
**Rationale:** The listener's primary consumer is StigmergyCoordinator (D4). Work state is a stigmergy concern (D1). Following the pattern: model types and their SPIs live in the same domain subpackage.
**Depends on:** D1 (AgentWorkState in stigmergy), D2 (single interface)
**Sources:** SwarmCoordinationListener.java (api/spi/stigmergy/), SwarmProvisioningAdvisor.java (api/spi/stigmergy/)
**Exploration:** quick
**Status:** captured

## D4: Work state ownership — StigmergyCoordinator

**Choice:** StigmergyCoordinator extends to track per-agent AgentWorkState alongside existing AgentLifecycleState
**Alternatives:**
- New AgentWorkStateTracker — separate tracker with its own agent map creates two sources of truth for "which agents exist"; coordination between tracker and coordinator adds complexity
- WorkerExecutionOrchestrator fire-and-forget — simpler but loses queryability; can't ask "what state is agent X in?"
**Rationale:** StigmergyCoordinator already owns per-agent state via ConcurrentHashMap<String, AgentState>. Adding workState to AgentState (or a parallel map) keeps all per-agent state in one place. The coordinator already fires SwarmCoordinationListener events — adding AgentWorkLifecycleListener dispatch is natural.
**Trade-offs:** StigmergyCoordinator gains more responsibility. Acceptable — it's the canonical per-agent state machine, and work state is a natural extension of "what this agent is doing."
**Sources:** StigmergyCoordinator.java (runtime-core/.../stigmergy/), AgentState.java (api/model/stigmergy/)
**Exploration:** quick
**Status:** captured

## D5: Integration point — WorkerExecutionOrchestrator

**Choice:** WorkerExecutionOrchestrator fires state transitions by calling StigmergyCoordinator
**Alternatives:**
- WorkerScheduleEventHandler + WorkflowExecutionCompletedHandler — state changes at handler boundaries in runtime-core; higher-level but further from actual execution; ASSIGNED→EXECUTING boundary is unclear from here
- Both layers (scattered across three classes) — most precise but scattered integration
**Rationale:** WorkerExecutionOrchestrator already fires WORKER_EXECUTION_STARTED and publishes outcomes. It has access to agentId and caseId. Adding state transitions here (ASSIGNED at entry, EXECUTING after setup, IDLE at completion) is natural and contained. Orchestrator is in common-core, which depends on api where the SPI lives.
**Trade-offs:** common-core needs to call StigmergyCoordinator (runtime-core). This means the orchestrator needs an injectable interface for state transitions, not a direct dependency on the runtime class. Acceptable — we can add transition methods to an existing api-level interface or create a thin one.
**Depends on:** D4 (StigmergyCoordinator owns state)
**Sources:** WorkerExecutionOrchestrator.java (common-core/.../executor/), WorkerScheduleEventHandler.java (runtime-core/.../handler/)
**Exploration:** quick
**Status:** captured

## D6: Role lifecycle — SPI surface only, integration deferred

**Choice:** Define `onRoleAdopted` and `onRoleReleased` on AgentWorkLifecycleListener now. Integration points (who calls them) deferred to future issues (Hive topology, agency#378+).
**Alternatives:**
- Omit role methods entirely — but the issue explicitly requires them, and they're needed for MOISE+ alignment (eidos#216)
- Integrate now with TopologyDescriptor changes — premature; topology changes are static declarations, not runtime adopt/release events
**Rationale:** The SPI surface is cheap to define (default no-op methods). The firing code depends on dynamic group management which doesn't exist yet. Defining the SPI now gives future work a stable contract to code against.
**Trade-offs:** Dead SPI methods until group management lands. Acceptable — defaults mean no consumer impact.
**Depends on:** D2 (single interface includes role events)
**Sources:** engine#1285 issue body §Integration — Role Lifecycle, eidos#216
**Exploration:** quick
**Status:** captured
