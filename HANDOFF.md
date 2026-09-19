# Handoff — Hive Mind Epic

**Branch:** `issue-1104-hive-mind`
**Epic:** casehubio/engine#1104
**Slot:** 197
**Date:** 2026-09-19

## What This Is

Self-organizing agent coordination patterns for CaseHub — from rule-based stigmergy through LLM-enhanced swarm intelligence to fully autonomous self-evolution.

## What Happened This Session

Fixed RuntimeBeans constructor wiring (pre-existing from #1107-#1111), advanced queue from #1111 to #1112, completed full design cycle for #1112 (Swarm execution model), and implemented 2 of 4 batches.

### RuntimeBeans Fix

Wired 18 missing constructor parameters across 4 classes + 1 new producer:
- CaseStatusChangedHandler (+7 params: observation, signal, convergence, stigmergy)
- CaseContextChangedEventHandler (+8 params: same + BudgetEnforcer)
- ScopedWorkerTerminationHandler (+2 params: ObservationRegistry, RuleRegistry)
- WorkerRuntimeFactory (+1 param: RuleRegistry)
- BudgetEnforcer (new producer — plain class, no CDI annotation)

### #1112 — Swarm Execution Model (IN PROGRESS)

Design complete, implementation 2/4 batches done.

**Design cycle:** 10 decisions (D73-D82, 1 deep-analysis on fingerprinting), standard decision review (3 rounds, 5 accepted revisions to foundation decisions + D83 new decision), 855-line spec, standard post-spec review (3 rounds), 2,223-line implementation plan (6 tasks, 4 batches).

**Key design decisions:**
- D73: Swarm extends stigmergy (same `planningStrategy: stigmergy`, SwarmConfig nests inside StigmergyConfig)
- D74: Multi-dimensional behavioral fingerprinting (4 domains: perception, communication, decision, effect with weighted cosine similarity)
- D75: Signal-based team affinity (emergent clusters from shared interests/signals/complementary relations)
- D76: Work redistribution via departure events + rule reactions (no engine-orchestrated redistribution)
- D77: SwarmProgressTracker with 3 metrics (exploration pace, consensus formation, stability)
- D78: MetricsSpace as 5th WorkerRuntime facet (read-only agent self-awareness)
- D80: Dual-trigger detection (periodic + event-triggered via dirty flag)

**Implementation progress:**

Batch 1: API Types (DONE)
- 7 new files: SwarmConfig, RoleDomainWeights, BehavioralFingerprint, DetectedRole, DetectedTeam, SwarmProgress, MetricsSpace
- StigmergyConfig gains `swarm` field, WorkerRuntime gains `metrics()`, 7 new CaseHubEventType values
- 14 files changed, all tests green

Batch 2: RoleTracker (DONE)
- SwarmEvent record, RoleTracker with fingerprint computation, cosine similarity, sliding window accumulation, connected component clustering, role evolution detection
- 12 tests all green
- Key fix: `cos(empty, empty) = 1.0` for correct role comparison when both agents lack activity in a domain

Batch 3: TeamDetector + SwarmProgressTracker (NOT STARTED)
Batch 4: Wiring + Integration (NOT STARTED)

### Decision review side-effects on foundation decisions

The standard decision review for D73-D82 also revised 7 foundation decisions:
- D3: documented history buffer default rationale
- D6: `Future.cancel(true)` for actual thread interruption
- D20: `allowProgrammaticObservers` gate on InterestSpace
- D42: deterministic cross-agent write dedup + CONTEXT_WRITE_CONFLICT event
- D51: OutputConvergenceMonitor scoped to traditional worker outputs
- D58: halfLife quiescence documentation requirement
- D63: validation failure at init for explicit triggers on stigmergy bindings
- D66: maxEvaluationCycles default reduced from 50000 to 10000
- D83 (new): Pipeline decomposition — extract 5 phases from CaseContextChangedEventHandler (separate issue)

### Known issue: RuntimeBeans.java wiring

`runtime/src/main/java/io/casehub/engine/internal/quarkus/RuntimeBeans.java` — the explicit CDI bean construction doesn't pass the new constructor parameters for the swarm trackers (RoleTracker, TeamDetector, SwarmProgressTracker). This will be fixed in Batch 4 Task 5 (Wiring).

### Known issue: pre-existing checkstyle failures

API module has 18 pre-existing checkstyle violations (present before swarm changes). Build passes with `-Dcheckstyle.skip=true`. Not introduced by this branch.

## Queue (3 remaining)

Active: #1112 (Batch 3-4 remaining)

Remaining: #1113-#1115.

## Repos in Slot

| Repo | Path | Role |
|------|------|------|
| engine | `slots/197/engine` | Primary |
| blocks | `slots/197/blocks` | LLM-enhanced observation (future) |
| eidos | `slots/197/eidos` | Proximity/topology query (future) |
| qhorus | `slots/197/qhorus` | Topic-based broadcast (future) |

## Artifacts

| Artifact | Path |
|----------|------|
| Design spec (#1105) | `wsp/specs/issue-1104-hive-mind/2026-09-16-environment-observation-spi-design.md` |
| Design spec (#1106) | `wsp/specs/issue-1104-hive-mind/2026-09-16-signal-pheromone-model-design.md` |
| Design spec (#1107) | `wsp/specs/issue-1104-hive-mind/2026-09-16-dynamic-interest-registration-design.md` |
| Design spec (#1108) | `wsp/specs/issue-1104-hive-mind/2026-09-16-agent-discovery-neighbor-awareness-design.md` |
| Design spec (#1109) | `wsp/specs/issue-1104-hive-mind/2026-09-16-local-rule-evaluation-design.md` |
| Design spec (#1110) | `wsp/specs/issue-1104-hive-mind/2026-09-17-convergence-detection-termination-design.md` |
| Design spec (#1111) | `wsp/specs/issue-1104-hive-mind/2026-09-18-stigmergy-execution-model-design.md` |
| Design spec (#1112) | `wsp/specs/issue-1104-hive-mind/2026-09-18-swarm-execution-model-design.md` |
| Decisions | `wsp/specs/issue-1104-hive-mind/decisions.md` |
| D73 exploration | `wsp/specs/issue-1104-hive-mind/explorations/D73-exploration.md` |
| Implementation plan (#1107) | `wsp/plans/2026-09-16-dynamic-interest-registration.md` |
| Implementation plan (#1108) | `wsp/plans/2026-09-16-agent-discovery-neighbor-awareness.md` |
| Implementation plan (#1109) | `wsp/plans/2026-09-16-local-rule-evaluation.md` |
| Implementation plan (#1110) | `wsp/plans/2026-09-17-convergence-detection-termination.md` |
| Implementation plan (#1111) | `wsp/plans/2026-09-18-stigmergy-execution-model.md` |
| Implementation plan (#1112) | `wsp/plans/2026-09-19-swarm-execution-model.md` |
| Design journal | `wsp/design/JOURNAL.md` |
| Diary entry | `wsp/blog/2026-09-16-mdp01-ants-dont-need-a-dispatcher.md` |
| Queue | `wsp/.plan` |

## Next Session

1. `work continue` to resume #1112 implementation
2. Execute Batch 3: TeamDetector (Task 3) + SwarmProgressTracker (Task 4)
3. Execute Batch 4: Wiring (Task 5) + Integration test (Task 6)
4. `work next` to advance to #1113
