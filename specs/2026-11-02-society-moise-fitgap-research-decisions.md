# Design Decisions — Self-Organising Agent Society

**Issue:** casehubio/eidos#216
**Date:** 2026-11-02

## D1: One dynamic model; eidos static is a seed

**Choice:** Single dynamic society model. Eidos static declarations (YAML, annotations) pre-populate the dynamic structure at boot — same as seeding a database.
**Alternatives:**
- Dual-mode (static + dynamic as separate concerns) — adds complexity for no benefit; static is just a special case of dynamic
- Emergent-only (no static declarations) — loses the ability to pre-wire known structures
**Rationale:** One implementation, one model. The static bootstrap path already exists and doesn't need to change — it just feeds into a model that now also supports runtime mutation.
**Trade-offs:** Static declarations must conform to the dynamic model's constraints (topology, policy fields).
**Sources:** Eidos org-api, MOISE+ OE instantiation pattern
**Exploration:** quick
**Status:** captured

## D2: Topology is a mutable group property

**Choice:** Groups have a CoordinationPattern (SWARM/HIVE/MESH/MANY) that can change at runtime. selfDetermination governs how freely groups can shift topology.
**Alternatives:**
- Fixed at formation — simpler but can't adapt. A detected swarm cluster that needs structured coordination would have to dissolve and reform as a Hive
- Inherited from parent — gives top-down consistency but removes local flexibility
**Rationale:** The Swarm → Hive transition is a core use case: agents cluster via stigmergy, stabilise, then adopt roles and supervised coordination. Blocking this transition forces unnecessary group dissolution/recreation.
**Trade-offs:** Topology transitions need careful state management — signal spaces, channels, role assignments may change.
**Sources:** Agentic engineering topology taxonomy (Swarm/Hive/Mesh/Many), engine IntegrationPolicy.selfDetermination
**Exploration:** quick
**Status:** captured

## D3: Group = identity + topology + policy + roles

**Choice:** A group is a composite of four concerns. The coordinator is a policy-designated role, not a special abstraction. No synthetic group-as-agent wrapper.
**Alternatives:**
- Group-as-agent (GroupAgent implements AgentRef) — adds a synthetic wrapper that duplicates what a coordinator role already provides
- Container-only (no policy or topology) — too thin; groups need to govern their own coordination
**Rationale:** A coordinator IS a real agent with its own capabilities, constrained by the group's policy. The group acts through its coordinator, not through an abstraction. This is how real organisations work — you interact with the team lead, not "the team."
**Trade-offs:** No guaranteed collective decision-making — the coordinator chooses when to consult the group. Policy can mandate consultation but can't structurally enforce it the way a GroupAgent adapter would.
**Sources:** MOISE+ deontic dimension, agency-core NormativeReasoning, orchestration patterns (supervisor, debate, voting)
**Exploration:** deep-analysis
**Status:** captured

## D4: N-level recursive hierarchy

**Choice:** One concept: Group. A group can contain agents and/or sub-groups. Society is the root group, team is a leaf group. Any depth in between.
**Alternatives:**
- Fixed three levels (Society/Group/Team as distinct types) — rigid, can't model deeper hierarchies
- Two levels (Society + Group) — simpler but forces flat group structures
**Rationale:** Recursive composition is simpler (one type) and more flexible (any depth). Matches AGR where groups can nest, and aligns with eidos OrganizationalUnit which already has parentUnitId.
**Trade-offs:** No structural enforcement of depth limits — a 10-level deep hierarchy is valid even if unwise. Policy can constrain this.
**Sources:** AGR (Ferber et al. 2003), eidos OrganizationalUnit.parentUnitId
**Exploration:** quick
**Status:** captured

## D5: Inter-group communication — stigmergy primary, shared context optional

**Choice:** Groups discover and communicate through stigmergy signals (same mechanism agents use). Shared context space available for simple use cases where groups need ambient awareness.
**Alternatives:**
- Explicit channels only — cleaner boundaries but no emergent inter-group discovery
- Shared context space only — simplest but noisiest, doesn't scale
**Rationale:** Consistent with how individual agents already work in the swarm. Groups deposit and observe signals the same way agents do. Shared context is an opt-in simplification, not the primary mechanism.
**Trade-offs:** Stigmergy is indirect — groups can't address each other specifically through signals alone. For sustained collaboration, Qhorus PRIVATE channels (qhorus#509) provide direct communication.
**Sources:** Engine stigmergy package, Qhorus channel model
**Exploration:** quick
**Status:** captured

## D6: All three formation paths

**Choice:** Bottom-up proposal (agents use coalition primitives), emergence promotion (DetectedTeam stabilises and becomes a group), and top-down spawning (parent creates sub-group). selfDetermination controls which paths are available.
**Alternatives:**
- Emergence + top-down only — simpler but agents can't proactively organise
- Proposal-only — most controlled but loses emergent self-organisation
**Rationale:** Each path serves a different need. Bottom-up for agent initiative. Emergence for organic structure. Top-down for intentional design. selfDetermination is the dial — low values restrict to top-down only; high values enable all three.
**Trade-offs:** Three formation paths increase complexity. Need clear rules for conflicts (what if a top-down restructure contradicts an emergent formation?). Desired-state reconciliation handles this — the most recent desired-state wins.
**Sources:** Agency-core CoalitionProposal, engine DetectedTeam/TeamDetector, eidos OrgRegistry
**Exploration:** quick
**Status:** captured

## D7: Dynamic society runtime lives in eidos

**Choice:** Grow eidos from identity/config store into identity + dynamic society runtime. Engine depends on eidos directly. Agency is downstream.
**Alternatives:**
- Agency-core — has the deliberation primitives but is downstream of engine
- New module — clean separation but another dependency to manage
**Rationale:** Eidos already owns identity, org structure, and relationships. Engine depends on it directly. Putting the society runtime in agency would create a circular dependency or force engine to duplicate org concepts.
**Trade-offs:** Eidos grows from a "starting configuration" store into a runtime. Needs clear internal separation between identity (what exists) and lifecycle (what happens).
**Sources:** Eidos org-api/org-runtime modules, engine dependency graph
**Exploration:** quick
**Status:** captured

## D8: Reorganisation protocol via casehub-desiredstate

**Choice:** Use casehub-desiredstate for organisational transitions. GoalCompiler compiles target org into a graph. TransitionPlanner diffs current vs desired. ReconciliationLoop handles drift.
**Alternatives:**
- Custom reorganisation protocol in eidos — reinvents desired-state's graph diffing, staging, fault handling
- JaCaMo-style OrgManager — single central manager that halts the organisation during transitions (ParaMoise addresses this but adds complexity)
**Rationale:** Desired-state already handles graph-based state reconciliation with approval gates, drift detection, staged rollouts, and CBR fault policy. Its README lists agent topology as a declared use case. The fit is direct — org units are nodes, relationships are edges.
**Trade-offs:** Dependency on desired-state. Org transitions become eventually-consistent (reconciliation loop, not instant). This is a feature for complex restructures but could feel slow for simple changes.
**Sources:** casehub-desiredstate (GoalCompiler, TransitionPlanner, ReconciliationLoop, PlanApprovalGate, SituationRecompiler), MOISE+ reorganisation, ParaMoise
**Exploration:** deep-analysis
**Status:** captured

## D9: Eidos org diagrams as desired-state goal specifications

**Choice:** Org diagrams are declarative, idempotent, git-versioned desired-state targets. You never imperatively restructure — declare the target state and desired-state transitions to it. Emergent formation = runtime editing the desired-state graph when selfDetermination permits.
**Alternatives:**
- Imperative API (addGroup, removeAgent, moveAgent) — direct but loses diffability, auditability, and rollback
- Hybrid (declarative for planned changes, imperative for emergency) — adds two code paths for the same operation
**Rationale:** Infrastructure-as-code for organisational structure. Same mechanism handles initial deployment (no state → diagram A), restructuring (A → B), recovery (drift → reconcile), and rollback (B → A). Top-down and bottom-up use the same path — the only difference is who edits the target state.
**Trade-offs:** All changes go through the reconciliation loop, even trivial ones. Desired-state's SituationRecompiler adds latency for adaptive restructuring. Acceptable for organisational changes which are inherently non-trivial.
**Sources:** casehub-desiredstate, infrastructure-as-code pattern (Terraform, Kubernetes)
**Exploration:** quick
**Status:** captured

## D10: Evolve eidos-org in place

**Choice:** Extend existing OrganizationalUnit with topology, policy, and dynamic lifecycle. No new modules, no migration.
**Alternatives:**
- New eidos-society module — cleaner separation but creates parallel models and migration cost
- Two-phase (extend first, extract later) — pragmatic but YAGNI
**Rationale:** Project is pre-release — no backward compat concerns. The existing org model has the right shape (hierarchy, membership, relationships, roles). It just needs lifecycle and topology added. The static bootstrap becomes one seeding path among several.
**Trade-offs:** The org module grows larger. If it becomes unwieldy, extraction is always possible later.
**Sources:** Eidos org-api module structure, project maturity (pre-release)
**Exploration:** quick
**Depends on:** D7 (runtime lives in eidos)
**Status:** captured
