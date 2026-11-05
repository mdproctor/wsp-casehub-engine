# Self-Organising Agent Society — MOISE+ Fit-Gap Research

**Date:** 2026-11-02
**Context:** Designing a self-organising society model for CaseHub's agentic architecture.
Society → Group → Team hierarchy (N-level recursive). Groups defined by identity + topology + policy + roles.

## MOISE+ Overview

[MOISE+](https://moise.sourceforge.net/) (Hübner, Sichman, Boissier — 2002) is the
canonical organisational model for multi-agent systems. It views organisation through
three dimensions:

- **Structural** — roles, inheritance links, groups (recursive), cardinality constraints
- **Functional** — social schemes (goal decomposition trees), missions (reusable goal bundles), execution ordering
- **Deontic** — obligations and permissions binding roles to missions; norm conflict resolution

An **Organisational Specification (OS)** is instantiated into an **Organisational Entity (OE)**
at runtime. The OE evolves through events: agent entrance/leaving, group creation/dissolution,
role adoption/release, scheme execution, mission commitment.

## Fit-Gap

### Structural Specification

| MOISE+ Concept | CaseHub Equivalent | Status | Notes |
|---|---|---|---|
| Roles | Eidos `Membership.role` + `@OrgUnit` | **Partial** | Roles exist but no inheritance/specialisation hierarchy between roles |
| Role inheritance | — | **Gap** | No role specialisation (e.g. "attacker inherits player") |
| Groups (recursive) | Eidos `OrganizationalUnit` + `parentUnitId` | **Good** | Hierarchy exists. Aligns with N-level recursive design decision |
| Group cardinality | — | **Gap** | No min/max constraints on group membership (eidos has max 50 hard limit only) |
| Compatibility links | Eidos `AgentRelationship` (SUPERVISES, DELEGATES_TO, ESCALATES_TO, REPORTS_TO, BACKS_UP) | **Good** | Relationship kinds cover acquaintance, authority, communication |
| Structural constraints (correspondence, dependency) | — | **Gap** | No "playing role X requires also playing role Y" constraints |

### Functional Specification

| MOISE+ Concept | CaseHub Equivalent | Status | Notes |
|---|---|---|---|
| Social Scheme (goal tree) | Agency-core HTN decomposition + orchestration patterns | **Good** | HTN + sequence/parallel/conditional patterns map well |
| Missions (goal bundles) | Agency-core `CoalitionProposal` + `JointIntention` | **Partial** | Coalition proposes, joint intention commits — but no formal reusable "mission" as a named, assignable goal bundle |
| Goal types (achieve / maintain) | — | **Gap** | No distinction between one-shot and ongoing goals |
| Execution ordering (sequence, choice, parallel) | Agency-core orchestration patterns | **Good** | Sequence, parallel, conditional, loop all exist |

### Deontic Specification

| MOISE+ Concept | CaseHub Equivalent | Status | Notes |
|---|---|---|---|
| Obligations (role → mission) | Agency-core `NormativeReasoning` | **Good** | Norm obligations exist |
| Permissions (role → mission) | Agency-core `NormativeReasoning` | **Good** | Norm permissions exist |
| Norm conflict resolution | Agency-core resolution strategies (priority, specificity, recency, escalation, most-restrictive) | **Good** | Already implemented |

### Runtime (Organisational Entity)

| MOISE+ Concept | CaseHub Equivalent | Status | Notes |
|---|---|---|---|
| OE instantiation | Eidos static bootstrap + engine swarm | **Partial** | Eidos declares, swarm runs — but not connected. Design decision: eidos is "starting configuration" that seeds a dynamic runtime |
| Agent entrance/leaving | `SwarmProvisioner` (join/depart lifecycle) | **Good** | For swarm agents. **Gap** for org unit membership — currently static |
| Group creation/dissolution | `DetectedTeam` (observation only) | **Gap** | Team detection observes but doesn't manage lifecycle. No runtime group creation/dissolution API |
| Role adoption/release | — | **Gap** | Roles are declared at boot, not adopted dynamically |
| SCH execution monitoring | `SwarmTelemetry` | **Partial** | Tracks per-agent work state, not mission/scheme progress |
| Mission commitment | `JointIntention` lifecycle (FORMED → ACTIVE → FULFILLED/DROPPED) | **Good** | Close match — joint intention IS mission commitment |

### Dynamic Reorganisation

| MOISE+ Concept | CaseHub Equivalent | Status | Notes |
|---|---|---|---|
| Reorganisation protocol | `casehub-desiredstate` | **Strong candidate** | See §Desired-State Mapping below |
| Parallel reorganisation (ParaMoise) | `casehub-desiredstate` CrossDomainCompositionEngine | **Candidate** | Multi-domain composition could handle concurrent org changes |
| Reorganisation governance | `casehub-desiredstate` PlanApprovalGate | **Good** | Maps to selfDetermination — high autonomy auto-approves, low requires supervisor |

## Desired-State as Reorganisation Protocol

`casehub-desiredstate` is a foundation-tier desired-state management runtime.
Its README already lists **agent topology** as a declared use case.
The mapping is direct:

| Desired-State Concept | Org Reorganisation Mapping |
|---|---|
| **GoalCompiler** | Compiles a Society spec (desired org structure) into a node graph. Nodes = groups/roles. Edges = membership, supervision, delegation |
| **ActualStateAdapter** | Reads current runtime org topology from the society runtime |
| **TransitionPlanner** | Diffs target vs. current → plan: dissolve groups, form new ones, move agents, assign roles. Dependency-ordered (children before parents on dissolution) |
| **PlanApprovalGate** | Reorganisation governance. High `selfDetermination` → auto-approve. Low → human/supervisor approval |
| **ReconciliationLoop** | Continuous alignment. Drift detection catches unexpected departures, dissolutions. Fault policy handles re-provisioning or escalation |
| **SituationRecompiler** | Adaptive reorganisation. Performance drop or coordination storm triggers recompilation of target org structure |
| **LifecycleManager** | Staged rollouts — form new groups (phase 1), migrate workload (phase 2), dissolve old groups (phase 3) |
| **FaultPolicy** | CBR-informed response to reorganisation faults (agent won't join, group formation fails) |

## Gap Summary

### Strong coverage (build on what exists)

- Structural hierarchy (eidos OrgUnit)
- Agent relationships (eidos AgentRelationship)
- Normative reasoning — obligations, permissions, conflict resolution (agency-core)
- Goal decomposition and orchestration (agency-core HTN + patterns)
- Joint intention / mission commitment (agency-core)
- Reorganisation runtime infrastructure (desired-state)

### Key gaps to address

1. **Role lifecycle** — roles are declared, not adopted/released at runtime
2. **Role inheritance** — no specialisation hierarchy between roles
3. **Mission as reusable goal bundle** — coalition + joint intention are close but not formalised as a named, assignable concept
4. **Goal types** (achieve vs. maintain) — no distinction
5. **Runtime group lifecycle** — formation, dissolution, membership changes as managed operations (not just observation)
6. **Org ↔ runtime bridge** — eidos spec and engine runtime are disconnected; the dynamic society model (in eidos) needs to be the living structure that the runtime animates
7. **Group topology** — groups need a coordination topology property (Swarm/Mesh/Hive/Many) that is mutable at runtime, governed by selfDetermination
8. **Group policy** — first-class policy objects binding roles to obligations/permissions within a group context
9. **Structural constraints** — role correspondence and dependency rules

## Design Decisions (captured so far)

| # | Decision | Rationale |
|---|---|---|
| D1 | One dynamic model; eidos static is a seed, not a separate mode | Static declaration is just pre-populating the dynamic structure at boot. One implementation, not dual-mode |
| D2 | Topology (Swarm/Mesh/Hive/Many) is a mutable group property | Groups can evolve coordination patterns as work changes. selfDetermination governs shift freedom |
| D3 | Group = identity + topology + policy + roles; coordinator is a policy-designated role | No synthetic group-as-agent abstraction. Policy governs roles, topology governs coordination |
| D4 | N-level recursive hierarchy | One concept: Group. Society is root, team is leaf. Any depth in between |
| D5 | Inter-group communication: stigmergy primary, shared context space optional | Consistent with individual agent model. Shared context available for simple use cases |
| D6 | All three formation paths: bottom-up proposal, emergence promotion, top-down spawning | selfDetermination controls which paths are available |
| D7 | ~~Dynamic society runtime lives in eidos~~ **REVISED D7**: Eidos is model layer (types, storage, SPIs). Society lifecycle orchestration lives in desiredstate/eidos | Eidos cannot depend on desiredstate (circular). GroupLifecycleService impl needs desiredstate-api. Eidos retains SPI interface + @DefaultBean stub. desiredstate/eidos provides the real implementation |
| D8 | Reorganisation protocol via casehub-desiredstate | Desired-state already handles graph-based state reconciliation with approval gates, drift detection, and staged rollouts. Agent topology is a declared use case |
| D9 | Eidos org diagrams ARE desired-state goal specifications | Org diagrams are declarative, idempotent, git-versioned. You never imperatively restructure — you declare the target state and desired-state transitions to it. Initial deployment = no state → diagram A. Restructuring = diagram A → B. Recovery = drift → reconcile back. Emergent formation (DetectedTeam promotion, coalition proposals) = runtime editing the desired-state graph when selfDetermination permits, triggering a new reconciliation cycle |

| D10 | Evolve eidos-org in place | Pre-release, no compat concerns. Extend existing OrganizationalUnit. No new eidos modules |
| D11 | Engine does not call GroupLifecycleService | Engine emits SWARM_TEAM_FORMED events. A listener in desiredstate/eidos handles emergence promotion. Engine stays unaware of org lifecycle |
| D12 | Desiredstate stays a separate repo | Desiredstate is domain-agnostic (GoalCompiler, TransitionPlanner work for any domain). Core jars have no engine deps. The society bridge is one module (desiredstate/eidos) |

## Repo Dependency Order

```
eidos (tier 0 — depends on nothing)
  ↓
qhorus (tier 1 — depends on eidos)
engine (tier 1 — depends on eidos, qhorus)
  ↓
desiredstate (tier 2 — depends on eidos; engine only via engine-adapter)
  ↓
agency (tier 3 — depends on eidos, engine, qhorus)
```

## Module Responsibility Split

| Layer | What | Where |
|---|---|---|
| Types, validation, storage SPIs | Records, enums, OrgRegistry, GroupInstanceStore | eidos-org-api |
| @DefaultBean stubs | UnsupportedGroupLifecycleService, UnsupportedGroupInstanceStore | eidos-org-runtime |
| In-memory storage | InMemoryOrgRegistry, InMemoryGroupInstanceStore | eidos-org-memory |
| Compile + diff + plan | OrgGoalCompiler, DesiredStateOrgAdapter | desiredstate/eidos |
| Reconcile + drift + fault | OrgDriftPolicy, OrgFaultPolicy, OrgNodeProvisioner | desiredstate/eidos |
| GroupLifecycleService impl | Desired-state-aware orchestration | desiredstate/eidos |
| Emergence promotion listener | SWARM_TEAM_FORMED → formGroup() | desiredstate/eidos |
| Swarm detection, event emission | TeamDetector, StigmergyCoordinator | engine |
| TopologyProvider, role events | Query org model for agent coordination | engine |
| Channel visibility | PUBLIC/PRIVATE/DIRECT | qhorus |
| Deliberation (coalition, negotiation) | Bottom-up proposals, normative reasoning | agency |

## Issue Tracker

| Repo | Issue | Status |
|---|---|---|
| eidos | #216 — Society epic (types, storage) | CLOSED |
| eidos | #227 — Move DefaultGroupLifecycleService to desiredstate | CLOSED |
| qhorus | #509 — Channel visibility | CLOSED |
| desiredstate | #211 — Society lifecycle bridge | CLOSED |
| engine | #1282 — CoordinationPattern + AuthorityModel | CLOSED |
| engine | #1284 — TopologyProvider SPI | CLOSED |
| engine | #1285 — AgentWorkLifecycleListener | CLOSED |
| agency | #380 — CoordinationTopology + HiveTopology | CLOSED |
| agency | #385 — TopologySpec YAML | CLOSED |

## Academic References

- [AGR — From Agents to Organizations (Ferber, Gutknecht, Michel 2003)](https://www.madkit.net/madkit/docs/articles/FromAgentstoOrganizations_AAMAS03_FerbGutMich.pdf)
- [MOISE+ — Structural, Functional, and Deontic Model (Hübner et al. 2002)](https://moise.sourceforge.net/doc/publications/Hubner-sbia2002.pdf)
- [MOISE project site](https://moise.sourceforge.net/)
- [Organisational Theory for Multi-Agent Interactions (Springer 2025)](https://link.springer.com/article/10.1007/s10791-025-09667-2)
- [Society Agent — Hierarchical Multi-Agent Architecture (TechRxiv 2026)](https://www.techrxiv.org/doi/full/10.36227/techrxiv.177211798.85464735/v1)
- [Stochastic Self-Organization in Multi-Agent Systems (arXiv 2025)](https://arxiv.org/abs/2510.00685)
- [Multi-Agent Oriented Reorganisation within JaCaMo (Sorici et al. 2012)](https://www.researchgate.net/publication/281328187_Multi-Agent_Oriented_Reorganisation_within_the_JaCaMo_infrastructure)

## Concrete Scenario: Cyber-Insurance Product Launch

Exercises all desired-state capabilities: graph diffing, dependency ordering,
staged rollout, approval gates, drift detection, fault recovery, adaptive recompilation.

### Before-State

```
Society: ClaimsOps
├── Group: AutoClaims (topology: Hive, selfDetermination: 0.3)
│   ├── Triage → [T1, T2]
│   ├── Assessor → [A1, A2, A3]
│   └── Coordinator → [C1]
├── Group: HomeClaims (topology: Hive, selfDetermination: 0.3)
│   ├── Triage → [T3, T4]
│   ├── Assessor → [A4, A5]
│   └── Coordinator → [C2]
└── Group: FraudWatch (topology: Swarm, selfDetermination: 0.7)
    ├── Analyst → [F1, F2, F3]
    └── Investigator → [F4]
```

### Trigger

New cyber-insurance product launches. 400% claim volume spike in 48 hours.
No CyberClaims group exists. AutoClaims volume drops 30%.

### GoalCompiler → Desired State

```
Society: ClaimsOps
├── AutoClaims — SHRINK (release T2, A3)
├── HomeClaims — UNCHANGED
├── CyberClaims — NEW (topology: Swarm → Hive)
│   ├── Triage → [T2, T5-new]
│   ├── Assessor → [A3, A6-new, A7-new]
│   ├── Specialist → [S1-new]      ← cyber-specific role
│   └── Coordinator → [C3-new]
└── FraudWatch — EXPAND (add F5-new for cyber fraud patterns)
```

### TransitionPlanner — Dependency-Ordered Phases

**Phase 1 — Form & Provision** (no dependencies):
Create empty CyberClaims group (starts as Swarm). Provision new agents T5, A6, A7, S1, C3, F5.

**Phase 2 — Migrate** (depends on Phase 1):
Move T2 from AutoClaims.Triage → CyberClaims.Triage.
Move A3 from AutoClaims.Assessor → CyberClaims.Assessor.
Add F5 to FraudWatch.Analyst.

**Phase 3 — Stabilise** (depends on Phase 2):
CyberClaims topology Swarm → Hive (assign coordinator C3, enforce role policy).
Verify all roles filled, cardinality constraints met.

### PlanApprovalGate — selfDetermination in Action

- **AutoClaims** (0.3) → requires supervisor approval to shrink. Plan pauses.
- **FraudWatch** (0.7) → auto-approves expansion. F5 joins immediately.
- **CyberClaims** (0.5, inherited) → auto-approves formation, requires approval for topology shift.

### Complication — Drift Detection

Phase 2 executes. T2 migrates. But A3 is mid-case — actively processing claim #8847.
Migration stalls. ReconciliationLoop detects **drift**: desired says A3 is in CyberClaims,
actual shows A3 still in AutoClaims with open workload.

### Fault Policy — CBR-Informed Recovery

CBR matches: *"agent with active workload — delay migration, provision substitute."*
- Provision A8-new into CyberClaims as temporary assessor
- Mark A3 migration as `PENDING_DRAIN` — executes when claim #8847 completes
- ReconciliationLoop keeps checking each cycle

### SituationRecompiler — Adaptive Re-planning

72 hours in. FraudWatch's swarm detects `SIGNAL_CONSENSUS` — cyber claims have
3x fraud rate. `COORDINATION_STORM` fires.

SituationRecompiler triggers **recompilation of the desired org**:
- FraudWatch spawns sub-group: `CyberFraudCell` (topology: Hive, selfDetermination: 0.8)
- F3, F5 move in with dedicated Investigator role
- Mesh connection to CyberClaims for direct signal relay

New transition plan generated, diffed, fed through PlanApprovalGate.
CyberFraudCell's high selfDetermination means it self-governs from formation —
can recruit, shift topology, dissolve without supervisor approval.

## Issue

**casehubio/eidos#216** — Self-organising agent society — dynamic group lifecycle with MOISE+ alignment

## Topology Taxonomy

From agentic engineering practice — each topology defines how the group thinks:

| Topology | Characteristics | CaseHub Mapping |
|---|---|---|
| **Swarm** | Decentralised, self-organising, shared signals, no central control | Engine stigmergy package — built |
| **Hive** | Role-assigned (planners, builders, critics), scoped memory, reflection, recursive | Agency-core orchestration patterns (supervisor, debate, voting) — partially built |
| **Mesh** | Message relay between peers, consensus, federation, distributed trust | Inter-group connective tissue — **gap** |
| **Many** | Isolated parallel execution, no coordination | Trivial — just agents with no topology |
