# Coordination Vocabulary Types — engine#1282

## Summary

Add shared vocabulary types to `io.casehub.api.model.coordination` in engine-api that define the conceptual model for distributed per-agent coordination. These types are used by engine (mechanical enforcement) and agency (LLM reasoning configuration). They are the foundation types that all other coordination issues (#1283–#1288) build upon.

**Parent epic:** engine#1281 (Swarm Mechanical Enforcement Layer)
**Cross-repo parent:** agency#377 (Distributed Per-Agent LLM Coordination)

## Design Principles

1. **Engine owns vocabulary and enforcement; agency provides intelligence.** These types live in engine-api so both layers share the same definitions.

2. **Closed enums for mechanical enforcement.** Types the engine switches on for enforcement are closed enums — the engine must handle every value.

3. **Open extensibility for application-defined concepts.** Where applications need domain-specific specialisation (agent roles), the types are extensible records with well-known constants.

4. **Platform roles have behavioral meaning.** The platform implements specific behavior for intrinsic roles (coordinator, supervisor, judge, moderator). These are first-class concepts, not just labels.

5. **Coordination overlays the Worker layer, not parallel to it.** The platform already has a Worker concept hierarchy (Worker → FlowWorker, PlaybookWorker, CamelWorker, AgentWorkerFunction). An agent participating in a coordination topology IS an AgentWorker — it ties into the existing Work infrastructure (WorkItem, WorkerFunction, WorkOrchestrator, AgentWorkState). The coordination vocabulary describes *how agents relate to each other structurally*, not *what they are* — the Worker layer already answers "what."

## Package

`io.casehub.api.model.coordination` — new package in the `api` module.

Rationale: Coordination is a superset of stigmergy (which is one communication mode). The existing `model/stigmergy` package holds swarm-specific types (SwarmConfig, SwarmProgress). The coordination vocabulary defines the broader conceptual model that subsumes stigmergy.

**Relationship to `model.stigmergy.CoordinationConfig`:** The existing `CoordinationConfig` record in `model.stigmergy` configures stigmergy-specific tuning parameters (consensus threshold, storm rate multiplier, interest hotspot threshold). It is part of `StigmergyConfig.coordination` and stays in `model.stigmergy` — it configures stigmergy's coordination behavior, not the broader coordination vocabulary. This parallels having `model.signal.Signal` for signal domain objects while signal-related SPIs sit in `spi.observation`.

## Type Definitions

### TopologyType (enum)

Identifies the coordination topology — how decision loops interact.

| Value | Semantics |
|-------|-----------|
| `CENTRALISED` | Single coordinator makes all decisions |
| `DISTRIBUTED` | Peer-to-peer, no central coordinator |
| `HYBRID` | Central coordinator for global decisions + per-agent autonomy for local decisions |

Engine uses this to determine provisioning strategy, lifecycle events, and which agents receive which signals.

**Graduated transitions** are not a topology value — they describe *how the topology changes*, not *what the topology is*. A topology that graduates from CENTRALISED to DISTRIBUTED changes the TopologyType value on the descriptor. The transition policy (when and how to graduate) is a separate concern owned by the topology provider (#1284 SwarmTopologyProvider).

### DecisionScope (enum)

Defines what an agent is authorised to observe and act on. Used by AgentCapabilityGuard (#1283) for mechanical enforcement.

| Value | Semantics |
|-------|-----------|
| `STIMULUS_DRIVEN` | Responds to incoming signals only, cannot initiate |
| `TASK_ADAPTATION` | Can adjust HOW it executes assigned work, cannot change WHAT |
| `FULL_AUTONOMY` | Can decide what to work on, who to collaborate with, when to stop |

**Note:** These types define the vocabulary for enforcement. Enforcement is deferred to #1283 (AgentCapabilityGuard). Until then, these types are informational only — no runtime enforcement prevents an agent from exceeding its declared scope.

### DecisionType (enum)

Categorises agent actions for scope enforcement. The AgentCapabilityGuard checks whether a given DecisionType is permitted for an agent's DecisionScope.

| Value | Semantics |
|-------|-----------|
| `TASK_SELECTION` | Choosing which task to work on |
| `COLLABORATION` | Deciding to collaborate with another agent |
| `SIGNALING` | Emitting signals to influence other agents |
| `PROVISIONING` | Requesting new agents or resources |
| `CONTEXT_MUTATION` | Writing to CaseContext keys that affect other agents' observations |

**Permission matrix invariant** (enforced by #1283): The permission matrix is a total function from (DecisionScope × DecisionType) → {PERMIT, DENY}. Every (scope, type) pair has a defined outcome — a static 2D lookup. The matrix is one layer of a multi-layer enforcement stack. It determines which action *categories* are permissible for a given scope. FULL_AUTONOMY permits all DecisionTypes. No DecisionType is universally denied across all scopes.

**Reference matrix** (definitive mapping for #1283):

| | TASK_SELECTION | COLLABORATION | SIGNALING | PROVISIONING | CONTEXT_MUTATION |
|---|---|---|---|---|---|
| **STIMULUS_DRIVEN** | DENY | DENY | PERMIT | DENY | DENY |
| **TASK_ADAPTATION** | DENY | PERMIT | PERMIT | DENY | PERMIT |
| **FULL_AUTONOMY** | PERMIT | PERMIT | PERMIT | PERMIT | PERMIT |

**Two-layer enforcement for STIMULUS_DRIVEN:** STIMULUS_DRIVEN + SIGNALING is PERMIT in the matrix. The "cannot initiate" constraint in STIMULUS_DRIVEN's semantics is enforced at a separate layer — the *activation guard*: STIMULUS_DRIVEN agents are only activated by incoming signals, so their signaling actions are inherently responsive. The matrix determines *what action categories are permitted* (static, context-free). The activation guard determines *when the agent may execute at all* (requires incoming signal as trigger). Both layers must pass for an action to proceed.

**Relationship to MeshParticipation.REACTIVE:** `MeshParticipation.REACTIVE` (in `spi.mesh`) controls mesh channel registration — "do not register; only engage when directly addressed." STIMULUS_DRIVEN controls decision scope — what categories of action an agent may perform. These are orthogonal: a STIMULUS_DRIVEN agent might be ACTIVE in the mesh (registered, posting status) but restricted to responding to incoming signals for its *decisions*. The activation guard enforcement mechanism is deferred to #1283 (AgentCapabilityGuard).

### StructuralRole (enum, renamed from RoleArchetype)

Structural position of an agent within the topology. Describes how the role relates to others. Small, stable set — engine derives structural relationships from this without hardcoding role names.

| Value | Semantics | Literature source |
|-------|-----------|-------------------|
| `INITIATOR` | Delegates to others — structural authority | FIPA CNP (Manager/Initiator), Moore taxonomy |
| `PEER` | Equal-status coordination — no coordinator relationship | Moore taxonomy, DoF paper (Active Agent) |
| `CONTRACTOR` | Receives delegation — contracted to perform specific work | FIPA CNP (Contractor), Smith 1980 (original Contract Net) |

Named `CONTRACTOR` (the FIPA Contract Net Protocol's original term for the agent that bids on and executes announced tasks) rather than `PARTICIPANT` or `WORKER`:
- **Not PARTICIPANT:** collides with `io.casehub.api.model.Participation.PARTICIPANT` (which describes compound completion semantics, not structural position). Both enums live under `io.casehub.api.model.*`.
- **Not WORKER:** the platform already has Worker as a first-class concept hierarchy (Worker → FlowWorker, PlaybookWorker, CamelWorker, AgentWorkerFunction). An agent in a coordination topology IS an AgentWorker at the Work layer. Adding `StructuralRole.WORKER` would conflate the structural coordination role with the execution identity.
- **CONTRACTOR** sits at the coordination protocol level — the same register as INITIATOR. Both describe the contractual relationship between agents: one initiates the contract, one contracts to perform it.

Grounded in MAS literature: FIPA Contract Net Protocol (Smith 1980, IEEE FIPA00029), Moore's HMAS taxonomy (arXiv 2508.12683), the Degrees of Freedom framework (arXiv 2610.17751).

### PlatformRole (enum)

Intrinsic behavioral contract — the platform implements specific behavior for these roles. Patterns dispatch against these values. Grows over time as the platform adds new behavioral patterns.

| Value | Platform behavior | Required archetype | Intended consumers |
|-------|-------------------|--------------------|--------------------|
| `COORDINATOR` | Orchestrates, delegates, allocates tasks, aggregates results | `INITIATOR` | Pattern builders (#1284 TopologyLifecycle), future orchestration SPIs |
| `SUPERVISOR` | Oversees, approves/rejects, gates progression | `INITIATOR` | OversightGateService, ActionRiskClassifier (existing in `api/spi/`) |
| `JUDGE` | Evaluates, scores, renders decisions | `PEER` | Voting/scoring patterns (#1286 VotingProtocol) |
| `MODERATOR` | Enforces protocol, manages turn-taking | `PEER` | Protocol enforcement patterns (#1286 VotingProtocol) |

Each PlatformRole carries a `requiredStructuralRole()` — the structural position that its behavioral contract implies. COORDINATOR and SUPERVISOR delegate to others (→ INITIATOR). JUDGE and MODERATOR participate as equals in decision processes (→ PEER). This mapping is enforced by AgentRole's compact constructor — incoherent pairings like `COORDINATOR + CONTRACTOR` are rejected at construction time.

Only `SUPERVISOR` is wired to existing code today. The other values are forward-looking — downstream specs (#1283–#1288) will define their behavioral SPIs and reference these values.

**Relationship to conductor infrastructure:** The existing conductor pattern (`ConductorInboxEntry`, `ConductorDecision`, `ConductorInboxManager` in `model.stigmergy` / `runtime.improvement`) implements an approval/rejection workflow with `Status.APPROVED`, `REJECTED`, `REDIRECTED`, `TIMED_OUT`, `AUTO_APPROVED`. This is the same behavioral pattern that `SUPERVISOR` describes at the vocabulary level — oversight and gating. When downstream specs (#1283–#1288) implement SUPERVISOR behavioral contracts, they must decide whether SUPERVISOR builds on, replaces, or coexists with the conductor infrastructure. This reconciliation is deferred to a new GitHub issue filed against #1288 (Agent Behavioral SPI).

### AgentRole (record)

An agent's role within a coordination topology. Carries three independent facets — not all are required for every role:

```java
public record AgentRole(
    Path name,                         // always: hierarchical identity + display label
    @Nullable StructuralRole structuralRole, // structural position (null for symbolic roles)
    @Nullable PlatformRole platformRole // behavioral contract (null for domain/symbolic roles)
) {
  public AgentRole {
    Objects.requireNonNull(name, "name must not be null");
    for (String segment : name.segments()) {
      if (segment.contains("/")) {
        throw new IllegalArgumentException(
            "AgentRole name must be constructed via Path.parse() — "
            + "segment contains unresolved delimiter: \"" + segment + "\"");
      }
    }
    if (platformRole != null && structuralRole == null) {
      throw new IllegalArgumentException(
          "platformRole requires structuralRole — a behavioral contract implies structural position");
    }
    if (platformRole != null && structuralRole != platformRole.requiredStructuralRole()) {
      throw new IllegalArgumentException(
          platformRole + " requires structuralRole " + platformRole.requiredStructuralRole()
          + ", got " + structuralRole);
    }
  }
}
```

**Compact constructor invariants:**

1. **Path segment integrity:** No segment may contain the `/` delimiter. This rejects Paths created via `Path.of("reviewer/code")` (one segment containing `/`) while accepting those from `Path.parse("reviewer/code")` (two segments `["reviewer", "code"]`). The canonical constructor is public by Java record specification — this validation ensures hierarchy matching (`isKindOf`) works correctly regardless of how the Path was created.

2. **PlatformRole requires structuralRole:** if `platformRole` is non-null, `structuralRole` must also be non-null. A platform behavioral contract (e.g. COORDINATOR orchestrates) always implies a structural position (e.g. INITIATOR delegates). The converse is not required — task-oriented roles have structuralRole without platformRole.

3. **PlatformRole → StructuralRole pairing:** if `platformRole` is non-null, `structuralRole` must match the platform role's required structural role. Each PlatformRole has exactly one valid StructuralRole — this is a closed, fixed mapping owned by the PlatformRole enum itself (see below).

**Factory methods:** `AgentRole.of(String name, ...)` uses `Path.parse(name)` internally, not `Path.of(name)`. This is critical for hierarchy matching — `Path.of("reviewer/code")` creates one segment `"reviewer/code"` (the `/` is just a character), while `Path.parse("reviewer/code")` splits on the delimiter into segments `["reviewer", "code"]`. Only `Path.parse` produces the segment structure that `Path.isAncestorOf()` requires.

**Three kinds of roles:**

1. **Platform-intrinsic** — all three facets present. Platform owns the behavior.
   - `AgentRole.COORDINATOR` = ("coordinator", INITIATOR, COORDINATOR)
   - `AgentRole.SUPERVISOR` = ("supervisor", INITIATOR, SUPERVISOR)
   - `AgentRole.JUDGE` = ("judge", PEER, JUDGE)
   - `AgentRole.MODERATOR` = ("moderator", PEER, MODERATOR)

2. **Task-oriented / domain** — name + archetype, no platform behavior. Applications create these.
   - `AgentRole.of("reviewer/code", CONTRACTOR)` — domain role with structural position
   - `AgentRole.of("reviewer/prose", CONTRACTOR)` — sibling specialisation

3. **Symbolic** — name only. Just a label for rendering/grouping.
   - `AgentRole.of("medic")` — no structural or behavioral binding

**Hierarchy matching** via `isKindOf()` — a new method on `AgentRole` (not on `Path`):

```java
public boolean isKindOf(AgentRole other) {
    return this.name.equals(other.name) || other.name.isAncestorOf(this.name);
}
```

`Path.isAncestorOf` is strictly-ancestor (returns `false` for self-comparison). `isKindOf` composes equals + isAncestorOf to give inclusive "is-a" semantics:

- `AgentRole.of("reviewer").isKindOf(AgentRole.of("reviewer"))` → true (equals)
- `AgentRole.of("reviewer/code").isKindOf(AgentRole.of("reviewer"))` → true (ancestor)
- `AgentRole.of("reviewer/code").isKindOf(AgentRole.of("judge"))` → false

Uses `io.casehub.platform.api.path.Path` — already in engine-api's dependency tree, already the established pattern for hierarchical extensible classification (types and labels on CaseDefinition, WorkItem).

**Path semantics for roles:**
- Role names must have at least one non-blank segment — `Path.parse("")` and `Path.parse("  ")` throw `IllegalArgumentException`. There is no "empty role" concept.
- `Path.root()` (empty segments) is not exposed as an AgentRole factory. The `AgentRole.of(String)` factory uses `Path.parse()`, which rejects blank input. If a future use case needs a wildcard role matching all roles, expose `AgentRole.ANY` using `Path.root()` explicitly.
- Path depth carries no semantic constraint — `AgentRole.of("a/b/c")` is equally valid as `AgentRole.of("a")`. Depth reflects domain-specific specialisation granularity, not a ranking.

**Relationship to `DetectedRole` / `DetectedTeam`:** `AgentRole` is a *declared/configured* concept — the topology assigns roles to agents. `DetectedRole` and `DetectedTeam` (in `model.stigmergy`) are *emergent/observed* concepts — the swarm execution model detects behavioral patterns via `RoleTracker` and `TeamDetector`. These are complementary views at different layers: configuration vs. observation. Whether detected patterns can influence declared roles (e.g., auto-promoting an agent whose behavior matches a coordinator pattern) is a behavioral contract for downstream SPIs (#1284 SwarmTopologyProvider), not a vocabulary concern.

## Relationships Between Types

```
Structure group:
  TopologyType               (which topology)
  AgentRole                  (who plays what)
       ├── Path name            (identity + display)
       ├── StructuralRole        (structural position)
       └── PlatformRole         (behavioral contract)

Enforcement group (consumed by AgentCapabilityGuard, #1283):
  DecisionScope              (what agent can observe/act on)
  DecisionType               (what category of action)
```

The structure group describes the coordination arrangement. The enforcement group constrains what agents may do within that arrangement. DecisionScope and DecisionType are per-agent configuration — independent of topology.

Decision authority (LOCAL / COORDINATED / CENTRAL) is derivable from TopologyType + StructuralRole. The AgentCapabilityGuard (#1283) should derive this rather than maintaining a separate enum — this prevents invalid combinations.

**Complete derivation mapping** (total function — every cell defined, including null archetype):

| TopologyType | INITIATOR | PEER | CONTRACTOR | null (symbolic) |
|---|---|---|---|---|
| **CENTRALISED** | CENTRAL | *invalid* | LOCAL | *invalid* |
| **DISTRIBUTED** | *invalid* | COORDINATED | *invalid* | *invalid* |
| **HYBRID** | CENTRAL | COORDINATED | LOCAL | *invalid* |

**Valid combinations (6):**
- CENTRALISED + INITIATOR → CENTRAL: the single coordinator; decisions affect the entire swarm
- CENTRALISED + CONTRACTOR → LOCAL: subordinates follow the coordinator; their own decisions are local
- DISTRIBUTED + PEER → COORDINATED: all agents coordinate as equals; no unilateral authority
- HYBRID + INITIATOR → CENTRAL: coordinator for global decisions
- HYBRID + PEER → COORDINATED: participate in coordination for shared concerns
- HYBRID + CONTRACTOR → LOCAL: subordinates with local-only decision scope

**Invalid combinations (6):** These are structurally incoherent — the topology and role contradict each other:
- CENTRALISED + PEER: CENTRALISED has a coordinator and subordinates; PEER ("equal-status, no coordinator relationship") contradicts the centralised structure
- DISTRIBUTED + INITIATOR: INITIATOR ("delegates to others — structural authority") contradicts DISTRIBUTED ("peer-to-peer, no central coordinator")
- DISTRIBUTED + CONTRACTOR: CONTRACTOR ("receives delegation — structural subordinate") contradicts DISTRIBUTED (no delegation hierarchy)
- Any topology + null archetype: Symbolic roles (null archetype) are labels for rendering/grouping — they have no structural position and cannot participate in decision authority derivation. An agent with a symbolic role appearing in a topology's agent-role assignments is a configuration error.

The derivation function in #1283 should throw `IllegalArgumentException` for invalid combinations. These are configuration errors — an agent cannot be a PEER in a CENTRALISED topology, because CENTRALISED by definition has a coordinator/subordinate structure. Similarly, a symbolic role cannot appear in a topology context that requires authority derivation.

**TopologyDescriptor** (runtime state: current topology + agent→role assignments) is deferred to #1283+ where the SPIs that construct and consume it are defined. This issue establishes the vocabulary enums and AgentRole record only.

**DecisionScope independence from topology:** DecisionScope and DecisionType are per-agent configuration — intentionally independent of topology. This means any combination of (TopologyType × StructuralRole × DecisionScope) is valid. Seemingly contradictory combinations are intentional:

- CENTRALISED + CONTRACTOR + FULL_AUTONOMY: The agent is structurally subordinate (receives delegation from the coordinator) but has full autonomy in *how* it executes — it can decide sub-tasks, collaborations, and timing. The coordinator decides *what* is delegated; the agent decides *how* to do it.
- DISTRIBUTED + PEER + STIMULUS_DRIVEN: A specialist peer that contributes only when prompted. In a peer network, this is a service-like agent that provides expertise on demand without proactively initiating work.

TopologyType × StructuralRole constrains *structural position* (who delegates to whom). DecisionScope constrains *operational freedom* (what categories of action the agent may perform). These dimensions are orthogonal by design.

## What This Issue Does NOT Include

- **SPIs** — no interfaces or behavioral contracts. Those come in #1283–#1288.
- **TopologyDescriptor** — runtime state (current topology + agent→role map) is deferred to #1283+ where SPIs consume it. Agent identity type (String vs typed wrapper) to be resolved then.
- **DetectedRole bridging** — the relationship between emergent `DetectedRole` (stigmergy) and declared `AgentRole` (coordination) is a behavioral concern for the SPI layer, not vocabulary. See §AgentRole for the conceptual distinction.
- **YAML surface** — DSL parity for these types comes in agency#385.
- **Convergence** — the convergence model already has its own types in `model/convergence`.
- **Communication mode** — stigmergy/direct/broadcast are a separate dimension, not modelled here.
- **Observation strategy** — event/tick/demand-driven triggers are a separate dimension.
- **Invocation budget** — cost control is agency-owned (#382).
- **Task coordination primitives** — `io.casehub.platform.compiler.CoordinationSpec` defines workflow-level coordination primitives (barriers, semaphores, signals, channels, quorums, state machines, sagas) for Serverless Workflow task synchronization. That is a different layer: task coordination within a workflow, not agent coordination within a topology.

**Deferral decomposition — #1283 scope partitioning:** Three independent dimension reviews pushed concerns to #1283 (AgentCapabilityGuard), accumulating scope beyond what one issue should address. The following partitioning clarifies ownership:

| Concern | Owner |
|---------|-------|
| AgentCapabilityGuard implementation (matrix enforcement + activation guard) | #1283 |
| DecisionAuthority derivation function | #1283 |
| Permission matrix enforcement | #1283 |
| Invalid TopologyType × StructuralRole rejection | #1283 |
| Agent identity type (String vs typed wrapper) | #1283 |
| TopologyDescriptor definition (type, constraints, construction) | #1284 |
| TopologyDescriptor robustness (invalid states, mutable Map, version/epoch) | #1284 |
| TopologyDescriptor coordinatorAgentId, agentRoles key type | #1284 |
| Topology transition policy (graduation, lifecycle) | #1284 |
| Conductor ↔ SUPERVISOR reconciliation | New issue (against #1288) |

#1283 should be further decomposed before implementation if needed — but the vocabulary spec's role is to clarify which deferred concerns belong where, not to dictate the decomposition of downstream issues.

## Resolved Questions

1. **`PlatformRole` lives in `model.coordination`.** It is a vocabulary enum consumed by `AgentRole` in this package. The SPI interfaces that implement platform role behavior (#1283–#1288) will depend on this package — the dependency direction is correct. Moving `PlatformRole` to `spi.coordination` would create a circular dependency between the vocabulary records and their enum facets.

2. **`DecisionAuthority` is derivable from `TopologyType + StructuralRole`.** Removed as a standalone enum. The `AgentCapabilityGuard` (#1283) derives authority from the topology and role assignment, preventing invalid combinations.

3. **`GRADUATED` is a transition policy, not a topology value.** Removed from `TopologyType`. A graduated transition changes the active `TopologyType` value on the descriptor. The transition policy (when/how to graduate) is owned by #1284 SwarmTopologyProvider.
