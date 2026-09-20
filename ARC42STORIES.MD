# casehub-engine — ARC42STORIES.MD

**Spec:** Arc42Stories v0.1
**Profile:** CaseHub — Foundation tier
**Profile ref:** `../parent/docs/arc42stories-casehub-profile.md` · fallback: `https://raw.githubusercontent.com/casehubio/parent/main/docs/arc42stories-casehub-profile.md`
**Build position:** After casehub-work-core, optionally after casehub-ledger and casehub-eidos-api. The top of the foundation stack.
**Consumed by:** All application-tier harness apps (via YamlCaseHub / CaseHubRuntime), claudony (implements engine SPIs)
**Depends on:** casehub-work-core (WorkItemLifecycleEvent + HumanTaskTarget), optionally casehub-ledger (CaseLedgerEntry), optionally casehub-eidos-api.

---

## §1 Introduction and Goals

### Description

`casehub-engine` is the hybrid choreography+blackboard orchestration engine. It executes CasePlanModel definitions — declarative goal+binding case definitions expressed in YAML — by evaluating binding conditions against an accumulated CaseContext (the blackboard) and dispatching work when conditions are satisfied.

Core capabilities: `CaseDefinition` (YAML/Java definition: goals, bindings, capabilities), `CaseHubReactor` (event-loop reactor — evaluates binding conditions on CaseContext change), `CaseContextChangedEvent` (fired on every blackboard update — triggers reactor), `GoalReachedEvent` (fired when a goal condition is satisfied), `WAITING` state (durable suspension pending async result — survives restarts via PendingWorkRegistry), `CaseLedgerEntry` (per-binding-dispatch audit record via casehub-ledger integration), composable routing signal architecture (workload, trust, experience, personality, semantic providers), `YamlCaseHub` (base class for loading YAML case definitions from classpath).

See `docs/guides/contributor-guide.md` §Module Structure for the full module table. Key layers: foundation (`api`, `worker-api`), core (`*-core` framework-neutral modules), framework (Quarkus and Spring binding shims), optional modules (`flow`, `a2a`, `mcp`, `react`, `queue`, etc.).

### Stakeholders

| Stakeholder | Interest |
|---|---|
| Application-tier harness apps | Consume this module's foundation primitives |
| Platform team | Maintain correctness and backward compatibility |
| LLM sessions (Claude) | Navigate architecture and implement extensions confidently |

### Quality Goals

| Goal | Scenario |
|---|---|
| Content-driven routing | Binding conditions evaluate accumulated CaseContext — routing adapts to what code/data actually contains, not what authors label |
| Durable suspension | WAITING state survives Quarkus restarts via PendingWorkRegistry — no case is lost on restart |
| Declarative goals | Goals state what must be achieved; bindings state the mechanism. These are independent — binding structure can change without changing goal definition |

---

## §2 Constraints

### Platform
Java 21 (on Java 26 JVM), Quarkus 3.32.2. Published to GitHub Packages as `io.casehub:casehub-engine:0.2-SNAPSHOT`.

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install
```

### Dependencies
Resolved from GitHub Packages: `https://maven.pkg.github.com/casehubio/*`. CI authentication via `GITHUB_TOKEN`.

---

## §3 Context and Scope

🔲 C4Context diagram — add in dedicated session.

**Boundary rules:** casehub-engine owns only the concerns described in §1. Domain logic (business rules, application-specific workflows) belongs in application-tier repos (devtown, aml, clinical, life, drafthouse, quarkmind).

See `../parent/docs/PLATFORM.md` Capability Ownership table for the full boundary map.

---

## §4 Solution Strategy

### Multi-framework split

**Before:** All logic lived in Quarkus CDI modules — framework annotations and dependency injection were entangled with business logic throughout.
**After:** Each core layer has three modules: `-core` (framework-neutral, all logic), Quarkus CDI bindings, and Spring bindings. The `-core` module holds all domain logic; framework modules are thin wiring shims.

What this changes:
- **Framework portability** — the same engine logic runs on Quarkus or Spring Boot without duplication
- **Testability** — core modules are unit-testable without a CDI container
- **Dependency clarity** — framework concerns are isolated to a few wiring files per layer

Core layers following this pattern: common, runtime, planning, resilience, ledger, engine-support. See `docs/guides/contributor-guide.md` §Module Structure for the complete module table.

### Hybrid choreography + planning model

Two execution paths, both triggered by CaseContext changes:

- **Choreography** (`CaseContextChangedEventHandler`) — evaluates binding `when()` conditions against the CaseContext blackboard, selects eligible bindings via `LoopControl`, dispatches by target type (CapabilityTarget, HumanTaskTarget, SubCaseTarget, SignalTarget, JudgmentTarget). Pure reactive — no planning state.
- **Planning** (`PlanningStrategyLoopControl`) — adds compound PlanItemDefinitions with completion semantics (All, MofN, FirstWins), scoped binding gating, and strategy-driven dispatch. Strategies: `ChoreographyStrategy` (fire all eligible), `SequentialPlanningStrategy` (one at a time).

Planning is additive — cases without a `planningStrategy` on their CaseDefinition use pure choreography. Cases with planning gain compound lifecycle (entry/exit conditions, repeatable compounds), goal decomposition (HTN/GOAP/LLM), and plan adaptation (repair, optimization, meta-reasoning).

See `docs/guides/contributor-guide.md` §Planning Module Architecture for handler wiring.

### Four-tier worker execution

| Tier | Mechanism | Durability | Use case |
|---|---|---|---|
| 1 — Execution | `WorkerRuntime` in-worker composition | None (in-memory) | Function chaining, sub-case spawn |
| 2 — Simple plan | `SequentialPlanningStrategy` | PlanItem state | Linear capability chains |
| 3 — Complex plan | Serverless Workflow (`casehub-engine-flow`) | SWF state | Durable branching, compensation |
| 4 — Multi-case | Blocks patterns (`casehub-blocks-engine-adapter`) | Cross-case | Agentic orchestration patterns |

Tiers 2–3 share the `PlanningStrategy.select()` seam. Tier 4 runs inside the `WorkerFunctionHandler` pipeline.

### Composable routing signal architecture

**Before:** Routing strategies were monolithic — each strategy (trust, load, CBR) was a standalone implementation with duplicated candidate filtering logic.
**After:** `ComposableAgentRoutingStrategy` blends scores from independent `RoutingSignalProvider` implementations. Each provider scores candidates independently; the compositor computes a weighted sum.

Built-in signal providers: workload (availability), trust (ledger maturity), experience (CBR history), personality (JPAF cognitive alignment), semantic (embedding similarity). Per-case weight configuration via `CaseDefinition.routingSignalWeights`.

What this changes:
- **New signals plug in** — implement `RoutingSignalProvider`, the compositor picks it up
- **Per-case tuning** — YAML configures which signals matter for each case type
- **No strategy explosion** — N signals don't need N! combined strategies

See `docs/guides/contributor-guide.md` §Routing Architecture for the full pipeline and signal provider details.

### @DefaultBean displacement

All SPI no-op defaults use `@DefaultBean @ApplicationScoped` (Quarkus) or equivalent Spring conditional. Consumer implementations displace automatically — no `selected-alternatives` configuration needed. This is the universal extension mechanism: define interface in `api/spi/`, provide `@DefaultBean` no-op in `runtime/internal/worker/`, consumer provides `@ApplicationScoped` implementation.

Strategy SPIs additionally follow the `NamedStrategy` convention — `id()` method, resolved by `EngineStrategyResolver`, selectable per CaseDefinition via YAML.

### Stigmergic coordination model

Agents coordinate indirectly through shared environment, inspired by biological stigmergy:

- **Signals** — temporal values with exponential decay and reinforcement. Agents deposit named signals with strength; signals lose half their strength every `halfLife`. Multiple agents depositing the same signal reinforce it.
- **Observations** — `EnvironmentObserver` instances watch CaseContext state and signal changes, producing structured `Observation` results. Classical observers: threshold, correlation, temporal sequence, signal strength.
- **Interests** — declarative observation registration via `InterestDeclaration`. Agents describe what they want to observe; the engine creates the appropriate observer. `InterestLandscape` provides anonymous aggregate view of collective attention.
- **Neighbors** — read-only query facade over existing registries. Proximity is emergent from shared activity (coactive PlanItems, shared interests, shared signals, complementary produced/watched keys).
- **Rules** — per-agent condition→action rules evaluated after observations. Actions: deposit signals, register interests, write context keys.
- **Convergence** — system-level detection of quiesced activity rates across four metrics (dispatches, signal deposits, context mutations, evaluation cycles). Budget enforcement caps cumulative metrics.

See `docs/guides/contributor-guide.md` for implementation details of each subsystem.

---

## §5 Building Block View

### Module Layers

The engine is organized in seven tiers. Each tier has a clear dependency direction: higher tiers depend on lower, never the reverse.

**Tier 1 — Foundation** (`casehub-engine-api`, `casehub-worker-api`). SPI interfaces, domain model (`Binding`, `CaseDefinition`, `Goal`, `Milestone`, `Capability`), `Agent` wrapper, routing SPIs, DAG/HTN plan types. Pure Java — no CDI, no Spring, no JPA.

**Tier 2 — Core** (`common-core`, `runtime-core`, `planning-core`, `resilience-core`, `ledger-core`, `engine-support-core`). Framework-neutral logic: domain objects (`CaseMetaModel`, `CaseInstance`), persistence SPIs, event types, handler implementations, scheduler abstractions. No CDI annotations, no Spring annotations.

**Tier 3 — Framework** (Quarkus: `common`, `runtime`, `planning`, `resilience`, `ledger`; Spring: `common-spring`, `runtime-spring`, `planning-spring`, `resilience-spring`, `ledger-spring`, `engine-support-spring`). Thin wiring shims over Tier 2 core modules — CDI qualifiers, producers, configuration bindings.

**Tier 4 — Scheduler** (`scheduler-quartz`, `scheduler-dbscheduler`). Pluggable worker execution backends. `@WorkerBackend @Priority` CDI qualifier selects the active backend. db-scheduler (`@Priority(10)`) wins when both are on the classpath.

**Tier 5 — Persistence** (`persistence-hibernate`, `persistence-memory`). JPA/Panache (PostgreSQL, RLS) and in-memory thread-safe implementations. Activated via `quarkus.arc.selected-alternatives` in test properties.

**Tier 6 — Optional modules** (`flow`, `a2a`, `mcp`, `react`, `work-cloudevent`, `queue`, `engine-ai`, `actor-state`, `inbound`, `annotations`, `graphql`, `generator`, `rest`). Activated by classpath presence. Each follows the `@DefaultBean` displacement pattern — absent module = transparent no-op.

**Tier 7 — Examples** (paired `*-dsl` and `*-annotated` modules). Demonstrate each pattern: choreography, sequential, GOAP, CBR ensemble, LLM decomposition, humanTask, subCase, A2A, MCP, plus scenario-specific annotated examples.

See `docs/guides/contributor-guide.md` §Module Structure for the complete module table with artifact IDs, folders, and purposes.

---

## §6 Runtime View

🔲 Key runtime scenarios — complete in dedicated session.

---

## §7 Deployment View

Published to GitHub Packages. CI: GitHub Actions. No GraalVM native target (foundation modules).

---

## §8 Crosscutting Concepts

References to governing protocols:

| Concern | Protocol / Reference |
|---|---|
| Module structure | `docs/protocols/universal/module-tier-structure.md` |
| Flyway migrations | `docs/protocols/universal/flyway-migration-rules.md` |
| CDI displacement (`@DefaultBean`) | `docs/protocols/casehub/alternative-extension-patterns.md` |
| Capability ownership | `../parent/docs/PLATFORM.md` Capability Ownership table |
| Architectural patterns | `../parent/docs/ARCHITECTURE.md` |

### Anti-patterns

| Symptom | Cause | Fix |
|---|---|---|
| Silent tenant mismatch in tests | Injected repo by concrete class instead of SPI interface | Use `@Inject CaseInstanceRepository`, never the concrete impl |
| `Tests run: 0` with no error | Test class named `*IT.java` | Rename to `*Test.java` — failsafe picks up IT instead of surefire |
| `@ObservesAsync` silently not invoked | Unreliable in `@QuarkusTest` | Inject bean, call observer method directly |
| Silent timeout in tests | `casehub-ledger` on classpath without H2 config | Add H2, `NoOpLedgerEntryRepository`, exclude capture beans |
| New `NamedStrategy` not discovered | `EngineStrategyResolver` constructor not updated | Add typed `Instance<>` injection for the new SPI type |
| Two stores created, tenant data split | `@Inject InMemoryCaseInstanceRepository` instead of SPI | `@Alternative @Priority(1)` test wrappers substitute only when injected by interface |
| JQ expression returns wrong data | Evaluating full layer document instead of working layer | All JQ evaluates `context.layer(ContextLayer.WORKING).asJsonNode()` |

See `docs/guides/contributor-guide.md` §CDI Conventions and §Test Conventions for detailed guidance.

---

## §9 Journeys and Chapters

🔲 **Bootstrap — full Journeys and Chapters pending dedicated session.**

Foundation modules use Chapters to track delivery milestones of the foundation capability itself (transport types, SPI increments, new adapters) rather than integrating external foundation modules.

Read `../parent/docs/arc42stories-spec.md §4 Solution Strategy` — "Foundation: Chapters = delivery milestones of the foundation capability itself."

The full §9 entry requires:
- Reading DESIGN.md (or equivalent) for the delivery arc
- Mapping milestones to Chapters
- Writing §9.4 Layer/Chapter entries with all 18 required subsections

Active development branch: `issue-1104-hive-mind` (stigmergic coordination — signals, observations, interests, neighbors, rules, convergence).

---

## §10 Architectural Decisions

Per the Arc42Stories spec: only decisions not captured inline elsewhere. Should be sparse.

🔲 Cross-cutting decisions — complete in dedicated session.

### 2026-06-22 · Agent Mesh SPIs extracted to engine-api (parent#93, engine#550)

New package `io.casehub.api.spi.mesh` added to `casehub-engine-api`. Previously these types lived in `claudony-casehub`, making them unavailable to other agent implementations without duplication. Extracted to Tier-1 engine-api so any agent (claudony, openclaw, future) shares canonical definitions.

Types added: `CaseChannelLayout` SPI + `ChannelSpec` record + `named()` factory; `NormativeChannelLayout` (3-channel: work/observe/oversight per PP-20260604-a7ad99); `SimpleLayout` (2-channel: work/observe, no governance gate); `MeshParticipationStrategy` SPI + `MeshParticipation` enum + `named()` factory; `ActiveParticipationStrategy`, `ReactiveParticipationStrategy`, `SilentParticipationStrategy`.

All types are pure Java (Tier-1 — no CDI, no JPA, no Quarkus). 446 tests green post-merge. Garden protocol `normative-channel-layout-single-source.md` promoted from openclaw repo-scope to platform-scope.

*engine commit: ab406f8e — fix(#550): address code review findings*

---

### 2026-06-04 · FlowWorker bridge — module boundary, SPI extraction, non-blocking dispatch

`WorkOrchestrator` was a concrete `@ApplicationScoped` class in `runtime`. Extracted as an interface to `casehub-engine-common/spi/` — the same placement as `CaseInstanceRepository` and `EventLogRepository` — because it uses `CaseInstance` and placing it in `api/spi/` would create a circular dependency (`api` ← `common` ← `api`). The concrete `DefaultWorkOrchestrator` remains in `runtime`; CDI resolves the implementation at startup without flow needing a runtime compile dependency.

New module `casehub-engine-flow` depends on `casehub-engine-common` only (not `runtime`). This is the correct isolation boundary: `FlowWorkerExecutor`, `CasehubDispatch`, `CasehubCallableTaskBuilder`, and `FlowExecutionRegistry` all live here and give the flow module visibility into the engine's domain types without pulling in the full runtime event loop.

The Workflow execution path in `QuartzWorkerExecutionJob` is now non-blocking: for `Worker(Workflow)`, the Quartz thread fires `workflowExecutor.execute()` and returns immediately. Success publishes `WORKER_EXECUTION_FINISHED`; failure publishes `WorkflowExecutionFailed` on the event bus. `QuartzWorkerExecutionJobListener` handles the failure via `@ConsumeEvent(blocking=true)`, persists `WORKER_EXECUTION_FAILED`, and calls `maybeRescheduleWorker(WorkerRetryContext)`. `WorkerRetryContext` replaces `JobExecutionContext` as the value object, allowing both sync and async failure paths to share one retry implementation.

`WORKFLOW_STEP_DISPATCHED` / `COMPLETED` / `FAILED` are emitted via `whenComplete` (always fires, preserving exception propagation). The YAML extension point is `CallableTaskBuilder<CallFunction>` via Java SPI — `call: casehub:dispatch` parses as `CallFunction` (in `io.serverlessworkflow.api.types`, not the `.func` experimental subpackage). `CasehubCallableTaskBuilder` uses a `ThreadLocal<String>` to pass capability between `init()` and `build()` because `ServiceLoader.Provider.get()` caches the builder instance.

*engine commit: 790a103 — feat(flow): FlowWorker ↔ WorkOrchestrator bridge — engine#206*

---

## §11 Quality Requirements

See §1 Quality Goals for the primary scenarios.

🔲 Full quality requirements table — complete in dedicated session.

---

## §12 Risks and Technical Debt

🔲 Active risks and technical debt — complete in dedicated session.

---

## §13 Glossary

| Term | Definition |
|---|---|
| `CasePlanModel` | YAML definition of a case: goals (what must be achieved) + bindings (conditions + capability dispatch). Evaluated by CaseHubReactor. |
| `CaseContext` | Shared blackboard for a case instance. Binding conditions read it; agent outputs write to it. Append-only during case lifetime. |
| `Binding` | A condition + capability pair. When the JQ condition evaluates true against CaseContext, the engine dispatches work to an agent with the named capability. |
| `WAITING state` | Durable engine state when a case is suspended pending an async result (human approval, CI run). Persisted via PendingWorkRegistry. |
| `YamlCaseHub` | Base class that loads a YAML case definition from the classpath and registers it with `CaseDefinitionRegistry`. |
| `ComposableAgentRoutingStrategy` | Default agent routing — blends scores from independent `RoutingSignalProvider` implementations (workload, trust, experience, personality, semantic). |
| `NamedStrategy` | Marker interface for pluggable strategies resolved by `EngineStrategyResolver`. All per-case-selectable routing and decomposition strategies extend this. |
| `@DefaultBean` | Quarkus CDI annotation; yields automatically to any consumer-provided `@ApplicationScoped` implementation. The universal extension mechanism for engine SPIs. |
| `WorkerResult` | Typed worker return value carrying output, optional `PlannedAction`, and outcome semantics (`Success`, `Declined`, `Failed`, `Expired`, `Completed`). |
| `Compound` | Container `PlanItemDefinition` with children, `CompletionSemantics` (All/MofN/FirstWins), scoped bindings, and entry/exit conditions. Replaces the retired Stage concept. |
| `DagPlan<T>` | Immutable validated directed acyclic graph for parallel execution. Factories: `singleton`, `sequence`, `parallel`, `fromNodes`. |
| `CbrConfig` | Per-case CBR retrieval configuration on `CaseDefinition`: feature extraction, similarity thresholds, retrieval timing, domain, adaptation. |

---

*Bootstrap document — generated 2026-06-02. Updated 2026-09-17 with §4 Solution Strategy, §5 Building Block View, §8 Anti-patterns, §13 Glossary. §3, §6, §9 require dedicated sessions.*
