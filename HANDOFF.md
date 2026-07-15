# Handoff — 2026-07-15

## What's Done

CI green. AML diagnostic complete. Unified execution model spec drafted.

- 10 commits landed on main (d7e79017..a6df47c4): SNAPSHOT drift fixes, null inputData NPE, binding re-dispatch loop, test profile contamination, Flyway migration, recovery test timing, YAML mapper fixes
- AML tenant mismatch: reproduced, diagnosed (package relocation CDI exclusion + null inputData + binding re-dispatch). Diagnostic sent. Consumer reproduction test added.
- Unified execution model spec at `docs/specs/2026-07-15-unified-execution-model-design.md` — has contradictions from iterative design discussion that need cleaning up before sharing with trebleel
- 4 stranded blog entries promoted to workspace main, 1 published to mdproctor.github.io

## Immediate Next Step

Revisit `docs/specs/2026-07-15-unified-execution-model-design.md` — resolve contradictions and reach a universal execution model that is richer and stronger than LangChain4j's.

**Where we exceed LangChain4j — preserve these advantages:**
- Two orthogonal dispatch dimensions (choreography + orchestration) composable per-node — LangChain4j has neither orthogonality nor per-node composition
- Compound PlanItems with nested strategies — arbitrary depth, each level can use a different algorithm
- Plan graph as model, CaseInstance as runtime — clean separation (their AgenticScope mixes both)
- Planning algorithms as peers (Sequential, Flow, HTN, GoalOriented) producing a universal output format (`DagPlan<T>`) — they have separate Planner implementations with no shared plan representation

**Where LangChain4j has gaps ours fills:**
- Their planners are flat — no composability (Sequential can't delegate a phase to GoalOriented)
- P2P (choreography) and workflow (orchestration) are separate impls with no hybrid — you pick one
- No compound tasks or hierarchical scoping within a single execution
- No separation of dispatch mode (when/now) from planning algorithm (how) — mixed into each Planner

**LangChain4j pattern mapping (verify ours covers all):**

| LangChain4j | Ours | Category |
|---|---|---|
| SequentialPlanner | Sequential | Planning algorithm |
| ParallelPlanner | Choreography (all concurrent) | Dispatch mode |
| LoopPlanner | Flow (control flow) | Planning algorithm |
| ConditionalPlanner | Flow (branching) | Planning algorithm |
| GoalOrientedPlanner | (unnamed) — graph search over agent I/O | Planning algorithm |
| Supervisor | HTN with LLM solver (or blocks technique) | Planning / technique boundary |
| P2PPlanner | Choreography — ContextChangeTrigger | Dispatch mode |

**The dispatch model — two dimensions, composable per-node:**

Dispatch modes (the two archetypes — EVERYTHING maps into these):
- Choreography — "do this when" (trigger-driven, reactive, pull)
  - Context-condition (ContextChangeTrigger)
  - Time-based (ScheduleTrigger — "when time >= X")
  - Event-based (Signal/message arrival)
- Orchestration — "do this now" (strategy-driven, proactive, push)
  - Selection criteria (how the strategy picks): priority-based, goal-driven, resource-aware
  - Planning algorithms (how the strategy builds the plan): Sequential, Flow, HTN, (unnamed goal-directed)

Modifiers (apply to either mode): resource-gated, speculative

Plan representation (output, not algorithm): `DagPlan<T>` — universal format all algorithms produce

**Planning algorithms (under orchestration):**

| Algorithm | What you specify | What the solver finds |
|---|---|---|
| Sequential | The steps, in order | Nothing — fixed list |
| Flow | The control flow (loops, conditionals, compensation) | Nothing — fixed graph |
| HTN | Decomposition methods (or LLM generates them) | How to break compound tasks into primitives |
| (unnamed) | Operators (capabilities with I/O schemas) + goal state | Sequence of operators that reaches the goal |

Key insight: Sequential stays simple (ordered list). Need loops? Use Flow. Don't grow Sequential into a workflow. Flow already partially exists as casehub-engine-flow (Serverless Workflow SDK) but positioned as worker execution tier, not planning strategy.

ReAct is NOT a separate algorithm — it's the native CONTEXT_CHANGED evaluation loop. Every cycle: strategy evaluates state (Thought) → dispatches worker (Action) → context changes from output (Observation) → repeat.

**What to fix:** Spec has contradictions (Section 10: C1-C4) and open questions (Q1-Q7). Start with C1 (choreography as mode vs strategy name) — it's the root confusion. Then verify against engine#101 sub-issues.

## Cross-Module

- AML — rebuild against latest engine SNAPSHOT (binding re-dispatch fix b1e9a4c3 + null inputData fix 55a602e6). Gate issue is CDI exclusion of relocated work-engine-adapter classes.
- casehub-work — engine SNAPSHOTs now published to GitHub Packages. CI should resolve.
- casehub-blocks — DAG unification (Phase 0: retire ExecutionPlan, adopt DagPlan) is first convergence step. Requires sequentialMerge() on DagPlan.

## What's Left

- Spec contradictions: choreography appears as both a dispatch archetype and a planning strategy name in the tables · S · Med
- engine#101 sub-issues: verify unified model covers all LangChain4j patterns (Sequential, Supervisor, GoalOriented, P2P, Loop, Conditional) · M · Med
- 202 local engine branches + 90 workspace branches stamped/closed — bulk deletion approved but not executed · XS · Low
- casehub-blocks: TrustRoutingPolicy 8th param propagation (3 test files) · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | Rewrite spec: clean model (two dispatch modes, planning algorithms under orchestration, DagPlan as output format) | M | Med | Coordinate with trebleel before implementing |
| — | Phase 0: sequentialMerge() on DagPlan | S | Low | Prerequisite for blocks ExecutionPlan retirement |
| — | Phase 1: Retire ChoreographyLoopControl | M | Med | PlanningStrategyLoopControl becomes sole LoopControl |
| #732 | Wire CaseContextStoreFactory through recovery path | M | Med | Blocks durable factories |
| #600 | HTN — hierarchical task decomposition | L | High | Under #595 epic — spec informs design |
| #689 | WorkItems boundary — typed payload | M | Med | ContextBridge arc |
