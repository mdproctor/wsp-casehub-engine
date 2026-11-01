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

## Package

`io.casehub.api.model.coordination` — new package in the `api` module.

Rationale: Coordination is a superset of stigmergy (which is one communication mode). The existing `model/stigmergy` package holds swarm-specific types (SwarmConfig, SwarmProgress). The coordination vocabulary defines the broader conceptual model that subsumes stigmergy.

## Type Definitions

### TopologyType (enum)

Identifies the coordination topology — how decision loops interact.

| Value | Semantics |
|-------|-----------|
| `CENTRALISED` | Single coordinator makes all decisions |
| `DISTRIBUTED` | Peer-to-peer, no central coordinator |
| `HYBRID` | Central coordinator for global + per-agent for local |
| `HIERARCHICAL` | Tree of coordinators at different scopes |
| `GRADUATED` | Dynamic, transitions between modes at runtime |

Engine uses this to determine provisioning strategy, lifecycle events, and which agents receive which signals.

### DecisionScope (enum)

Defines what an agent is authorised to observe and act on. Used by AgentCapabilityGuard (#1283) for mechanical enforcement.

| Value | Semantics |
|-------|-----------|
| `REACTIVE` | Responds to incoming signals only, cannot initiate |
| `TASK_ADAPTATION` | Can adjust HOW it executes assigned work, cannot change WHAT |
| `FULL_AUTONOMY` | Can decide what to work on, who to collaborate with, when to stop |

### DecisionType (enum)

Categorises agent actions for scope enforcement. The AgentCapabilityGuard checks whether a given DecisionType is permitted for an agent's DecisionScope.

| Value | Semantics |
|-------|-----------|
| `TASK_SELECTION` | Choosing which task to work on |
| `COLLABORATION` | Deciding to collaborate with another agent |
| `SIGNALING` | Emitting signals to influence other agents |
| `PROVISIONING` | Requesting new agents or resources |

### DecisionAuthority (enum)

Per-agent authority level within the topology.

| Value | Semantics |
|-------|-----------|
| `LOCAL` | Decisions affect only this agent |
| `COORDINATED` | Decisions require coordination with peers or coordinator |
| `CENTRAL` | Decisions affect the entire swarm |

### RoleArchetype (enum)

Structural position of an agent within the topology. Describes how the role relates to others. Small, stable set — engine derives structural relationships from this without hardcoding role names.

| Value | Semantics | Literature source |
|-------|-----------|-------------------|
| `INITIATOR` | Delegates to others — structural authority | FIPA CNP (Manager/Initiator), Moore taxonomy |
| `PEER` | Equal-status coordination — no coordinator relationship | Moore taxonomy, DoF paper (Active Agent) |
| `PARTICIPANT` | Receives delegation — structural subordinate | FIPA CNP (Participant/Contractor) |

Grounded in MAS literature consensus: FIPA Contract Net Protocol, Moore's HMAS taxonomy (arXiv 2508.12683), the Degrees of Freedom framework (arXiv 2610.17751).

### PlatformRole (enum)

Intrinsic behavioral contract — the platform implements specific behavior for these roles. Patterns dispatch against these values. Grows over time as the platform adds new behavioral patterns.

| Value | Platform behavior | Where wired |
|-------|-------------------|-------------|
| `COORDINATOR` | Orchestrates, delegates, allocates tasks, aggregates results | OrchestratedDriver, WorkOrchestrator, pattern builders |
| `SUPERVISOR` | Oversees, approves/rejects, gates progression | OversightGateService, ActionRiskClassifier |
| `JUDGE` | Evaluates, scores, renders decisions | Debate SELECTOR, voting patterns |
| `MODERATOR` | Enforces protocol, manages turn-taking | Debate MODERATOR |

### AgentRole (record)

An agent's role within a coordination topology. Carries three independent facets — not all are required for every role:

```java
public record AgentRole(
    Path name,                         // always: hierarchical identity + display label
    @Nullable RoleArchetype archetype, // structural position (null for symbolic roles)
    @Nullable PlatformRole platformRole // behavioral contract (null for domain/symbolic roles)
)
```

**Three kinds of roles:**

1. **Platform-intrinsic** — all three facets present. Platform owns the behavior.
   - `AgentRole.COORDINATOR` = ("coordinator", INITIATOR, COORDINATOR)
   - `AgentRole.SUPERVISOR` = ("supervisor", INITIATOR, SUPERVISOR)
   - `AgentRole.JUDGE` = ("judge", PEER, JUDGE)
   - `AgentRole.MODERATOR` = ("moderator", PEER, MODERATOR)

2. **Task-oriented / domain** — name + archetype, no platform behavior. Applications create these.
   - `AgentRole.of("reviewer/code", PARTICIPANT)` — domain role with structural position
   - `AgentRole.of("reviewer/prose", PARTICIPANT)` — sibling specialisation

3. **Symbolic** — name only. Just a label for rendering/grouping.
   - `AgentRole.of("medic")` — no structural or behavioral binding

**Hierarchy matching** via `Path.isAncestorOf()`:
- `AgentRole.of("reviewer").isKindOf(AgentRole.of("reviewer"))` → true
- `AgentRole.of("reviewer/code").isKindOf(AgentRole.of("reviewer"))` → true
- `AgentRole.of("reviewer/code").isKindOf(AgentRole.of("judge"))` → false

Uses `io.casehub.platform.api.path.Path` — already in engine-api's dependency tree, already the established pattern for hierarchical extensible classification (types and labels on CaseDefinition, WorkItem).

### TopologyDescriptor (record)

Runtime topology state — describes the current coordination topology and which agents play which roles.

```java
public record TopologyDescriptor(
    TopologyType type,
    @Nullable String coordinatorAgentId,
    Map<String, AgentRole> agentRoles
)
```

- `type` — the active topology
- `coordinatorAgentId` — absent in DISTRIBUTED (no coordinator)
- `agentRoles` — maps agent ID → role assignment

## Relationships Between Types

```
TopologyDescriptor
  ├── TopologyType          (which topology)
  ├── coordinatorAgentId    (who coordinates, if anyone)
  └── agentRoles            (who plays what)
       └── AgentRole
            ├── Path name            (identity + display)
            ├── RoleArchetype        (structural position)
            └── PlatformRole         (behavioral contract)

AgentCapabilityGuard (#1283) uses:
  ├── DecisionScope         (what agent can observe/act on)
  ├── DecisionType          (what category of action)
  └── DecisionAuthority     (scope of effect)
```

DecisionScope, DecisionType, and DecisionAuthority are per-agent configuration that the guard enforces. They are independent of topology — an agent's scope doesn't change when the topology changes.

TopologyType, TopologyDescriptor, AgentRole, RoleArchetype, and PlatformRole describe the coordination structure. They change when the topology changes (especially in GRADUATED mode).

## What This Issue Does NOT Include

- **SPIs** — no interfaces or behavioral contracts. Those come in #1283–#1288.
- **YAML surface** — DSL parity for these types comes in agency#385.
- **Convergence** — the convergence model already has its own types in `model/convergence`.
- **Communication mode** — stigmergy/direct/broadcast are a separate dimension, not modelled here.
- **Observation strategy** — event/tick/demand-driven triggers are a separate dimension.
- **Invocation budget** — cost control is agency-owned (#382).

## Open Questions

1. Should `PlatformRole` live in this package or in `io.casehub.api.spi.coordination`? It describes behavioral contracts, which is SPI territory. Counter-argument: it's a vocabulary enum consumed by records in this package; SPIs would depend on it, not the other way around.

2. Should `DecisionAuthority` be on the same agent assignment as `DecisionScope`, or is it derivable from `TopologyType + RoleArchetype`? In CENTRALISED, the coordinator has CENTRAL authority and participants have LOCAL. Is there ever a case where this mapping doesn't hold?

3. Is `GRADUATED` topology a TopologyType value or a meta-concept? If a topology graduates from CENTRALISED to DISTRIBUTED, it changes the TopologyType value. The descriptor updates. Does GRADUATED need to exist as its own value, or is it a transition policy that lives elsewhere?
