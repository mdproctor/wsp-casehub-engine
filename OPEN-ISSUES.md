# casehub-engine — Open Issues

*Generated 2026-06-28 — 89 open issues*

## Link Types

| Relationship | Meaning |
|---|---|
| blocked by #N | Cannot start until #N completes |
| blocks #N | #N is waiting on this issue |
| depends on #N | Requires #N (softer than blocked by) |
| child of epic #N | Belongs to parent epic #N |
| part of #N | Grouped under #N |
| cross-repo | Dependency in another casehubio repo |

## Scale / Complexity

**Scale:** XS (lines) / S (single class) / M (multi-class) / L (substantial feature) / XL (major rework)  
**Complexity:** Low (clear path) / Med (some unknowns) / High (significant design required)

## Epics (23)

| # | Title | Scale | Cmplx | Linked | Notes |
|--:|-------|:-----:|:-----:|--------|-------|
| [2](https://github.com/casehubio/engine/issues/2) | Pluggable Expression Evaluation Framework | L | Med | — | Goal |
| [77](https://github.com/casehubio/engine/issues/77) | Epic: Blackboard Architecture Evolution — Research-Identified Future Improvements | L | High | child of epic [#30](https://github.com/casehubio/engine/issues/30) | Captures future improvements to the blackboard architecture identified through academic... |
| [84](https://github.com/casehubio/engine/issues/84) | Epic: Milestone, Stage, and Goal — Full Conceptual Alignment | L | High | — | Tracks the work to fully align the Milestone, Stage, and Goal concepts into a |
| [101](https://github.com/casehubio/engine/issues/101) | feat: LLM supervisor mode — extending casehub's orchestration model with dynamic LLM-driven planning | L | High | — | LangChain4j provides a supervisor mode for multi-agent LLM orchestration. This issue captures a... |
| [102](https://github.com/casehubio/engine/issues/102) | Epic: casehub Ecosystem Use Cases — Enterprise AI Agent Orchestration Patterns | L | High | — | This epic groups the concrete use cases where the casehub ecosystem — casehub/casehub-engine,... |
| [107](https://github.com/casehubio/engine/issues/107) | feat: elastic research teams — multi-agent mesh with Qhorus peer-to-peer coordination | L | High | — | Research tasks benefit from multiple specialised agents working in parallel — each investigating... |
| [108](https://github.com/casehubio/engine/issues/108) | feat: Long-Running Case Management — persisting cases across days, weeks, and months | L | High | — | Current AI agent frameworks operate within a process lifetime. A LangChain4j agent call, an... |
| [115](https://github.com/casehubio/engine/issues/115) | feat: Human Escalation in Agent Pipelines — first-class human workers with full provenance | L | High | — | Automated agents handle the majority of AI-assisted work efficiently: document classification,... |
| [116](https://github.com/casehubio/engine/issues/116) | feat: compliance and audit workflows for regulated industries | L | High | — | Financial services, healthcare, legal, and pharmaceutical industries require audit trails that... |
| [166](https://github.com/casehubio/engine/issues/166) | feat: AI-Assisted Analytical Writing Review — Editorial Pipeline App with Layered Capability Model | L | High | child of epic [#102](https://github.com/casehubio/engine/issues/102) | This issue captures a concrete application of the casehub ecosystem: a guided AI-assisted... |
| [201](https://github.com/casehubio/engine/issues/201) | epic: adaptive execution architecture — four models, context bridging, lineage (ADR-0007) | XL | High | — | Implements the full architecture defined in ADR-0007: four co-equal execution models feeding a... |
| [202](https://github.com/casehubio/engine/issues/202) | epic: agnostic routing layer and execution kernel | L | High | — | Formalises the shared execution path as an explicit, agnostic execution kernel. All four routing... |
| [203](https://github.com/casehubio/engine/issues/203) | epic: ContextBridge protocol and worker-level context selection | L | High | child of epic [#209](https://github.com/casehubio/engine/issues/209) | Defines the `ContextBridge<T>` SPI so any execution model can declare its own context adapter.... |
| [204](https://github.com/casehubio/engine/issues/204) | epic: scope propagation, return-path tracking, and context write-through | L | High | — | Ensures context flows correctly down through nested execution boundaries and back up via... |
| [205](https://github.com/casehubio/engine/issues/205) | epic: complete causal lineage across all execution boundaries | L | High | child of epic [#204](https://github.com/casehubio/engine/issues/204) | Ensures every unit of work at every level emits a `CaseLedgerEntry` node so `causedByEntryId`... |
| [207](https://github.com/casehubio/engine/issues/207) | epic: orchestration — rules-based (Drools/DMN → WorkOrchestrator) | L | High | — | Enables a rules engine (Drools, DMN) to evaluate case context and produce an ordered sequence of... |
| [208](https://github.com/casehubio/engine/issues/208) | epic: plan-based execution — PlanExecutor, Plan data model, PlanSource SPI | L | High | — | Introduces plan-based execution as a first-class execution model. A planner (human, LLM, or... |
| [209](https://github.com/casehubio/engine/issues/209) | epic: langchain4j-agentic integration — AgenticScopeBridge, CasehubPlanner, AgentListener | L | High | — | Integrates langchain4j-agentic (`quarkus-langchain4j-agentic`) with casehub-engine with zero... |
| [210](https://github.com/casehubio/engine/issues/210) | epic: cancellation, timeout, and error recovery across execution boundaries | L | High | — | Defines how cancellation, deadline propagation, and error recovery work when execution spans... |
| [211](https://github.com/casehubio/engine/issues/211) | epic: observability — cross-boundary lineage queries, context stack visibility, A2A tracing | L | High | — | Provides query and observability APIs over the complete causal lineage graph, the runtime... |
| [230](https://github.com/casehubio/engine/issues/230) | feat: normative layer audit — apply speech-act vocabulary to all inter-component communications | XL | High | — | The ecosystem should eat its own dogfood: every communication boundary between components should... |
| [445](https://github.com/casehubio/engine/issues/445) | epic: full Drools integration — typed blackboard memory, expression engine, rules-based orchestration | — | — | blocks [#5](https://github.com/casehubio/engine/issues/5); blocks [#446](https://github.com/casehubio/engine/issues/446); blocks [#207](https://github.com/casehubio/engine/issues/207); depends on [#446](https://github.com/casehubio/engine/issues/446) | Delivers full Drools integration across three layers: pluggable expression language... |
| [501](https://github.com/casehubio/engine/issues/501) | Epic: Semantic failure routing — DECLINED/FAILED outcome handling for application-tier failure cascades | XL | High | — | Application-tier harnesses (devtown, aml, clinical, life) need to implement failure cascades —... |

## Active Backlog (51)

| # | Title | Scale | Cmplx | Linked | Notes |
|--:|-------|:-----:|:-----:|--------|-------|
| [10](https://github.com/casehubio/engine/issues/10) | Deterministic Replay / Restate Execution Model | XL | High | — | **Goal** |
| [22](https://github.com/casehubio/engine/issues/22) | SLA/SLO Tracking and Deadline Monitoring | L | Med | — | **Description** |
| [23](https://github.com/casehubio/engine/issues/23) | External Event Ingestion | L | Med | — | **Description** |
| [189](https://github.com/casehubio/engine/issues/189) | experiment: normative layer interoperability test — Tower of Babel vs formal semantics | L | High | — | Without a normative layer, independently implemented LLM agents will express the same... |
| [212](https://github.com/casehubio/engine/issues/212) | feat: casehub-work multi-instance WorkItem spawning — lineage, context, and casehub-work-adapter integration | L | High | child of epic [#106](https://github.com/casehubio/engine/issues/106) | quarkus-work (quarkiverse/quarkus-work Epic #106) added multi-instance WorkItem spawning: a... |
| [236](https://github.com/casehubio/engine/issues/236) | feat: casehub-examples/ — working examples module ported from casehub-poc (blocks poc archival) | M | Low | — | The `casehub-examples/` module does not yet exist in casehub-engine. It is a prerequisite for... |
| [237](https://github.com/casehubio/engine/issues/237) | feat: long-lived workers with lifecycle scopes (CASE / STAGE / BINDING) | L | High | — | Phase 3 migration item from casehub-poc. Workers in the engine currently have no structured... |
| [238](https://github.com/casehubio/engine/issues/238) | feat: JavaBeanCaseFile<T> — typed POJO-backed CaseContext | M | Med | — | Phase 3 migration item. Currently `CaseContext` is JSON/`JsonNode`-backed. This issue tracks a... |
| [247](https://github.com/casehubio/engine/issues/247) | Implement SecretManager SPI for K8s/Vault integration | L | Low | — | Currently, environment variable resolution in YAML configuration uses a simple `${VAR}`... |
| [284](https://github.com/casehubio/engine/issues/284) | Implement Kubernetes Secrets integration for SecretManager SPI | M | Low | — | Implement Kubernetes Secrets integration for `SecretManager` SPI. |
| [285](https://github.com/casehubio/engine/issues/285) | Implement Kubernetes ConfigMaps integration for ConfigManager SPI | M | Low | — | Implement Kubernetes ConfigMaps integration for `ConfigManager` SPI. |
| [286](https://github.com/casehubio/engine/issues/286) | Implement HashiCorp Vault integration for SecretManager SPI | M | Low | — | Implement HashiCorp Vault integration for `SecretManager` SPI. |
| [287](https://github.com/casehubio/engine/issues/287) | Implement secret caching, rotation, and audit logging | M | Low | — | Implement secret caching, rotation, and audit logging for `SecretManager` SPI. |
| [327](https://github.com/casehubio/engine/issues/327) | HumanTaskTarget: support runtime-evaluated expiresIn — static YAML Duration can't vary per case instance | M | Med | — | `HumanTaskTarget.expiresIn()` is a static `Duration` parsed from the YAML binding definition at... |
| [340](https://github.com/casehubio/engine/issues/340) | design: no A2A entry point for CaseInstance — external A2A orchestrators cannot start CaseHub cases | M | Low | — | Audit finding #26 from casehubio/parent#4 (platform coherence analysis — M7 batch). |
| [364](https://github.com/casehubio/engine/issues/364) | design: PlanItem timing race — PENDING window allows duplicate dispatch for HumanTask/SubCase bindings | M | Med | — | When two `CONTEXT_CHANGED` events arrive in rapid succession, a binding whose PlanItem is still... |
| [383](https://github.com/casehubio/engine/issues/383) | feat: oversight response loop — COMMAND from human re-triggers agent routing | M | Med | — | When `AgentRoutingEscalationHandler` posts a QUERY to the case oversight channel (engine#377),... |
| [384](https://github.com/casehubio/engine/issues/384) | feat: PlanItem state during agent routing escalation | M | Med | — | When `AgentRoutingEscalationHandler` fires (engine#377), the PlanItem stays PENDING. This has... |
| [414](https://github.com/casehubio/engine/issues/414) | docs: complete ARC42STORIES.MD — Foundation tier, delivery milestones from blogs + git log | — | — | — | casehub-engine is an Orchestration-tier module — the hybrid choreography+blackboard engine that... |
| [419](https://github.com/casehubio/engine/issues/419) | feat: CaseContextProvider SPI — pluggable context backing store for LangChain4j AgenticScope interoperability | — | — | — | Extracted from #101 (LLM supervisor mode), where it was identified as a distinct concern... |
| [431](https://github.com/casehubio/engine/issues/431) | feat: workflow worker PlannedAction support — declare actions from Serverless Workflow steps | — | — | — | engine#402 added `ActionRiskClassifier` SPI with `WorkerResult` as the return type for function... |
| [433](https://github.com/casehubio/engine/issues/433) | feat: persist pendingActionGate in CaseInstanceEntity for restart resilience | — | — | — | engine#402 stores `pendingActionGate` in-memory only on `CaseInstance`. If the engine restarts... |
| [438](https://github.com/casehubio/engine/issues/438) | fix: extract OversightGateService from casehub-openclaw to casehub-engine-api | — | — | — | `OversightGateService` is currently implemented in `casehub-openclaw` but its intended home is... |
| [439](https://github.com/casehubio/engine/issues/439) | feat: dynamic title, scope, and expiresIn for humanTask binding via JQ expression | — | — | — | engine#387 adds dynamic `candidateGroups` and `candidateUsers` via JQ expression, following the... |
| [440](https://github.com/casehubio/engine/issues/440) | Dynamic case definition registration — allow running cases to generate and launch new case types at runtime | L | High | — | Currently all case definitions are CDI-discovered at `StartupEvent` via `Instance<CaseHub>`. The... |
| [442](https://github.com/casehubio/engine/issues/442) | design: universal pluggable routing strategy architecture across the platform | — | — | — | Routing decisions in the platform — who receives a task, which worker gets a case, which... |
| [446](https://github.com/casehubio/engine/issues/446) | feat(drools): WorkingMemoryBridge — typed bridge from CaseContext panels to Drools WorkingMemory | M | Med | depends on [#80](https://github.com/casehubio/engine/issues/80); child of epic [#445](https://github.com/casehubio/engine/issues/445) | Part of epic #445 — full Drools integration. Depends on #80 (memory stratification) and #81... |
| [448](https://github.com/casehubio/engine/issues/448) | feat: Worker(Plan.of(...)) — plan-based worker function type | — | — | — | Design: `specs/issue-413-sx-scale-batch/2026-06-08-hybrid-execution-design.md` (Option B) |
| [449](https://github.com/casehubio/engine/issues/449) | feat: plan binding target type in CaseDefinition YAML DSL | — | — | — | Design: `specs/issue-413-sx-scale-batch/2026-06-08-hybrid-execution-design.md` (Option A) |
| [454](https://github.com/casehubio/engine/issues/454) | feat(acl): add security-impl module with JPA-backed flat grant enforcement | — | — | — | ACL authorization model design approved in... |
| [456](https://github.com/casehubio/engine/issues/456) | feat(acl): thread dispatch identity via PropagationContext | — | — | — | ACL authorization model design approved in... |
| [457](https://github.com/casehubio/engine/issues/457) | feat(acl): extend case definition YAML schema with authorization section | — | — | — | ACL authorization model design approved in... |
| [458](https://github.com/casehubio/engine/issues/458) | epic(acl): ACL authorization model — flat grant access control for casehub-engine | — | — | — | Implement flat grant access control in casehub-engine. Direct `(actor, resource, action)` grants... |
| [461](https://github.com/casehubio/engine/issues/461) | feat: composite WorkerExecutionManager for multi-worker co-deployment | — | — | — | `workers-camel` and `workers-http` (in progress) both provide `@ApplicationScoped`... |
| [478](https://github.com/casehubio/engine/issues/478) | feat: CaseRetriever integration at plan creation — bridge CBR Retrieve step to ImplementationRoutingStrategy | — | — | — | Part of the CBR platform capability — casehubio/parent#227. |
| [483](https://github.com/casehubio/engine/issues/483) | feat: signalAndAwait() — bulk context update + settlement detection for synchronous callers | — | — | part of [#490](https://github.com/casehubio/engine/issues/490) | `CaseHubRuntime.signal(UUID, String, Object)` is fire-and-forget and single-path. Two gaps for... |
| [484](https://github.com/casehubio/engine/issues/484) | feat: SequenceWorker — lightweight sequential orchestration of workers and sub-cases without Quarkus Flow | — | — | part of [#490](https://github.com/casehubio/engine/issues/490) | Quarkus Flow (Serverless Workflow) provides durable, resilient workflow orchestration. But many... |
| [485](https://github.com/casehubio/engine/issues/485) | feat: WorkerRuntime — execution context SPI for orchestrating workers that need to spawn sub-cases or signal the case | — | — | part of [#490](https://github.com/casehubio/engine/issues/490) | `SequenceWorker` (#484) needs to spawn sub-cases as steps in a sequence and wait for them to... |
| [490](https://github.com/casehubio/engine/issues/490) | Epic: QuarkMind-driven engine API expansion — sync execution, sequencing, and orchestration primitives | — | — | — | Four engine capabilities surfaced by the QuarkMind → casehub-engine migration. QuarkMind's... |
| [505](https://github.com/casehubio/engine/issues/505) | feat: AgentRoutingStrategy — consume eidos capability metadata and CBR patterns | L | High | — | casehubio/parent#258 |
| [507](https://github.com/casehubio/engine/issues/507) | feat: human task / approval gate — engine-level case step type | — | — | — | Support for human task / approval gate case steps — a case step that pauses execution and waits... |
| [510](https://github.com/casehubio/engine/issues/510) | feat: Case-level SLA — time-triggered binding for overall case deadline | M | Med | — | The engine has no case-level timeout mechanism. WorkItem SLA (casehub-work) handles individual... |
| [528](https://github.com/casehubio/engine/issues/528) | refactor: extract WorkerFunction.Flow to optional module — decouple engine-api from Serverless Workflow types | — | — | — | `WorkerFunction` in `casehub-engine-api` is a sealed interface that hard-codes... |
| [532](https://github.com/casehubio/engine/issues/532) | design: Worker data coordination patterns — DataExchange and DataChannel alongside Blackboard | — | — | — | Surfaced during casehubio/casehub-desiredstate#28 brainstorming. Workers currently have one data... |
| [548](https://github.com/casehubio/engine/issues/548) | feat: support composed GoalExpression — nested anyOf(allOf(...), goal) | — | — | — | `GoalExpression` in the schema (`casehub-engine-schema`) has `List<String> allOf` and... |
| [549](https://github.com/casehubio/engine/issues/549) | feat: expiresAtExpression — absolute deadline for humanTask WorkItems via ExpressionEngine.extractString() | — | — | — | HumanTaskTarget currently supports expiresIn (Duration from creation) and claimDeadlineHours... |
| [564](https://github.com/casehubio/engine/issues/564) | feat: add PlannedAction support to FlowWorkerExecutor / FuncDSL | — | — | — | FlowWorkerExecutor wraps the Map output of a FuncDSL workflow as WorkerResult internally |
| [571](https://github.com/casehubio/engine/issues/571) | feat: enrich CaseLifecycleEvent with case context snapshot | — | — | — | `CaseLifecycleEvent` is a thin record: `(caseId, tenancyId, commandType, eventType, caseStatus,... |
| [577](https://github.com/casehubio/engine/issues/577) | feat: Belbin-aware agent routing — phase-appropriate role selection using eidos vocabulary | — | — | — | Belbin's research shows effective teams need different role archetypes at different project... |
| [578](https://github.com/casehubio/engine/issues/578) | refactor: migrate work-adapter from casehub-work to casehub-work-api (SPI) | — | — | — | casehub-work#275 extracts `WorkItemCreator` + `WorkItemLifecycle` SPIs and `WorkItemEvent`... |
| [579](https://github.com/casehubio/engine/issues/579) | refactor: migrate WorkItemLifecycleAdapter from source() to workItem() | — | — | — | casehub-work #278 replaces `WorkLifecycleEvent.source(): Object` with a typed... |

## Future / Exploratory (15)

| # | Title | Scale | Cmplx | Linked | Notes |
|--:|-------|:-----:|:-----:|--------|-------|
| [5](https://github.com/casehubio/engine/issues/5) | Drools Expression Engine | M | Low | — | Add support for Drools rules. |
| [9](https://github.com/casehubio/engine/issues/9) | YAML schema support for typed expressions | M | Low | — | Add support for engine type in YAML schema. |
| [78](https://github.com/casehubio/engine/issues/78) | Meta-level control knowledge sources — BB1-style stall detection and strategy switching | L | High | child of epic [#77](https://github.com/casehubio/engine/issues/77) | Implement BB1-style meta-level control: control knowledge sources that monitor whether... |
| [79](https://github.com/casehubio/engine/issues/79) | Dual-space blackboard — private agent scratchpad separate from shared workspace | M | High | child of epic [#77](https://github.com/casehubio/engine/issues/77) | Add a private scratchpad zone to the blackboard so agents can work internally before committing... |
| [82](https://github.com/casehubio/engine/issues/82) | Dynamic agent selection — capability and confidence-based activation beyond key existence | M | High | child of epic [#77](https://github.com/casehubio/engine/issues/77) | Extend Binding entry conditions beyond key existence to include capability matching, confidence... |
| [83](https://github.com/casehubio/engine/issues/83) | Reason maintenance — structured justification tracking for Binding-triggered Worker activations | M | High | child of epic [#77](https://github.com/casehubio/engine/issues/77); child of epic [#30](https://github.com/casehubio/engine/issues/30) | Track *why* each Worker was activated — which CaseContext keys satisfied its Binding's... |
| [103](https://github.com/casehubio/engine/issues/103) | feat: Contract Net Protocol — capability bidding and multi-worker task negotiation | L | High | — | The Contract Net Protocol (FIPA, 1994) is a foundational multi-agent coordination mechanism in... |
| [104](https://github.com/casehubio/engine/issues/104) | feat: RAG pipelines with large artefact sharing via Qhorus | L | High | — | Retrieval Augmented Generation (RAG) pipelines chain retrieval, chunking, embedding, reranking,... |
| [105](https://github.com/casehubio/engine/issues/105) | feat: Saga pattern — compensating worker chains for distributed agent transactions | L | High | — | When a multi-step agent pipeline modifies external state — writes to a database, calls a payment... |
| [106](https://github.com/casehubio/engine/issues/106) | feat: Case replay and deterministic re-execution from EventLog | L | High | — | quarkus-ledger provides an immutable, append-only ordered log of every event in a case.... |
| [109](https://github.com/casehubio/engine/issues/109) | feat: code review and generation pipelines with Claudony | L | High | — | A code generation pipeline involves multiple stages with feedback cycles: a specification is... |
| [110](https://github.com/casehubio/engine/issues/110) | feat: LLM goal decomposition — generating explicit typed plans from natural-language goals | L | High | — | Planning frameworks for agentic AI typically decompose a high-level goal into a sequence of... |
| [111](https://github.com/casehubio/engine/issues/111) | feat: Multi-modal agent pipelines — unified orchestration across vision, audio, OCR, and NLP workers | L | High | — | Real-world documents and cases rarely contain a single modality. An insurance claim includes... |
| [113](https://github.com/casehubio/engine/issues/113) | feat: Regulatory decision automation with traceable reasoning and human sign-off gates | L | High | — | Financial services, pharmaceutical, and medical device industries are beginning to deploy... |
| [114](https://github.com/casehubio/engine/issues/114) | feat: ReAct cycles with full auditability — reason-act-observe loops recorded in EventLog | L | High | — | ReAct (Reasoning + Acting) is a fundamental agentic pattern: an LLM reasons about what to do... |

