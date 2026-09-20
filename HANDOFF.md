# Handoff — Hive Mind Epic

**Branch:** `issue-1104-hive-mind`
**Epic:** casehubio/engine#1104
**Slot:** 197
**Date:** 2026-09-20

## What This Is

Self-organizing agent coordination patterns for CaseHub — from rule-based stigmergy through LLM-enhanced swarm intelligence to fully autonomous self-evolution.

## What Happened This Session

Advanced queue from #1113 to #1114. Closed #1113 on GitHub (all tasks done from previous session). Then: full brainstorming + research cycle for #1114 (Autonomous self-improvement). The design evolved significantly during brainstorming — what started as "agents submit PRs through DevTown" became a cognitive self-improvement architecture using the full blocks/neocortex cognitive stack.

### #1113 — Self-Provisioning Swarm (CLOSED)

Closed on GitHub at session start. All implementation was done in the previous session. Ticked off the epic checkbox.

### #1114 — Autonomous Self-Improvement (DESIGN PHASE — IN PROGRESS)

**Design brainstorming completed.** 14 decisions captured (D92–D105), Standard decision review passed (3 rounds, approved). The design evolved through several user-driven course corrections:

**Key design decisions:**
- D92: Signals trigger goals — two-layer identification (signals = sensors, goals = actuators)
- D93: Case-as-improvement — each improvement spawns a child case via SubCaseBinding
- D94: Full working engine implementations (not stubs) — same pattern as all hive mind issues
- D95: Two-dimensional improvement taxonomy (operational + capability) with research-driven growth loop. Operational = deps, lint, coverage, CI, recipes. Capability = reasoning, strategies, tools, techniques via internet/Google Scholar research
- D96: DevTown as standard `code-review` capability worker — no special SPI
- D97: Layered ImprovementBudget with structural self-modification denial
- D98: Three-layer event-sourced outcome tracking — EventLog (source of truth) + signals (real-time) + CBR (historical)
- D99: GoalKind.SELF_IMPROVEMENT — goal evaluators apply improvement-specific logic
- D100: #1114 = full single-shot cycle; #1115 = continuous loop + growth direction
- D101: Hybrid Java/YAML case template (existing convention)
- D103: Drive system integration — balanced competing needs (blocks DriveOrchestrator), NOT rigid Maslow hierarchy. CURIOSITY/COMPETENCE/AFFILIATION/AUTONOMY axes fed by improvement metrics
- D104: Self-improvement as a cognitive agent — full CognitionCore stack (PAD mood, drives, narrative, strategy learning, mental model, goals, memory hygiene). Both centralised (dedicated cognitive agent) AND distributed (swarm-wide sensing)
- D105: Full cognitive memory — MindMap as the agent's lived experience (builds, interactions, relationships, research, emotional associations), not just research storage. Consolidation, curiosity signals, mood-congruent retrieval

**Research document written:** 650+ line research paper at `wsp/specs/issue-1104-hive-mind/2026-09-20-cognitive-self-improvement-research.md` covering:
- Theoretical foundations (BDI, PAD, drives, swarm+cognitive architecture, memory, generative agents)
- Full CaseHub cognitive stack capabilities inventory (9 subsystems mapped)
- The closed cognitive loop architecture
- Configurable layered design ("head in the clouds, feet on the ground")
- Goodhart's Law as open risk
- Human interaction thesis (cognitive agents as collaboration partners)
- Feasibility analysis + 24 cited references

**Adversarial review completed:** `wsp/specs/issue-1104-hive-mind/2026-09-20-adversarial-review.md` — 8 attack angles. Key findings:
- "Research loop is fantasy" → rebutted (this IS how CaseHub is built)
- "Anthropomorphic theatre" → rebutted (WackyManor evidence + model capability trajectory)
- Goodhart's Law / approval optimisation → best punch, structural risk acknowledged
- Most concerns resolved by configurable layered approach

**Scope expanded to multi-epic.** The design is larger than a single issue:

| Epic | Scope | Repo | Status |
|------|-------|------|--------|
| 1. Engine foundation | Case lifecycle, signals, ImprovementBudget, rule-based workers, DevTown | engine | Next to spec + implement |
| 2. Cognitive agent | CognitionCore integration, improvement DriveSource implementations, PAD feedback | blocks + engine | Future — after epic 1 proves mechanical foundation |
| 3. Research loop | Search infrastructure, paper retrieval, LLM synthesis | blocks + neocortex | Future — automated version of current dev methodology |
| 4. Full cognitive memory | MindMap integration, consolidation, emotional associations | neocortex | Future — after epic 2 proves cognitive value |
| 5. Continuous evolution (#1115) | Standing directive, feedback loop, growth direction | engine + blocks | Future — wraps epics 1-4 in autonomous loop |

### Slot maintenance

- Rebased engine branch onto latest origin/main (clean, no conflicts)
- Switched blocks, eidos, qhorus to main and pulled latest (all in sync with canonical)
- Nuked slot .m2 (333M) for fresh artifacts on next build

### Known issues (pre-existing, unchanged)

- API module checkstyle violations (pre-existing). Build passes with `-Dcheckstyle.skip=true`.
- 5 compilation errors in engine-support-core (pre-existing).

## Queue (2 remaining)

Active: #1114 — design phase in progress (decisions captured, research document written, need parent spec + child spec + implementation plan)

Remaining: #1115

## Repos in Slot

| Repo | Path | Branch | Role |
|------|------|--------|------|
| engine | `slots/197/engine` | `issue-1104-hive-mind` | Primary — rebased to latest main |
| blocks | `slots/197/blocks` | `main` | Synced — cognitive stack source |
| eidos | `slots/197/eidos` | `main` | Synced |
| qhorus | `slots/197/qhorus` | `main` | Synced |

## Artifacts

| Artifact | Path |
|----------|------|
| Research document | `wsp/specs/issue-1104-hive-mind/2026-09-20-cognitive-self-improvement-research.md` |
| Adversarial review | `wsp/specs/issue-1104-hive-mind/2026-09-20-adversarial-review.md` |
| Decisions (D1–D105) | `wsp/specs/issue-1104-hive-mind/decisions.md` |
| Pipeline state | `wsp/specs/issue-1104-hive-mind/pipeline.state` |
| Cognitive stack inventory | `/tmp/neocortex-capabilities.md` (session artifact — copy to workspace if needed) |
| Memory stack inventory | `/tmp/neocortex-memory-stack.md` (session artifact — copy to workspace if needed) |
| All prior specs (#1105–#1113) | `wsp/specs/issue-1104-hive-mind/*.md` |
| All prior plans (#1107–#1113) | `wsp/plans/*.md` |
| Queue | `wsp/.plan` |

## Next Session — How to Proceed

The design brainstorming surfaced a multi-epic scope. Here's the path forward:

### Immediate (this branch, #1114)

1. **Write parent spec** — draws from the research document. Captures the full cognitive self-improvement vision as the north star. Lives in workspace specs.

2. **Write #1114 child spec (Epic 1: Engine Foundation)** — scoped to what engine delivers:
   - Improvement signal types (`improvement:stability:*`, `improvement:quality:*`, `improvement:capability:*`)
   - Signal → goal formation bridge (GoalKind.SELF_IMPROVEMENT)
   - ImprovementBudget + ImprovementBudgetEnforcer (layered, structural deny-list)
   - Improvement case template (hybrid Java/YAML, conditional bindings)
   - Rule-based workers: DependencyUpdateWorker, LintFixWorker, CoverageGapWorker, CITriageWorker, RecipeWorker
   - DevTown as standard `code-review` capability worker
   - Outcome tracking (all 3 layers: EventLog + signals + CBR traces)
   - SPI extension points for epics 2–4

3. **Implementation plan** (writing-plans) for #1114 child spec

4. **Implement** Epic 1

### Future (separate issues, cross-repo)

5. **File issues** for epics 2–5 in appropriate repos (blocks, neocortex, engine)
6. **Epic 2** (blocks + engine) — cognitive agent integration, improvement DriveSource implementations
7. **Epic 3** (blocks + neocortex) — research loop automation
8. **Epic 4** (neocortex) — full cognitive memory
9. **Epic 5** = #1115 (engine + blocks) — continuous evolution loop

### Design principle to carry forward

**Head in the clouds, feet on the ground.** Every cognitive capability is independently configurable, independently measurable, and independently toggleable via `CognitionConfig` pattern. Build the full vision; configure for current reality.
