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

The root Society group is either declared in the YAML seed (§7, the `society:`
root element) or auto-created with default policy on first group formation if
no seed exists. Every group must have a parent except the root.

**Spec vs runtime (D9):** `OrganizationalUnit` is the immutable declarative
specification — the desired state. `GroupInstance` is the mutable runtime
entity — the actual state. The org diagram declares what SHOULD exist;
GroupInstance records what DOES exist. This mirrors MOISE+'s OS/OE distinction.

A group's specification has four facets:

| Facet | What it defines | Spec type |
|---|---|---|
| **Identity** | ID, name, kind (vocabulary-grounded), parent, members | `OrganizationalUnit` fields |
| **Topology** | Coordination topology (open-string vocabulary, D20) + AuthorityModel | `topology` / `topologyVocabulary` |
| **Policy** | Role obligations/permissions, selfDetermination, formation rules | `GroupPolicy` |
| **Roles** | Named roles with cardinality and capability expectations | `RolePolicy` |

Runtime behaviour is tracked on `GroupInstance`:

| Runtime concern | Tracked on | Examples |
|---|---|---|
| Lifecycle state | `GroupInstance.state` | PROPOSED, FORMING, ACTIVE, REORGANISING, DISSOLVING, DISSOLVED |
| Current topology | `GroupInstance.effectiveTopology` | May differ from spec during transitions |
| Role adoptions | `GroupInstance.roleAdoptions` | Which agents currently hold which roles |

### 2.1 Topology

Topology is an open-string vocabulary term (D20), consistent with eidos's
`slot`/`slotVocabulary` pattern on `AgentDescriptor`. Each well-known topology
value implies a default AuthorityModel but can be overridden:

| Value | Default Authority | Characteristics | Runtime Primitive |
|---|---|---|---|
| `swarm` | DISTRIBUTED | Decentralised, stigmergy signals, self-organising | Engine stigmergy — group-scoped signal filtering |
| `hive` | CENTRALISED | Role-assigned, supervised orchestration, scoped memory | Agency-core orchestration — coordinator routes work items, members report via group channel |
| `mesh` | DISTRIBUTED | Message relay, consensus, federation, inter-group | Qhorus channels — all-to-all messaging within the group, coordinator-to-coordinator across groups |
| `many` | — | Isolated parallel, no coordination | No runtime primitives activated — members work independently |

Domain applications can define custom topology values via vocabulary registrars.
The engine selects runtime behaviour based on topology value: `swarm` activates
group-scoped stigmergy signal filtering, `hive` activates coordinator-routed
work dispatch, `mesh` activates Qhorus peer messaging. Unknown topology values
receive default behaviour (group identity and membership only, no coordination
primitives). The activation mechanism is the proposed TopologyProvider SPI
(engine#1284).

`many` is an organizational abstraction, not a coordination one. It differs from
ungrouped agents in three ways: (1) policy application — agents share group
policy (formation rules, role constraints) without coordination overhead;
(2) organizational identity — the group is queryable via OrgRegistry and
participates in desired-state reconciliation; (3) topology transition — the
group can later transition to `swarm`/`hive`/`mesh` as coordination needs emerge.

Hive is not a separate construct — it is a group configured with roles, policy,
orchestration patterns, and a private Qhorus channel. The topology value selects
which coordination mechanisms are active, not a different implementation.

Topology transitions (e.g. `swarm` → `hive` when a detected cluster stabilises
and adopts roles) are governed by selfDetermination:
- Low (< 0.3): topology changes require external approval
- Medium (0.3–0.7): auto-approve within topology family, approval across
- High (> 0.7): group self-governs topology changes

**Precedence with IntegrationPolicy.selfDetermination:** Engine's
`IntegrationPolicy.selfDetermination` (per-case stigmergy config) governs
individual agent autonomy in swarm coordination. `OrganizationalUnit.selfDetermination`
governs group-level structural decisions — topology changes, formation path
availability, dissolution voting. When an agent is in a group, the group's
selfDetermination is authoritative for structural decisions. IntegrationPolicy's
value remains the default for ungrouped agents and for non-structural stigmergy
behaviour (signal weighting, bootstrap richness). TopologyProvider (engine#1284)
resolves the effective value based on group membership.

**Topology transitions** use the REORGANISING lifecycle state (on GroupInstance):
1. GroupInstance enters REORGANISING — blocks concurrent formation, dissolution,
   and further topology changes for this group
2. Execute transition steps (update coordination pattern, activate/deactivate
   coordination mechanisms, adjust role requirements, sync channels)
3. On success → ACTIVE with new topology
4. On failure → rollback to previous topology, enter ACTIVE, emit fault event

If the system crashes mid-transition, ReconciliationLoop detects drift (partial
topology state) and re-drives the transition from the desired-state graph.

### 2.1.1 selfDetermination Semantics

selfDetermination is an independent property per group — it is neither inherited
from nor capped by a parent group's value.

- **Scope:** governs the group's own structural decisions (topology changes,
  role additions, membership changes, sub-group creation)
- **Default:** 0.0 if unspecified (all changes require external approval —
  safe default for declarative-only groups)
- **Parent independence:** a parent's selfDetermination governs parent-level
  restructuring (adding/removing child groups). A child's selfDetermination
  governs the child's internal restructuring. A highly autonomous child (0.7)
  under a controlled parent (0.3) can restructure itself internally, but the
  parent cannot restructure without approval
- **Validation:** [0.0, 1.0] range enforced at record construction

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
An agent can hold roles in multiple groups simultaneously. When an agent holds
roles in multiple groups, normative reasoning (agency-core's conflict resolution
strategies) operates on the union of obligations across all groups. Cross-group
conflicts are resolved using the same strategies (priority, specificity, recency,
escalation, most-restrictive) — the group's policy priority determines which
group's obligations take precedence when capacity conflicts arise.

The coordinator is a policy-designated role — it has specific obligations and
permissions but is not architecturally special. Any agent meeting the role's
requirements can adopt it.

**Normative framework integration:** Obligation and permission strings in role
policy are norm identifiers — references into agency-core's normative reasoning
framework (`io.casehub.agency.deliberation.normative`). At enforcement time, the
society runtime wraps each obligation/permission in a `NormDecision<String>` with:
- `source`: `"{groupId}:{roleName}"` (identifies the originating group and role)
- `priority`: from `GroupPolicy.normativePriority` (lower wins in
  `PriorityResolution` — sorted ascending, first wins)
- `specificity`: derived from the group's context via defined derivation rules
  (see below)
- `establishedAt`: the role adoption timestamp (`RoleAdoption.adoptedAt`)

**NormSpecificity derivation:** `NormSpecificity` has five levels (UNIVERSAL,
DOMAIN, TENANT, CASE_TYPE, INSTANCE). A group's specificity is derived as:

| Condition | Specificity | Rationale |
|---|---|---|
| `GroupPolicy.normSpecificity` is set | explicit value | operator override |
| Group's `kindVocabulary` matches a case type vocabulary | CASE_TYPE | group is scoped to a specific case type |
| Default | TENANT | groups are tenancy-scoped |

`GroupPolicy` gains an optional `NormSpecificity normSpecificity` field
(default: null, meaning auto-derive). This allows operators to explicitly set
DOMAIN or UNIVERSAL for cross-tenant policy groups. The derivation ensures
that a fraud-claims team (CASE_TYPE) naturally overrides a tenant-wide
compliance group (TENANT) for fraud-related obligations, without requiring
explicit normativePriority tuning.

When obligations from different groups conflict for the same agent,
`ConflictResolutionStrategy<String>` resolves them using the standard resolution
strategies (priority, specificity, recency, escalation, most-restrictive).

**Priority derivation:** `GroupPolicy.normativePriority` is an explicit int field
(default: hierarchy depth — root = 0, leaf = N). Lower values win in
`PriorityResolution`. The default derivation means parent-level obligations
take precedence over child-level obligations: a society-wide norm like
"comply-with-regulations" overrides a team-level "process-fast" when they
conflict for the same agent's capacity. Operators can override by setting
`normativePriority` explicitly in the YAML.

If two groups at the same NormSpecificity level have the same normativePriority,
resolution falls through to the next strategy in the chain (recency, then
escalation). A diagnostic warning is logged when this happens, since equal-
priority ties indicate the operator should assign explicit priorities.

### 2.3 Inter-Group Communication

**Primary: stigmergy.** Groups deposit and observe signals in the shared signal
space, consistent with how individual agents coordinate in the swarm. Groups
discover each other through signal patterns.

**Sustained collaboration: Qhorus channels.** When groups need direct communication:
- Group internal comms → Qhorus PRIVATE channel (qhorus#509)
- Agent-to-agent DM → Qhorus DIRECT channel
- Inter-group collaboration → PRIVATE channel between coordinators
- Society broadcast → PUBLIC channel with BROADCAST semantic

**Relationship to normative 3-channel layout:** Group channels are orthogonal to
the case-level work/observe/oversight layout defined in `CaseChannelLayout`.
Case channels serve task execution within a case context; group channels serve
organisational communication across cases. An agent participates in both: it uses
case channels for work dispatch and group channels for coordination with its
organisational peers. The PRIVATE/DIRECT/PUBLIC/BROADCAST types from qhorus#509
are channel visibility scopes — they do not replace the case-level
`ChannelSemantic` (APPEND) or the normative 3-channel purpose structure.
Group channels use `ChannelSemantic.APPEND` and permit all speech act types.

## 3. Formation

Three paths, all producing the same runtime structure. selfDetermination controls
which paths are available:

**Formation serialization:** All formation paths funnel through
SocietyLifecycleManager, which serializes formation requests per-society.

SocietyLifecycleManager maintains a per-agent membership version counter
(monotonic long, incremented on every membership change — join, leave,
group dissolution). When a formation request is initiated, it captures the
current version for each proposed agent. At commit time, `formGroup()` acquires
a per-society lock, validates that all captured versions are still current
(no agent's membership has changed since the request was created), performs
the membership update (incrementing each agent's version), and releases the
lock. If any version has changed, the entire formation fails atomically with
a membership-conflict event.

The per-society lock ensures atomicity across all proposed members within a
single formation request — no concurrent formation can modify membership while
the lock is held. This prevents the TOCTOU race where per-agent checks could
pass individually but see stale state collectively.

### 3.1 Top-Down Spawning

A parent group (or operator) creates a sub-group with declared membership, topology,
and policy. This is the path used by eidos static declarations at boot.

### 3.2 Emergence Promotion

Engine's TeamDetector observes agent affinity clusters. When a DetectedTeam's
`stabilityCount` exceeds a configurable threshold, the society runtime promotes
it to a Group:
1. Create Group with members from DetectedTeam
2. Assign initial topology from `DetectedTeam.dominantRelations` (a
   `Set<NeighborRelation>`). When multiple relations are present, the
   highest-priority relation wins:

   | Relation (priority order) | Default Topology | Rationale |
   |---|---|---|
   | COMPLEMENTARY (highest) | HIVE | Different skills combining — needs role assignment |
   | SHARED_INTEREST | MESH | Common topic, independent approaches — consensus |
   | COACTIVE | SWARM | Parallel peers doing similar work — stigmergy |
   | SHARED_SIGNAL (lowest) | SWARM | Responding to same signals — stigmergy fit |

   The topology is always a default — the group can transition later via
   selfDetermination governance.
3. Inherit policy defaults from parent group
4. Emit formation event

**Scope bridging — case to org:** `TeamDetector.detect(UUID caseId, SwarmConfig
config)` operates per-case: `CaseTeamState` is keyed by caseId, and
`DetectedTeam` represents agents collaborating within a single case execution.
Promotion intentionally crosses from case scope to org scope — that is the
purpose of emergence. The bridge works as follows:

- **tenancyId derivation:** The case execution context carries a tenancyId
  (every case runs under a tenancy). SocietyLifecycleManager obtains the
  tenancyId from the case context when processing a promotion, and the
  promoted group is created under that tenancy's society.
- **Cross-case applicability:** A promoted group is org-scoped. Once agents A
  and B are promoted from case X into group G, group G applies when those
  agents work on any case under the same tenancy — not just case X. The
  group's coordination topology and policy govern their collaboration
  regardless of which case they are executing.
- **Global re-detection suppression:** After promotion, suppression is org-wide,
  not per-case. `TeamDetector` checks group membership via
  `OrgRegistry.unitsFor(agentId, tenancyId)` before including an agent in
  detection. If agents A and B are already in a promoted group, detecting
  them working together in case Y does NOT trigger a new promotion. This
  prevents duplicate group creation across cases. When the promoted group
  dissolves, suppression lifts and those agents become eligible for detection
  in any case.

### 3.3 Bottom-Up Proposal

Agents use agency-core's CoalitionProposal to propose a group:
1. Agent creates CoalitionProposal (task + required capabilities + proposed members)
2. CoalitionEvaluator scores capability coverage via `CoalitionContext`, which
   provides capability resolution. `CoalitionProposal.coveredCapabilities()` is
   removed — it currently returns member names as a proxy and creates a false
   cross-module dependency. Instead, `CoalitionContext` carries a
   `CapabilityResolver` that maps `AgentRef` names to `AgentCapability` records
   from eidos `AgentRegistry`. The evaluator implementation (which lives in a
   module that depends on both agency-core and eidos-api) injects the resolver.
   This keeps `CoalitionProposal` (agency-core) decoupled from eidos-api.
3. If viable, SocietyLifecycleManager creates a temporary Qhorus negotiation
   channel and attaches `NegotiationProtocol` (a `ChannelProtocol` implementation
   that evaluates dispatch advisories asynchronously via channel message
   projection). Proposed members exchange messages on this channel; the protocol
   evaluates consensus state after each message. SocietyLifecycleManager observes
   the channel for the negotiation outcome.
4. On acceptance (negotiation protocol reaches consensus), society runtime creates
   the Group
5. Members adopt roles via JointIntention commitment (FORMED → ACTIVE)

**Failure modes:**
- **Negotiation timeout:** NegotiationProtocol has a configurable deadline.
  If consensus is not reached, the proposal fails and a formation-failed event
  is emitted. No group is created.
- **Partial acceptance:** Formation requires all proposed members to accept
  (quorum = 100% by default, configurable in FormationRules). If fewer accept,
  the proposal fails — CoalitionEvaluator's capability score was computed for
  the full membership and may be invalid for a subset.
- **Member claimed during negotiation:** Between proposal acceptance and group
  creation, a proposed member may join another group via emergence promotion or
  a concurrent proposal. The formation serialization mechanism (CAS check on
  membership) catches this — the creation step fails for the conflicting
  member, failing the entire formation.
- **selfDetermination gating:** Bottom-up proposal requires selfDetermination
  ≥ 0.5 on the parent group. Below 0.5, agents cannot initiate formation —
  only top-down spawning and emergence promotion (with approval) are available.

## 4. Dissolution

Groups dissolve when:
- All members depart (natural dissolution)
- Parent group or operator explicitly dissolves (top-down)
- JointIntention reaches DROPPED state (mission abandoned)
- selfDetermination permits and members vote to dissolve

**Work-aware dissolution:** Before entering DISSOLVING state, SocietyLifecycleManager
checks for in-flight work (active case assignments, pending tasks). If work is
in-flight:
- **Desired-state dissolution** (restructuring): the agent's work item is
  marked PENDING_DRAIN in the transition plan. Dissolution proceeds for idle
  members; busy members are drained and migrated as work completes. The
  ReconciliationLoop tracks convergence. PENDING_DRAIN has a configurable
  timeout (default: 30 minutes, configurable via `GroupPolicy`). After timeout,
  `SocietyLifecycleManager` handles the stuck item directly — this is a
  lifecycle-level concern, not a desired-state graph concern.
  `SocietyLifecycleManager` can: force-reassign work to the parent group's
  coordinator (imperative action via engine work dispatch), escalate to a
  human review node (desired-state mutation via `addReviewNode`), or
  force-dissolve with orphaned work items logged as fault events. The
  specific action is CBR-informed. `OrgFaultPolicy` handles graph-level
  faults (group failed to provision, agent migration failed) where the
  remediation is expressible as `List<GraphMutation<DesiredNode>>`.
  PENDING_DRAIN timeout is distinct: it requires imperative runtime actions
  (work reassignment, lifecycle state change) that only
  `SocietyLifecycleManager` can perform.
- **Natural dissolution** (all members depart): the group enters DISSOLVING
  immediately — departing agents already handled their own work lifecycle.
- **Top-down dissolution**: the dissolving parent's desired-state plan handles
  work reassignment to sibling or replacement groups before dissolution.
- **Vote/JointIntention dissolution**: enters DISSOLVING, work items are
  orphaned to the parent group's coordinator for reassignment.

Dissolution is dependency-ordered: sub-groups dissolve before parent. Members
release all roles (emitting onRoleReleased events). Group channel closes but
retains history.

## 5. Reorganisation via Desired-State

Eidos org diagrams are declarative desired-state goal specifications (D9).
Organisational transitions use `casehub-desiredstate`:

### 5.1 Flow

```
Org Diagram (YAML) → OrgGoalCompiler → Node Graph
                                           ↓
Current Runtime Org → DesiredStateOrgAdapter → Actual Graph
                                           ↓
                                      TransitionPlanner
                                           ↓
                                      TransitionPlan
                                      (removals → suspensions → additions)
                                           ↓
                                      PlanApprovalPolicy.evaluate()
                                      (selfDetermination controls approval decision)
                                           ↓ (if REQUIRES_APPROVAL)
                                      PlanApprovalHandler.submit() → check() loop
                                           ↓
                                      Execute (staged phases)
                                           ↓
                                      ReconciliationLoop
                                      (drift detection, fault recovery)
```

**Plan approval mapping:** `PlanApprovalPolicy` (desiredstate SPI) evaluates
whether a transition plan needs approval. The org-specific implementation
(`OrgPlanApprovalPolicy`) delegates to the group's **current** selfDetermination
value (the value on the existing `OrganizationalUnit`, not the proposed value
in the transition plan):
- selfDetermination ≥ 0.7 → `PlanApprovalDecision.AUTO_APPROVED`
- selfDetermination 0.3–0.7 → topology-change-only plans auto-approved;
  membership-changing plans require approval
- selfDetermination < 0.3 → all plans require approval

**Plan-level aggregation:** `PlanApprovalPolicy.evaluate(TransitionPlan plan,
String tenancyId, PlanContext context)` receives the entire plan, which may
affect multiple groups. `OrgPlanApprovalPolicy` evaluates per-group (each
affected group's current selfDetermination) and returns the **most
restrictive** result. If any affected group requires approval, the entire plan
requires approval — plans are atomic and cannot be partially approved.

**Circularity prevention:** Using the **current** (pre-transition) value
prevents self-referential evaluation. If a plan changes Group A's
selfDetermination from 0.8 to 0.2, the evaluation uses 0.8 (auto-approve).
The plan executes, and future plans affecting A will see 0.2 (require
approval). If the same plan also changes Group B from 0.2 to 0.8, B's
current 0.2 triggers approval for the entire plan. This is the correct
conservative behaviour — the most restricted group gates the transition.

When approval is required, `PlanApprovalHandler` manages the lifecycle: submit
the plan for review, poll via `check()`, cancel if superseded.

### 5.2 Operations

| Operation | Mechanism |
|---|---|
| Initial deployment | No current state → diagram A |
| Restructuring | Diagram A → diagram B |
| Scaling | Diagram A → A' with more agents |
| Recovery | Drift from diagram A → reconcile back |
| Rollback | Diagram B failing → reapply diagram A |
| Emergent restructuring | Runtime edits desired-state graph (selfDetermination permits) → new reconciliation cycle |

### 5.3 Org-Specific Desired-State Types

**NodeType:** `org:unit` (existing `OrgNodeTypes.UNIT`) — groups are org units
and use the existing node type. Agent membership, roles, and topology are
properties of the group node's `OrgUnitNodeSpec`, not separate nodes. This
keeps the graph shallow and avoids per-agent node overhead.

The existing `org:relationship` node type (`OrgNodeTypes.RELATIONSHIP`) is
preserved — `AgentRelationship` records (supervision, delegation, escalation,
backup) are orthogonal to group structure and continue to be compiled as
separate nodes by `OrgGoalCompiler`. The extended compiler adds new fields
(topology, policy, selfDetermination) to `OrgUnitNodeSpec` but does not
change relationship node handling.

**Provisioning** (`OrgNodeProvisioner implements NodeProvisioner`):
- `provision()` → register OrganizationalUnit (spec) + create GroupInstance (FORMING), create Qhorus
  PRIVATE channel, assign initial roles, add members, transition to ACTIVE.
  Returns `ProvisionResult.Success` when ACTIVE, `ProvisionResult.Failed` if
  any step fails irreversibly (e.g. no agents meet role requirements).
  `ProvisionResult.PendingApproval` when selfDetermination < 0.3 and the
  formation requires external approval.
- `deprovision()` → dissolve group (work-aware, see §4), close channel.

**Drift detection** (`OrgDriftPolicy implements DriftPolicy`):
- Agent departure from a group (membership count drops below minimum)
- Role released with no replacement (obligation gap)
- Autonomous topology change (selfDetermination-permitted, but diverges from
  desired-state target)
- Group dissolved by natural causes (all members left)

Drift decisions: RECONCILE (re-provision missing agent or role), IGNORE
(tolerate within policy bounds), ESCALATE (human review via addReviewNode).

**Fault recovery** (`OrgFaultPolicy implements FaultPolicy`):
- Group formation failure → re-attempt with substitute agents (CBR-informed),
  or escalate to human review node
- Agent migration failure (mid-work) → mark PENDING_DRAIN, provision substitute
- Channel creation failure → retry on next reconciliation cycle

### 5.4 Adaptive Reorganisation

SituationRecompiler triggers recompilation of the target org when situations arise
(coordination storm, performance drop, signal consensus on structural change).
The recompiled target is diffed against current state and transitions through the
same PlanApprovalPolicy → PlanApprovalHandler → Execute → Reconcile flow.

**Stabilization window:** After a recompilation is applied,
`SocietyLifecycleManager` enforces a per-society stabilization period
(default: 5 minutes, configurable). This prevents cascade storms where a
restructure triggers new situations that trigger more restructuring. The
window is per-society, not per-group — a society-wide restructure needs
global stabilization. Situations that fire during the window are queued by
`SocietyLifecycleManager` and evaluated after stabilization completes.

**Ownership:** `SocietyLifecycleManager` owns the stabilization state (last
recompilation timestamp, queued situations) as part of its per-society
in-memory state — alongside the per-society lock and membership version
counters. The `SituationRecompiler` SPI remains stateless. When a situation
triggers, `SocietyLifecycleManager` checks the window before delegating to
the recompiler. On restart, stabilization state is lost (in-memory), which
is safe: the desired-state graph prevents re-applying already-applied
recompilations (the diff would be empty), so an early-firing recompilation
after restart produces a no-op transition plan.

## 6. Implementation — Evolve Eidos Org

Extend existing eidos-org modules (no new eidos modules). Cross-repo changes
in `casehub-desiredstate/eidos` (existing bridge module) and `casehub-engine-api`
(new SPIs).

### 6.1 API Changes (eidos-org-api)

**OrganizationalUnit** — the declarative specification (desired state, D9). Add
fields for topology, policy, and selfDetermination. No runtime state here:

```java
public record OrganizationalUnit(
    String unitId,
    String name,
    String kind,                           // existing
    String kindVocabulary,                 // existing
    String tenancyId,                      // existing — required, non-null
    String parentUnitId,                   // existing
    List<Membership> members,              // existing
    List<AgentCapability> capabilities,    // existing — typed records
    List<AgentGoal> goals,                 // existing — typed records
    List<AgentConstraint> constraints,     // existing — typed records
    // NEW — declarative spec fields only
    String topology,                       // open-string vocabulary term (D20)
    String topologyVocabulary,             // vocabulary URI for topology grounding
    AuthorityModel authorityModel,
    GroupPolicy policy,
    double selfDetermination              // 0.0–1.0
) {}
```

Existing fields retain their types — `AgentCapability`, `AgentGoal`, and
`AgentConstraint` are typed domain records from `eidos-api`, not plain strings.
`tenancyId` remains required (compact constructor `Objects.requireNonNull`);
every registry query requires it for tenancy isolation.

**GroupInstance** — the mutable runtime entity (actual state, D9). Tracks
lifecycle, actual membership, current effective topology, and role adoptions:

```java
public record GroupInstance(
    String unitId,                         // references OrganizationalUnit.unitId
    String tenancyId,                      // required — tenancy isolation
    GroupLifecycleState state,             // PROPOSED, FORMING, ACTIVE, REORGANISING, DISSOLVING, DISSOLVED
    List<ActualMembership> members,        // who IS in the group right now
    String effectiveTopology,              // may differ from spec during transitions
    AuthorityModel effectiveAuthority,
    List<RoleAdoption> roleAdoptions,      // which agents currently hold which roles
    Instant stateChangedAt                 // timestamp of last state transition
) {}

public record ActualMembership(
    String agentId,
    Instant joinedAt
) {}

public record RoleAdoption(
    String agentId,
    String role,
    String roleVocabulary,
    Instant adoptedAt
) {}
```

**Invariant:** `roleAdoptions` is a subset of `members` by agentId — an agent
must be a member before adopting a role. `joinGroup()` adds to `members`;
`adoptRole()` adds to `roleAdoptions` (fails if not a member). `leaveGroup()`
removes from both `members` and `roleAdoptions` (releases all roles first).
An agent can be a member without a role (valid during formation, between role
transitions, or in `many`-topology groups where roles are optional).

`DesiredStateOrgAdapter` reads `GroupInstance` as actual state; `OrgGoalCompiler`
compiles `OrganizationalUnit` as desired state. The diff between them drives
reconciliation.

**Persistence:** `GroupInstance` is transient (in-memory via `OrgRegistry`).
The desired-state graph (persisted via `DesiredStateHistoryStore`) is the
authoritative recovery layer. On restart: the `DesiredStateHistoryStore`
provides the desired state, `OrgRegistry` is empty (no actual state), and
`ReconciliationLoop` re-provisions all groups from the desired-state graph —
the same path used for initial deployment (§7 unified boot path).

**Timestamp recovery semantics:** On restart, `ActualMembership.joinedAt`
and `RoleAdoption.adoptedAt` are set to the re-provisioning time (current
time). This is semantically correct — the agent joined the runtime group
instance at restart time. For normative conflict resolution,
`NormDecision.establishedAt` resets accordingly. This has minimal impact:
recency (`RecencyResolution`) is the last strategy in the conflict resolution
chain — priority and specificity handle the common cases. If two obligations
have equal priority and equal specificity post-restart, they will have equal
recency (both re-provisioned simultaneously), and resolution falls through to
escalation — the correct conservative behaviour when the system cannot
distinguish temporal ordering after a restart.

**Universal desired-state-first rule:** Every `GroupLifecycleService` mutation
updates the desired-state graph before updating `GroupInstance`. This applies
to all operations without exception: top-down formation, emergence promotion,
bottom-up proposal acceptance, membership changes (join/leave), role adoption/
release, topology transition, and dissolution. `SocietyLifecycleManager`
mutates the desired-state graph first, calls
`ReconciliationLoop.requestReconciliation()`, then updates `GroupInstance`.
This ensures the desired-state graph is always ahead of or in sync with
runtime state — restart recovery reconstructs from the authoritative graph.

**Single entry point for OrgRegistry mutations:** All writes to `OrgRegistry`
(both `OrganizationalUnit` and `GroupInstance`) go through
`SocietyLifecycleManager`. This includes reconciliation-driven provisioning:
`OrgNodeProvisioner.provision()` delegates to `SocietyLifecycleManager` for
group creation, membership updates, and role assignments — it does NOT write
to `OrgRegistry` directly. This ensures the per-society lock in
`SocietyLifecycleManager` protects all mutation paths. Without this, a
concurrent formation request (direct path) and reconciliation cycle
(reconciliation path) could write to `OrgRegistry` without shared mutual
exclusion.

Note: `OrgRegistry` currently has only an in-memory implementation
(`InMemoryOrgRegistry`). JPA persistence for `OrganizationalUnit` is future
work tracked under eidos#216. When implemented, the JPA-persisted
`OrganizationalUnit` records would provide a warm-start optimisation
(skip re-provisioning units that already exist), but correctness does not
depend on it — the desired-state reconciliation path is always available.

**GroupPolicy** — new record:

```java
public record GroupPolicy(
    Map<String, RolePolicy> roles,
    FormationRules formationRules,
    double topologyChangeThreshold,
    int normativePriority,           // lower wins in PriorityResolution; default: hierarchy depth
    NormSpecificity normSpecificity   // nullable — null means auto-derive (§2.2)
) {}

public record RolePolicy(
    List<String> obligations,
    List<String> permissions,
    int minAgents,
    int maxAgents
) {}
```

**RolePolicy cardinality vs MAX_MEMBERS:** `minAgents`/`maxAgents` are per-role
constraints. An agent can hold multiple roles, so the sum of `maxAgents` across
roles can exceed `MAX_MEMBERS` (50) — roles overlap. Validation rules:
- At formation: each role with `minAgents > 0` must have enough members assigned
- At dissolution/leave: if a member departure would leave a role below `minAgents`,
  the group enters REORGANISING to recruit a replacement before allowing the departure
- `MAX_MEMBERS` is a hard cap on distinct agents; role cardinality is orthogonal

**FormationRules** — new record:

```java
public record FormationRules(
    Set<FormationPath> allowedPaths,     // TOP_DOWN, EMERGENCE, BOTTOM_UP
    int minGroupSize,                    // minimum members to form (default: 2)
    int maxGroupSize,                    // max members at formation (≤ MAX_MEMBERS)
    boolean allowSubGroups,              // can this group contain sub-groups
    double proposalQuorum               // fraction of proposed members that must accept (default: 1.0)
) {}

public enum FormationPath {
    TOP_DOWN, EMERGENCE, BOTTOM_UP
}
```

**GroupFormationRequest** — new record:

```java
public record GroupFormationRequest(
    String parentUnitId,
    String tenancyId,
    String name,
    String kind,
    String kindVocabulary,
    String topology,                     // open-string vocabulary term (D20)
    String topologyVocabulary,
    AuthorityModel authorityModel,
    GroupPolicy policy,
    double selfDetermination,
    List<Membership> initialMembers,
    FormationPath formationPath          // which path created this request
) {}
```

All three formation paths (§3) produce a `GroupFormationRequest` and submit it
to `GroupLifecycleService.formGroup()`. The request carries the formation path
for auditing and policy enforcement — `FormationRules.allowedPaths` is checked
before the group is created.

**Topology-policy validation:** At formation and topology transition,
`GroupLifecycleService` validates consistency between topology and policy:
- `hive` activates coordinator-routed work dispatch. If no role in
  `GroupPolicy.roles` carries coordinator-class permissions (`change-topology`
  or `provision-agent`), a diagnostic warning is emitted: work dispatch will
  be inert until a coordinator-class role is defined and adopted. Formation
  proceeds — the group is valid but its coordination mechanism is inactive.
- `many` has no coordination primitives. Role definitions remain valid (they
  carry normative obligations even without coordination). No warning needed.
- `mesh` uses peer messaging. No specific role requirement — all members are
  peers.
- Unknown topology values: no validation (custom topologies define their own
  coordination semantics via TopologyProvider).

**GroupLifecycleState** — new enum with defined transitions:

```java
public enum GroupLifecycleState {
    PROPOSED, FORMING, ACTIVE, REORGANISING, DISSOLVING, DISSOLVED
}
```

Valid state transitions:
```
PROPOSED → FORMING       (proposal accepted, begin formation)
PROPOSED → DISSOLVED     (proposal rejected or CAS conflict)
FORMING → ACTIVE         (formation complete: all members joined, roles assigned)
FORMING → DISSOLVED      (formation aborted: timeout, evaluation failure, CAS conflict)
ACTIVE → REORGANISING    (topology change or desired-state transition initiated)
ACTIVE → DISSOLVING      (dissolution triggered by any path in §4)
REORGANISING → ACTIVE    (transition completed successfully)
REORGANISING → DISSOLVING (transition abandoned, dissolve instead)
DISSOLVING → DISSOLVED   (all members released, channel closed, work drained/reassigned)
```

During REORGANISING: membership is frozen (no join/leave), roles cannot be
adopted or released, the group can still communicate via its existing channel.
During DISSOLVING: no new work is accepted, existing work drains or is reassigned.

**OrgRegistry** — remains a pure CRUD store. No lifecycle operations added.
Existing methods (`registerUnit`, `removeUnit`, `findUnit`, `findUnits`,
`childUnits`, `ancestorUnits`, `unitsFor`, `membersOf`, relationship methods)
are unchanged. All methods require `tenancyId` — this invariant is preserved.

New query methods for GroupInstance:

```java
Optional<GroupInstance> findGroupInstance(String unitId, String tenancyId);
List<GroupInstance> findGroupInstances(OrgQuery query);
```

**GroupLifecycleService** — new SPI for lifecycle operations. All methods
require `tenancyId` (tenancy isolation). `SocietyLifecycleManager` (§6.2)
is the runtime implementation.

```java
public interface GroupLifecycleService {
    GroupInstance formGroup(GroupFormationRequest request);
    void dissolveGroup(String unitId, String tenancyId);
    void joinGroup(String unitId, String agentId, String role, String tenancyId);
    void leaveGroup(String unitId, String agentId, String tenancyId);
    void adoptRole(String unitId, String agentId, String role, String tenancyId);
    void releaseRole(String unitId, String agentId, String role, String tenancyId);
    void changeTopology(String unitId, String topology, String topologyVocabulary,
                        AuthorityModel authority, String tenancyId);
}
```

Callers interact with `GroupLifecycleService` for group operations, not
`OrgRegistry`. The lifecycle service orchestrates validation, event emission,
Qhorus channel management, and desired-state updates internally, calling
`OrgRegistry` for persistence.

### 6.2 Runtime Changes

**SocietyLifecycleManager** (eidos-org-runtime) — implements
`GroupLifecycleService`. Orchestrates group formation, dissolution, membership
changes. Emits events consumed by engine (role lifecycle, topology changes)
and desired-state (actual state updates). Calls `OrgRegistry` for persistence
of both `OrganizationalUnit` (spec) and `GroupInstance` (runtime state).

**DesiredStateOrgAdapter** (casehub-desiredstate/eidos module) — implements
desired-state's `ActualStateAdapter` SPI, handling `NodeType` `org:unit`. Reads
`GroupInstance` from `OrgRegistry` as actual state, presents it as a node graph
for diffing against the desired graph. Lives in `casehub-desiredstate-eidos`
alongside `OrgGoalCompiler` — both are desiredstate bridge types that depend on
`desiredstate-api` + `eidos-org-api`.

**OrgGoalCompiler** (casehub-desiredstate/eidos module) — already exists at
`io.casehub.eidos.org.runtime.OrgGoalCompiler` implementing
`GoalCompiler<OrgGoal>`. Currently compiles `List<OrganizationalUnit>` +
`List<AgentRelationship>` into `OrgUnitNodeSpec`/`OrgRelationshipNodeSpec` nodes.
This spec extends the existing compiler to handle the new fields (topology,
policy, selfDetermination, state) in its `OrgUnitNodeSpec`, and adds group-aware
dependency edges (parent→child ordering for staged provisioning). The compiler
stays in the desiredstate/eidos bridge module — it depends on both
`desiredstate-api` and `eidos-org-api`, which is the standard bridge pattern
(cf. `casehub-desiredstate/engine-adapter`).

### 6.3 Engine Integration

Engine consumes eidos org events through new SPIs defined as part of this spec:

**TopologyProvider** (engine#1284) — new SPI in `casehub-engine-api`:

```java
public interface TopologyProvider {
    Optional<String> topologyFor(String agentId, String tenancyId);
    double effectiveSelfDetermination(String agentId, String tenancyId);
}
```

Queries eidos OrgRegistry for the agent's group membership. Returns the group's
topology (open-string vocabulary term) and resolves selfDetermination precedence
(group value wins over `IntegrationPolicy` default when the agent is in a group).
The engine maps well-known topology values (`swarm`, `hive`, `mesh`, `many`) to
coordination primitives; unknown values receive default behaviour.

**AgentWorkLifecycleListener** (engine#1285) — new SPI in `casehub-engine-api`:

```java
public interface AgentWorkLifecycleListener {
    void onRoleAdopted(String agentId, String unitId, String role, String tenancyId);
    void onRoleReleased(String agentId, String unitId, String role, String tenancyId);
    void onTopologyChanged(String unitId, String fromTopology,
                           String toTopology, String tenancyId);
}
```

**TeamDetector** (existing) feeds DetectedTeam data to SocietyLifecycleManager
for emergence promotion. Promoted agents are excluded from subsequent detection
cycles via group membership tracking.

### 6.4 Qhorus Integration

When a group forms, SocietyLifecycleManager calls Qhorus to create the group's
PRIVATE channel (qhorus#509). Coordinator gets MODERATOR role. Membership
synchronised on join/leave. Channel closed (not deleted) on dissolution.

**Failure isolation:** Channel creation is best-effort — group formation does not
fail because Qhorus is unavailable. If channel creation fails, the group forms
without a channel and a drift record is created. ReconciliationLoop retries
channel creation on subsequent cycles until successful.

**Degraded coordination without channel:** When a group forms without a channel,
coordination behaviour is topology-dependent:
- `swarm`: uses stigmergy (group-scoped signal filtering), not channels.
  Unaffected by channel failure — fully operational.
- `hive`: coordinator dispatches work via engine work dispatch (not channel-
  dependent). The channel provides coordination communication (status reports,
  plan discussions). Without channel: work dispatch is functional, coordination
  communication is deferred. Degraded but operational.
- `mesh`: uses Qhorus peer messaging as its coordination mechanism. Without
  channel: mesh coordination is inert. The group is structurally valid (members,
  roles, policy all apply) but peer messaging is deferred.
- `many`: no coordination primitives. Unaffected.

When `ReconciliationLoop` successfully creates a deferred channel, it emits a
channel-ready event via `AgentWorkLifecycleListener`. Topology-specific
coordination mechanisms that were deferred (mesh peer messaging) activate upon
observing this event.

Coordinator changes trigger a Qhorus MODERATOR role transfer. If the transfer
fails, the old agent retains MODERATOR temporarily — the next reconciliation
cycle detects the divergence and retries. Membership sync uses the same
eventually-consistent pattern: OrgRegistry is authoritative, Qhorus channel
membership converges via reconciliation.

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

**Schema migration:** This YAML uses `society:` as the root key, replacing the
existing `organization:` root key used by `ClasspathYamlOrgRegistrar`. This is
a breaking schema change. The existing `organization:` schema does not support
topology, policy, or selfDetermination. Since eidos is pre-release (all
consumers are internal), the migration is a clean replacement — update all
`META-INF/eidos/organization.yaml` files to the new `society:` schema.

**Types replaced:** Three existing types require coordinated replacement:
1. `ClasspathYamlOrgRegistrar` — updated: new root key (`society:` replaces
   `organization:`), new resource path, new schema parser.
2. `YamlOrganization` — replaced: currently produces flat
   `List<OrganizationalUnit>`. New schema model handles nested `groups:` with
   inline `topology:`, `policy:`, `roles:`, and `members:`.
3. `OrgBootstrap` — superseded: currently calls `registry::registerUnit`
   imperatively for each unit. Replaced by the unified boot path:
   `ClasspathYamlOrgRegistrar` → `OrgGoalCompiler` → `ReconciliationLoop`
   → `OrgNodeProvisioner`. The imperative bootstrap is eliminated in favour
   of desired-state reconciliation for all org mutations.

This YAML is both the eidos seed (loaded at boot by ClasspathYamlOrgRegistrar)
and a desired-state goal specification (compiled by OrgGoalCompiler for
restructuring).

**tenancyId resolution:** The YAML omits `tenancyId` because it is a deployment-
time concern, not a design-time one. `ClasspathYamlOrgRegistrar` injects the
tenancyId from the deployment's tenant configuration (the same mechanism used
by the existing org YAML loader). Each `OrganizationalUnit` created from the
YAML receives the deployment tenant's ID. Multi-tenant deployments use separate
YAML seeds per tenant, each loaded under its respective tenancyId.

**Unified boot path (D8, D9):** At startup, `ClasspathYamlOrgRegistrar` checks
`DesiredStateHistoryStore` for an existing persisted graph for this tenancy:

- **Persisted graph exists** (restart after prior run): the persisted graph is
  authoritative. YAML is NOT recompiled — dynamic changes made at runtime
  (emergence promotions, bottom-up proposals, role adoptions, topology
  transitions) are preserved in the persisted graph and survive restart. The
  `ReconciliationLoop` runs reconciliation against the persisted desired state
  and the empty actual state (in-memory `OrgRegistry`), re-provisioning all
  groups to match the persisted graph.
- **No persisted graph** (fresh deployment): `ClasspathYamlOrgRegistrar` loads
  the YAML, passes it to `OrgGoalCompiler` to compile the initial desired-state
  graph, and the `ReconciliationLoop` runs initial reconciliation: desired graph
  vs empty actual state → `TransitionPlan` with all additions → auto-approved →
  execute (provisions all groups via `OrgNodeProvisioner`).
- **Operator wants to restructure via YAML:** they update the YAML and trigger
  an explicit restructuring operation (diagram A → diagram B, per §5.2). This
  recompiles from YAML, producing a new desired-state graph that the
  `ReconciliationLoop` transitions to. The boot path does not implicitly
  override the persisted graph — restructuring is always an explicit operation.

This eliminates the dual-path problem where boot uses imperative
`registerUnit()` while runtime uses desired-state reconciliation. One path
for all org mutations.

**Event model:** Society events (formation, dissolution, role lifecycle, topology
change) follow the platform's CDI event model — synchronous within a thread,
ordered per-aggregate (per-group). Cross-service events use CloudEvents via
the existing engine event bus. Ordering is guaranteed within a single group's
lifecycle; cross-group event ordering is not guaranteed and consumers must
tolerate out-of-order delivery across groups.

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
| **desired-state** | TBD-1 | Extend OrgGoalCompiler for topology/policy/state fields |
| **desired-state** | TBD-2 | OrgNodeProvisioner + OrgDriftPolicy + OrgFaultPolicy |
| **desired-state** | TBD-3 | DesiredStateOrgAdapter (ActualStateAdapter SPI impl) |

## References

- [MOISE+ (Hübner et al. 2002)](https://moise.sourceforge.net/doc/publications/Hubner-sbia2002.pdf)
- [AGR — From Agents to Organizations (Ferber et al. 2003)](https://www.madkit.net/madkit/docs/articles/FromAgentstoOrganizations_AAMAS03_FerbGutMich.pdf)
- [Society Agent (TechRxiv 2026)](https://www.techrxiv.org/doi/full/10.36227/techrxiv.177211798.85464735/v1)
- [Organisational Theory for Multi-Agent Interactions (Springer 2025)](https://link.springer.com/article/10.1007/s10791-025-09667-2)
- [Multi-Agent Oriented Reorganisation within JaCaMo (Sorici et al. 2012)](https://www.researchgate.net/publication/281328187_Multi_Agent_Oriented_Reorganisation_within_the_JaCaMo_infrastructure)
- Fit-gap research: `2026-11-02-society-moise-fitgap-research.md`
- Decisions: `2026-11-02-society-moise-fitgap-research-decisions.md`
