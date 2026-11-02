# Self-Organising Agent Society — Design Specification

**Issue:** casehubio/eidos#216
**Date:** 2026-11-02
**Decisions:** `2026-11-02-society-moise-fitgap-research-decisions.md`
**Research:** `2026-11-02-society-moise-fitgap-research.md`

## 1. Purpose

Enable CaseHub's agentic architecture to support self-forming collaborative groups
that organise, coordinate, and restructure at runtime. A society of agents where
groups adopt coordination topologies (Swarm/Hive/Mesh/Many), enforce policies
through roles, and restructure through desired-state reconciliation.

## 2. Model

One concept: **Group**. A group can contain agents and/or sub-groups (N-level
recursive). The root group is the Society. Leaf groups are teams. Depth is
unconstrained.

A group has four facets:

| Facet | What it defines | Runtime behaviour |
|---|---|---|
| **Identity** | ID, name, kind (vocabulary-grounded), parent, members | Addressable, queryable, registered in OrgRegistry |
| **Topology** | CoordinationPattern (SWARM/HIVE/MESH/MANY) + AuthorityModel | Mutable at runtime, governed by selfDetermination |
| **Policy** | Role obligations/permissions, selfDetermination, formation rules | Enforced by normative reasoning (deontic dimension) |
| **Roles** | Named roles with adoption/release lifecycle | Agents adopt/release roles dynamically, events emitted |

### 2.1 Topology

Each CoordinationPattern implies a default AuthorityModel but can be overridden:

| Pattern | Default Authority | Characteristics | Runtime Primitive |
|---|---|---|---|
| SWARM | DISTRIBUTED | Decentralised, stigmergy signals, self-organising | Engine stigmergy |
| HIVE | CENTRALISED | Role-assigned, supervised orchestration, scoped memory | Agency-core orchestration patterns |
| MESH | DISTRIBUTED | Message relay, consensus, federation, inter-group | Qhorus channels |
| MANY | — | Isolated parallel, no coordination | None |

Hive is not a separate construct — it is a group configured with roles, policy,
orchestration patterns, and a private Qhorus channel. The topology enum selects
which coordination mechanisms are active, not a different implementation.

Topology transitions (e.g. SWARM → HIVE when a detected cluster stabilises and
adopts roles) are governed by selfDetermination:
- Low (< 0.3): topology changes require external approval
- Medium (0.3–0.7): auto-approve within pattern family, approval across
- High (> 0.7): group self-governs topology changes

### 2.2 Roles

Roles are adopted and released at runtime, not just declared at boot (MOISE+
alignment). Each role carries obligations and permissions defined by the group's
policy:

```yaml
policy:
  roles:
    coordinator:
      obligations: [review-output, approve-plan, report-status]
      permissions: [provision-agent, change-topology, create-subgroup]
    analyst:
      obligations: [process-assigned-work]
      permissions: [signal-consensus, request-collaboration]
```

Role lifecycle events (`onRoleAdopted`, `onRoleReleased` in
AgentWorkLifecycleListener — engine#1285) make role transitions observable.
An agent can hold roles in multiple groups simultaneously.

The coordinator is a policy-designated role — it has specific obligations and
permissions but is not architecturally special. Any agent meeting the role's
requirements can adopt it.

### 2.3 Inter-Group Communication

**Primary: stigmergy.** Groups deposit and observe signals in the shared signal
space, consistent with how individual agents coordinate in the swarm. Groups
discover each other through signal patterns.

**Optional: shared context space.** For simple use cases where groups need ambient
awareness without signal protocol overhead.

**Sustained collaboration: Qhorus channels.** When groups need direct communication:
- Group internal comms → Qhorus PRIVATE channel (qhorus#509)
- Agent-to-agent DM → Qhorus DIRECT channel
- Inter-group collaboration → PRIVATE channel between coordinators
- Society broadcast → PUBLIC channel with BROADCAST semantic

## 3. Formation

Three paths, all producing the same runtime structure. selfDetermination controls
which paths are available:

### 3.1 Top-Down Spawning

A parent group (or operator) creates a sub-group with declared membership, topology,
and policy. This is the path used by eidos static declarations at boot.

### 3.2 Emergence Promotion

Engine's TeamDetector observes agent affinity clusters. When a DetectedTeam's
`stabilityCount` exceeds a configurable threshold, the society runtime promotes
it to a Group:
1. Create Group with members from DetectedTeam
2. Assign initial topology based on detected relations (COACTIVE → SWARM,
   COMPLEMENTARY → HIVE candidate)
3. Inherit policy defaults from parent group
4. Emit formation event

### 3.3 Bottom-Up Proposal

Agents use agency-core's CoalitionProposal to propose a group:
1. Agent creates CoalitionProposal (task + required capabilities + proposed members)
2. CoalitionEvaluator scores capability coverage
3. If viable, proposed members negotiate via NegotiationProtocol
4. On acceptance, society runtime creates the Group
5. Members adopt roles via JointIntention commitment

## 4. Dissolution

Groups dissolve when:
- All members depart (natural dissolution)
- Parent group or operator explicitly dissolves (top-down)
- JointIntention reaches DROPPED state (mission abandoned)
- selfDetermination permits and members vote to dissolve

Dissolution is dependency-ordered: sub-groups dissolve before parent. Members
release all roles (emitting onRoleReleased events). Group channel closes but
retains history.

## 5. Reorganisation via Desired-State

Eidos org diagrams are declarative desired-state goal specifications (D9).
Organisational transitions use `casehub-desiredstate`:

### 5.1 Flow

```
Org Diagram (YAML) → GoalCompiler → Node Graph
                                        ↓
Current Runtime Org → ActualStateAdapter → Actual Graph
                                        ↓
                                   TransitionPlanner
                                        ↓
                                   TransitionPlan
                                   (removals → suspensions → additions)
                                        ↓
                                   PlanApprovalGate
                                   (selfDetermination controls approval)
                                        ↓
                                   Execute (staged phases)
                                        ↓
                                   ReconciliationLoop
                                   (drift detection, fault recovery)
```

### 5.2 Operations

| Operation | Mechanism |
|---|---|
| Initial deployment | No current state → diagram A |
| Restructuring | Diagram A → diagram B |
| Scaling | Diagram A → A' with more agents |
| Recovery | Drift from diagram A → reconcile back |
| Rollback | Diagram B failing → reapply diagram A |
| Emergent restructuring | Runtime edits desired-state graph (selfDetermination permits) → new reconciliation cycle |

### 5.3 Adaptive Reorganisation

SituationRecompiler triggers recompilation of the target org when situations arise
(coordination storm, performance drop, signal consensus on structural change).
The recompiled target is diffed against current state and transitions through the
same PlanApprovalGate → Execute → Reconcile flow.

## 6. Implementation — Evolve Eidos Org

Extend existing eidos-org modules. No new modules.

### 6.1 API Changes (eidos-org-api)

**OrganizationalUnit** — add fields:

```java
public record OrganizationalUnit(
    String unitId,
    String name,
    String kind,                    // existing
    String kindVocabulary,          // existing
    String parentUnitId,            // existing
    List<Membership> members,       // existing
    List<String> capabilities,      // existing
    List<String> goals,             // existing
    List<String> constraints,       // existing
    // NEW
    CoordinationPattern topology,
    AuthorityModel authorityModel,
    GroupPolicy policy,
    double selfDetermination,       // 0.0–1.0
    GroupLifecycleState state       // FORMING, ACTIVE, RESTRUCTURING, DISSOLVING
) {}
```

**GroupPolicy** — new record:

```java
public record GroupPolicy(
    Map<String, RolePolicy> roles,
    FormationRules formationRules,
    double topologyChangeThreshold
) {}

public record RolePolicy(
    List<String> obligations,
    List<String> permissions,
    int minAgents,
    int maxAgents
) {}
```

**GroupLifecycleState** — new enum:

```java
public enum GroupLifecycleState {
    FORMING, ACTIVE, RESTRUCTURING, DISSOLVING, DISSOLVED
}
```

**OrgRegistry** — add lifecycle operations:

```java
// Existing: register, removeUnit, query methods
// NEW
OrganizationalUnit formGroup(GroupFormationRequest request);
void dissolveGroup(String unitId);
void joinGroup(String unitId, String agentId, String role);
void leaveGroup(String unitId, String agentId);
void adoptRole(String unitId, String agentId, String role);
void releaseRole(String unitId, String agentId, String role);
void changeTopology(String unitId, CoordinationPattern pattern, AuthorityModel authority);
```

### 6.2 Runtime Changes (eidos-org-runtime)

**SocietyLifecycleManager** — orchestrates group formation, dissolution, membership
changes. Emits events consumed by engine (role lifecycle, topology changes) and
desired-state (actual state updates).

**DesiredStateOrgAdapter** — implements desired-state's ActualStateAdapter SPI.
Reads current org structure from OrgRegistry, presents it as a node graph for
diffing.

**OrgGoalCompiler** — implements desired-state's GoalCompiler SPI. Compiles an
eidos org diagram (YAML) into a desired-state node graph.

### 6.3 Engine Integration

Engine consumes eidos org events through existing SPI patterns:
- TopologyProvider (engine#1284) queries eidos OrgRegistry for group topology
- AgentWorkLifecycleListener (engine#1285) receives role adopt/release events
- TeamDetector feeds DetectedTeam data to SocietyLifecycleManager for emergence promotion

### 6.4 Qhorus Integration

When a group forms, SocietyLifecycleManager calls Qhorus to create the group's
PRIVATE channel (qhorus#509). Coordinator gets MODERATOR role. Membership
synchronised on join/leave. Channel closed (not deleted) on dissolution.

## 7. YAML Declaration (Eidos Seed)

```yaml
society:
  id: claims-ops
  name: Claims Operations
  selfDetermination: 0.5

  groups:
    - id: auto-claims
      topology:
        pattern: hive
        authority: centralised
      selfDetermination: 0.3
      policy:
        roles:
          coordinator:
            obligations: [review-output, approve-plan]
            permissions: [provision-agent, change-topology]
          triage:
            obligations: [classify-incoming]
            permissions: [signal-consensus]
          assessor:
            obligations: [process-assigned-work]
            permissions: [request-collaboration]
      members:
        - agentId: agent-t1
          role: triage
        - agentId: agent-c1
          role: coordinator

    - id: fraud-watch
      topology:
        pattern: swarm
        authority: distributed
      selfDetermination: 0.7
      policy:
        roles:
          analyst:
            obligations: [monitor-signals]
          investigator:
            obligations: [deep-analysis]
      members:
        - agentId: agent-f1
          role: analyst
```

This YAML is both the eidos seed (loaded at boot by ClasspathYamlOrgRegistrar)
and a desired-state goal specification (compiled by OrgGoalCompiler for
restructuring).

## 8. Cross-Repo Issue Map

| Repo | Issue | Relationship |
|---|---|---|
| **eidos** | #216 | Epic — this spec |
| **engine** | #1282 | CoordinationPattern + AuthorityModel vocabulary |
| **engine** | #1284 | TopologyProvider SPI |
| **engine** | #1285 | Role lifecycle events |
| **engine** | #1287 | Superseded by qhorus#509 |
| **agency** | #377 | Parent epic — distributed per-agent coordination |
| **agency** | #380 | CoordinationTopology SPI + HiveTopology |
| **agency** | #385 | TopologySpec YAML parity |
| **qhorus** | #509 | Channel visibility (PUBLIC/PRIVATE/DIRECT) |
| **desired-state** | — | OrgGoalCompiler + DesiredStateOrgAdapter (new issues TBD) |

## References

- [MOISE+ (Hübner et al. 2002)](https://moise.sourceforge.net/doc/publications/Hubner-sbia2002.pdf)
- [AGR — From Agents to Organizations (Ferber et al. 2003)](https://www.madkit.net/madkit/docs/articles/FromAgentstoOrganizations_AAMAS03_FerbGutMich.pdf)
- [Society Agent (TechRxiv 2026)](https://www.techrxiv.org/doi/full/10.36227/techrxiv.177211798.85464735/v1)
- [Organisational Theory for Multi-Agent Interactions (Springer 2025)](https://link.springer.com/article/10.1007/s10791-025-09667-2)
- [Multi-Agent Oriented Reorganisation within JaCaMo (Sorici et al. 2012)](https://www.researchgate.net/publication/281328187_Multi_Agent_Oriented_Reorganisation_within_the_JaCaMo_infrastructure)
- Fit-gap research: `2026-11-02-society-moise-fitgap-research.md`
- Decisions: `2026-11-02-society-moise-fitgap-research-decisions.md`
