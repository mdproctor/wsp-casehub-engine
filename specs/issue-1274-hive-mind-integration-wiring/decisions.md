## D1: Dependency model for eidos/qhorus integration

**Choice:** Direct injection with optional beans (`Instance<T>`)
**Alternatives:**
- Engine-defined SPI adapters — adds a wrapping layer that mirrors eidos/qhorus APIs with no design benefit
- New integration module — premature for a few injections and method calls
**Rationale:** engine-api already depends on eidos-api and qhorus-api; the types are on the classpath. Pre-release platform — no backward-compat concern. eidos-routing precedent exists only because routing involves complex domain translation; swarm integration is simpler.
**Trade-offs:** Engine gains a runtime dependency on eidos/qhorus API types (already present via engine-api). If eidos/qhorus APIs change, engine code changes too — acceptable for a pre-release platform that bumps versions together.
**Sources:** engine/api/pom.xml (existing eidos-api/qhorus-api deps), eidos-routing module (precedent), AgentGraphQuery.java, RuntimeCollaborationQuery.java, MessageDispatcher.java
**Exploration:** quick
**Status:** captured

## D2: Eidos integration points (#1279)

**Choice:** Wire all three: SwarmProvisioner + coActiveAgents/topAgentsByOutcome, StigmergyCoordinator + coActiveAgents for neighbor awareness, TeamDetector + RuntimeCollaborationQuery.collaborators()
**Alternatives:**
- SwarmProvisioner only — highest-value single point, YAGNI for the others
- SwarmProvisioner + StigmergyCoordinator — skip TeamDetector enrichment
**Rationale:** User wants full integration. All three are independent call sites — no cascading complexity. Pre-release platform, wire it all now.
**Trade-offs:** More injection points to maintain. TeamDetector enrichment may prove low-value in practice — can be removed later if unused.
**Sources:** AgentGraphQuery (coActiveAgents, topAgentsByOutcome), RuntimeCollaborationQuery (collaborators), SwarmProvisioner.java, StigmergyCoordinator.java, TeamDetector.java
**Exploration:** quick
**Depends on:** D1 (dependency model)
**Status:** captured

## D3: Qhorus broadcast strategy (#1280)

**Choice:** Support both CDI domain events and coordination channel posting — don't prescribe one strategy over the other
**Alternatives:**
- CDI events only — engine emits, bridge/router handles delivery later
- Channel posting only — engine posts directly to coordination channel via CaseChannelProvider
- Defer entirely to qhorus — wait for routers/brokers before implementing
**Rationale:** Too early to know which strategy works best for each use case. CDI events suit decoupled "something happened" notifications. Channel posting suits structured protocol-enforced communication where agents need to acknowledge or act. Both are valid; best practices will evolve with use. Qhorus will eventually have routers/brokers for intent-based routing ("I need X" without direct RPC), but they don't exist yet.
**Trade-offs:** Supporting both means slightly more surface area. But choosing one prematurely locks out valid use cases.
**Sources:** MessageDispatcher.broadcast() throws (unimplemented). CaseChannelProvider is the engine's only qhorus access path. Coordination channel exists but is empty today. MessageType.STATUS is agent-visible; EVENT is not. CommitmentState provides protocol-level ack in qhorus; platform notifications provide laissez-faire engagement tracking. See parent/docs/platform/messaging-vs-notifications.md and protocol PP-20261031-060886.
**Exploration:** deep-analysis
**Depends on:** D1 (dependency model)
**Status:** captured

## D4: MessageType for swarm coordination events

**Choice:** Use STATUS for state changes agents should react to (provisioning, departures, pattern detections). Use EVENT for telemetry-only signals on the observe channel (agent-invisible).
**Alternatives:**
- COMMAND for all — creates obligations, too heavy for informational broadcasts
- EVENT for all — agent-invisible, agents can't react to swarm changes
**Rationale:** STATUS is agent-visible and informational — agents observe and adapt voluntarily. Fits the stigmergy model: no enforced obligation, just environmental state change. COMMAND would impose protocol overhead (commitment tracking) on events that don't require a response.
**Trade-offs:** STATUS doesn't create commitments — no way to verify agents actually processed the change. Acceptable for stigmergy where adaptation is voluntary.
**Sources:** MessageType.isAgentVisible() (EVENT=false, STATUS=true). CommitmentState lifecycle only triggered by COMMAND.
**Exploration:** quick
**Depends on:** D3 (broadcast strategy)
**Status:** captured
