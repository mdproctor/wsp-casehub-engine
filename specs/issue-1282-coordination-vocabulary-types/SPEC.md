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

**Permission matrix invariant** (enforced by #1283): The permission matrix is a total function from (DecisionScope × DecisionType) → {PERMIT, DENY}. Every (scope, type) pair has a defined outcome. FULL_AUTONOMY permits all DecisionTypes. STIMULUS_DRIVEN permits SIGNALING only as a response within a signal handler (not spontaneous emission). No DecisionType is universally denied across all scopes.

### RoleArchetype (enum)

Structural position of an agent within the topology. Describes how the role relates to others. Small, stable set — engine derives structural relationships from this without hardcoding role names.

| Value | Semantics | Literature source |
|-------|-----------|-------------------|
| `INITIATOR` | Delegates to others — structural authority | FIPA CNP (Manager/Initiator), Moore taxonomy |
| `PEER` | Equal-status coordination — no coordinator relationship | Moore taxonomy, DoF paper (Active Agent) |
| `RESPONDENT` | Receives delegation — structural subordinate | FIPA CNP (Participant/Contractor) |

Named `RESPONDENT` rather than `PARTICIPANT` to avoid collision with `io.casehub.api.model.Participation.PARTICIPANT` (which describes mesh participation level, not structural position). Both enums live under `io.casehub.api.model.*`; the rename eliminates static-import ambiguity and cognitive overload from the same word meaning different things.

Grounded in MAS literature consensus: FIPA Contract Net Protocol, Moore's HMAS taxonomy (arXiv 2508.12683), the Degrees of Freedom framework (arXiv 2610.17751).

### PlatformRole (enum)

Intrinsic behavioral contract — the platform implements specific behavior for these roles. Patterns dispatch against these values. Grows over time as the platform adds new behavioral patterns.

| Value | Platform behavior | Intended consumers |
|-------|-------------------|--------------------|
| `COORDINATOR` | Orchestrates, delegates, allocates tasks, aggregates results | Pattern builders (#1284 TopologyLifecycle), future orchestration SPIs |
| `SUPERVISOR` | Oversees, approves/rejects, gates progression | OversightGateService, ActionRiskClassifier (existing in `api/spi/`) |
| `JUDGE` | Evaluates, scores, renders decisions | Voting/scoring patterns (#1286 VotingProtocol) |
| `MODERATOR` | Enforces protocol, manages turn-taking | Protocol enforcement patterns (#1286 VotingProtocol) |

Only `SUPERVISOR` is wired to existing code today. The other values are forward-looking — downstream specs (#1283–#1288) will define their behavioral SPIs and reference these values.

### AgentRole (record)

An agent's role within a coordination topology. Carries three independent facets — not all are required for every role:

```java
public record AgentRole(
    Path name,                         // always: hierarchical identity + display label
    @Nullable RoleArchetype archetype, // structural position (null for symbolic roles)
    @Nullable PlatformRole platformRole // behavioral contract (null for domain/symbolic roles)
) {
  public AgentRole {
    Objects.requireNonNull(name, "name must not be null");
    if (platformRole != null && archetype == null) {
      throw new IllegalArgumentException(
          "platformRole requires archetype — a behavioral contract implies structural position");
    }
  }
}
```

**Compact constructor invariant:** if `platformRole` is non-null, `archetype` must also be non-null. A platform behavioral contract (e.g. COORDINATOR orchestrates) always implies a structural position (e.g. INITIATOR delegates). The converse is not required — task-oriented roles have archetype without platformRole.

**Factory methods:** `AgentRole.of(String name, ...)` uses `Path.parse(name)` internally, not `Path.of(name)`. This is critical for hierarchy matching — `Path.of("reviewer/code")` creates one segment `"reviewer/code"` (the `/` is just a character), while `Path.parse("reviewer/code")` splits on the delimiter into segments `["reviewer", "code"]`. Only `Path.parse` produces the segment structure that `Path.isAncestorOf()` requires.

**Three kinds of roles:**

1. **Platform-intrinsic** — all three facets present. Platform owns the behavior.
   - `AgentRole.COORDINATOR` = ("coordinator", INITIATOR, COORDINATOR)
   - `AgentRole.SUPERVISOR` = ("supervisor", INITIATOR, SUPERVISOR)
   - `AgentRole.JUDGE` = ("judge", PEER, JUDGE)
   - `AgentRole.MODERATOR` = ("moderator", PEER, MODERATOR)

2. **Task-oriented / domain** — name + archetype, no platform behavior. Applications create these.
   - `AgentRole.of("reviewer/code", RESPONDENT)` — domain role with structural position
   - `AgentRole.of("reviewer/prose", RESPONDENT)` — sibling specialisation

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
       ├── RoleArchetype        (structural position)
       └── PlatformRole         (behavioral contract)

Enforcement group (consumed by AgentCapabilityGuard, #1283):
  DecisionScope              (what agent can observe/act on)
  DecisionType               (what category of action)
```

The structure group describes the coordination arrangement. The enforcement group constrains what agents may do within that arrangement. DecisionScope and DecisionType are per-agent configuration — independent of topology.

Decision authority (LOCAL / COORDINATED / CENTRAL) is derivable from TopologyType + RoleArchetype: CENTRALISED + INITIATOR → CENTRAL, CENTRALISED + RESPONDENT → LOCAL, DISTRIBUTED + PEER → COORDINATED. The AgentCapabilityGuard (#1283) should derive this rather than maintaining a separate enum — this prevents invalid combinations (e.g., CENTRAL authority on a RESPONDENT in a DISTRIBUTED topology).

**TopologyDescriptor** (runtime state: current topology + agent→role assignments) is deferred to #1283+ where the SPIs that construct and consume it are defined. This issue establishes the vocabulary enums and AgentRole record only.

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

## Resolved Questions

1. **`PlatformRole` lives in `model.coordination`.** It is a vocabulary enum consumed by `AgentRole` in this package. The SPI interfaces that implement platform role behavior (#1283–#1288) will depend on this package — the dependency direction is correct. Moving `PlatformRole` to `spi.coordination` would create a circular dependency between the vocabulary records and their enum facets.

2. **`DecisionAuthority` is derivable from `TopologyType + RoleArchetype`.** Removed as a standalone enum. The `AgentCapabilityGuard` (#1283) derives authority from the topology and role assignment, preventing invalid combinations.

3. **`GRADUATED` is a transition policy, not a topology value.** Removed from `TopologyType`. A graduated transition changes the active `TopologyType` value on the descriptor. The transition policy (when/how to graduate) is owned by #1284 SwarmTopologyProvider.
