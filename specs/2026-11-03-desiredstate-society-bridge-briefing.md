# Briefing: casehub-desiredstate#211 — Society Lifecycle Bridge

**Issue:** casehubio/casehub-desiredstate#211
**Date:** 2026-11-03
**Upstream (landed):** eidos#216 (types, storage), qhorus#509 (channel visibility)
**Downstream (in progress):** engine slot 220 (TopologyProvider, AgentWorkLifecycleListener)

## Context

The self-organising agent society model is split across repos:
- **eidos** (CLOSED, landed) — types, storage, SPIs: GroupInstance,
  GroupLifecycleService interface, GroupInstanceStore, RoleSpec, GroupPolicy,
  FormationRules, GroupLifecycleState, AuthorityModel
- **qhorus** (CLOSED, landed) — channel visibility: PUBLIC/PRIVATE/DIRECT,
  createGroupChannel, membership sync API
- **desiredstate#211** (THIS WORK) — lifecycle orchestration: the
  GroupLifecycleService implementation + desired-state reconciliation bridge

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

Engine does NOT call GroupLifecycleService. Engine emits SWARM_TEAM_FORMED
events. A listener in desiredstate/eidos handles emergence promotion.

## What to Build

### 1. Move DefaultGroupLifecycleService from eidos

eidos-org-runtime has `DefaultGroupLifecycleService` implementing
`GroupLifecycleService`. Move the implementation here, make it
desired-state-aware: write desired-state graph FIRST, then delegate to
OrgRegistry/GroupInstanceStore for storage. The eidos version will be
replaced with an `@DefaultBean` stub (eidos#227, blocked by this).

### 2. Extend OrgGoalCompiler

Already exists at `io.casehub.eidos.org.runtime.OrgGoalCompiler` in
desiredstate/eidos. Currently compiles `List<OrganizationalUnit>` +
`List<AgentRelationship>` into nodes.

Extend:
- Add topology, policy, selfDetermination fields to `OrgUnitNodeSpec`
- Add group-aware dependency edges (parent→child ordering for staged
  provisioning)

### 3. New: OrgNodeProvisioner

`NodeProvisioner` impl for `org:unit` node type.

- `provision()`: write OrganizationalUnit (spec) + GroupInstance
  (FORMING→ACTIVE), create Qhorus PRIVATE channel, assign roles.
  Delegates to GroupLifecycleService for all OrgRegistry writes.
- `deprovision()`: dissolve group (work-aware), close channel

### 4. New: DesiredStateOrgAdapter

`ActualStateAdapter` impl — reads GroupInstance from OrgRegistry,
presents as node graph for diffing against the desired graph.

### 5. New: OrgDriftPolicy

Detects:
- Member departure below minimum
- Role gap (obligation unfulfilled)
- Autonomous topology change (selfDetermination-permitted but diverges
  from desired-state target)
- Natural dissolution (all members left)

Decisions: RECONCILE, IGNORE (within policy bounds), ESCALATE.

### 6. New: OrgFaultPolicy

Recovers:
- Group formation failure → re-attempt with substitute agents
  (CBR-informed), or escalate to review node
- Agent migration failure (mid-work) → PENDING_DRAIN, provision substitute
- Channel creation failure → retry on next reconciliation cycle

### 7. New: Emergence Promotion Listener

Listens to engine's `SWARM_TEAM_FORMED` events.

Maps `DetectedTeam.dominantRelations` to initial topology:

| Relation (priority order) | Default Topology |
|---|---|
| COMPLEMENTARY (highest) | hive |
| SHARED_INTEREST | mesh |
| COACTIVE | swarm |
| SHARED_SIGNAL (lowest) | swarm |

Calls `GroupLifecycleService.formGroup()` with
`FormationOrigin.EMERGENCE`.

## Dependencies

- `desiredstate-api` (already on classpath)
- `eidos-org-api` (already on classpath)
- **Add**: `qhorus-api` for channel creation (best-effort — group forms
  without channel if Qhorus unavailable)

## Key Design Rules

- **Desired-state-first (D9)**: every GroupLifecycleService mutation
  writes desired-state graph before OrgRegistry
- **Single entry point**: all OrgRegistry writes go through this
  GroupLifecycleService impl. OrgNodeProvisioner delegates to it, not
  to OrgRegistry directly
- **Formation serialization**: per-society lock + per-agent membership
  version CAS (spec §3)
- **Qhorus best-effort**: channel creation failure → drift record →
  ReconciliationLoop retries (spec §6.4)
- **Stabilization window**: after recompilation, 5-minute per-society
  cooldown prevents cascade storms (spec §5.4)
- **Topology transitions use REORGANISING state**: blocks concurrent
  mutations, rolls back on failure (spec §2.1)
- **Unified boot path**: check DesiredStateHistoryStore for persisted
  graph → if exists, reconcile from it; if not, compile YAML → reconcile
  (spec §7)

## Qhorus Integration

When a group forms, create Qhorus PRIVATE channel (qhorus#509):
- Coordinator gets MODERATOR role
- Other members get PARTICIPANT role
- `boundGroupId` in metadata links channel to group
- Membership synced on join/leave
- Channel closed (not deleted) on dissolution — retains history

Degraded behaviour without channel:
- `swarm`: unaffected (uses stigmergy)
- `hive`: partially operational (work dispatch works, coordination deferred)
- `mesh`: inert (peer messaging is the mechanism)
- `many`: unaffected (no coordination)

## Concrete Scenario (from spec)

Insurance claims processing — cyber-insurance product launch. 400%
claim volume spike triggers:
1. GoalCompiler produces new desired state (add CyberClaims group, shrink AutoClaims)
2. TransitionPlanner diffs — Phase 1: form, Phase 2: migrate agents, Phase 3: stabilise topology
3. PlanApprovalGate — selfDetermination controls approval (AutoClaims 0.3 requires approval, FraudWatch 0.7 auto-approves)
4. Drift detection — agent A3 mid-case, migration stalls → PENDING_DRAIN + provision substitute
5. SituationRecompiler — fraud rate spike → spawn CyberFraudCell sub-group

Full scenario in research doc.

## Spec Location

Full design spec (994 lines, reviewed with 85 issues resolved):
`wksp/specs/2026-11-02-self-organising-agent-society-design.md`

Fit-gap research + decisions:
`wksp/specs/2026-11-02-society-moise-fitgap-research.md`

Note: `wksp/` in the desiredstate project may not have a symlink to the
engine workspace. The spec files are in the engine workspace at:
`/Users/mdproctor/claude/public/casehub/engine/specs/`
