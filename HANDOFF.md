# Handoff — Hive Mind Epic

**Branch:** `issue-1104-hive-mind`
**Epic:** casehubio/engine#1104
**Slot:** 197
**Date:** 2026-09-20

## What This Is

Self-organizing agent coordination patterns for CaseHub — from rule-based stigmergy through LLM-enhanced swarm intelligence to fully autonomous self-evolution.

## What Happened This Session

Advanced queue from #1114 to #1115. Completed full brainstorming and planning cycle for #1115 (Continuous Evolution Loop). No implementation code this session — design only.

### #1115 — Continuous Evolution Loop (DESIGNED, READY FOR IMPLEMENTATION)

**Decisions captured:** D106–D115 (8 new + 2 surfaced by review)

| # | Decision | Type |
|---|----------|------|
| D106 | Hybrid trigger — event-driven + timer backstop | Standing directive |
| D107 | Confidence-tiered rollback — proportional response | Rollback on regression |
| D108 | Health score + circuit breaker (CLOSED/OPEN/HALF_OPEN) | Data autophagy prevention |
| D109 | Emergent capability taxonomy + selection strategies | Growth direction |
| D110 | Tiered research cadence (horizon scan / scouting / deep dive) | Research methodology |
| D111 | Research corpus (Living Systematic Review) + HIL queue | Persistent research storage |
| D112 | Continuous improvement methodology — PRISMA, Tech Radar, Wardley | End-to-end structured process |
| D113 | Concurrent improvement conflict avoidance — serialize overlapping paths | Scheduling |
| D114 | Signal namespace convention (shared, naming convention) | Surfaced by decision review |
| D115 | CaseHubEventType flat enum growth acknowledged | Surfaced by decision review |

**Key design insight:** The research-to-implementation pipeline uses established methodologies (PRISMA protocol, Technology Radar, Wardley Mapping, Horizon Scanning) rather than custom terminology. A methodology document (`2026-09-20-continuous-improvement-methodology.md`) governs the structured process from scanning to implementation — mitigating the risk that LLMs produce impressive analysis without actionable output.

**Reviews completed:**
- Decision review (standard, 3 rounds, $29) — 8 verified fixes, 2 new decisions (D114, D115)
- Spec review (standard, 3 rounds, $47) — 19 verified fixes, 2 deferred (minor wiring gaps)

**Spec:** `wsp/specs/issue-1104-hive-mind/2026-09-20-continuous-evolution-loop-design.md` — 1135 lines, 16 sections
**Plan:** `wsp/plans/2026-09-20-continuous-evolution-loop.md` — 1220 lines, 8 batches, 15 tasks
**Methodology:** `wsp/specs/issue-1104-hive-mind/2026-09-20-continuous-improvement-methodology.md`

### Prior work (#1105–#1114, previous sessions)

All implemented. See git log for details. #1114 (Autonomous Self-Improvement Engine Foundation) provides the base that #1115 builds on.

### Known issues (pre-existing, unchanged)

- API module checkstyle violations (pre-existing). Build passes with `-Dcheckstyle.skip=true`.
- 5 compilation errors in engine-support-core (pre-existing).
- Build command: `/opt/homebrew/bin/mvn install -pl api,schema,codegen,common-core,engine-support-core,runtime-core -am -Dcheckstyle.skip=true -Dspotless.check.skip=true -DskipTests`

## Queue (1 remaining)

Active: #1115 — design complete, implementation plan ready, 15 tasks across 8 batches

## Repos in Slot

| Repo | Path | Branch | Role |
|------|------|--------|------|
| engine | `slots/197/engine` | `issue-1104-hive-mind` | Primary |
| blocks | `slots/197/blocks` | `main` | Cognitive stack source |
| eidos | `slots/197/eidos` | `main` | Synced |
| qhorus | `slots/197/qhorus` | `main` | Synced |

## Artifacts

| Artifact | Path |
|----------|------|
| Vision spec | `wsp/specs/issue-1104-hive-mind/2026-09-20-cognitive-self-improvement-vision.md` |
| Engine foundation spec (#1114) | `wsp/specs/issue-1104-hive-mind/2026-09-20-autonomous-self-improvement-engine-foundation.md` |
| Continuous evolution spec (#1115) | `wsp/specs/issue-1104-hive-mind/2026-09-20-continuous-evolution-loop-design.md` |
| Improvement methodology | `wsp/specs/issue-1104-hive-mind/2026-09-20-continuous-improvement-methodology.md` |
| Implementation plan (#1115) | `wsp/plans/2026-09-20-continuous-evolution-loop.md` |
| Decisions (D1–D115) | `wsp/specs/issue-1104-hive-mind/decisions.md` |
| All prior specs (#1105–#1114) | `wsp/specs/issue-1104-hive-mind/*.md` |
| Queue | `wsp/.plan` |

## Next Session — How to Proceed

1. **`work continue`** — resumes on #1115
2. **Execute the plan** — `executing-plans` with `wsp/plans/2026-09-20-continuous-evolution-loop.md`
3. Batch 1 (Configuration Records) is the starting point — new API types for RollbackPolicy, HealthPolicy, EvolutionConfig

### Design principles to carry forward

- **Head in the clouds, feet on the ground.** Configurable, toggleable, measurable.
- **Capability Evolution Protocol.** Before implementing cognitive integration points, check current blocks/neocortex code.
- **SPIs, not implementations.** Engine provides rule-based defaults + SPI contracts. Blocks provides LLM-powered implementations.
- **Standard terminology.** Use PRISMA, Technology Radar, Wardley Mapping, Horizon Scanning — not custom names.
- **Methodology is the rails.** The structured process (scan → screen → extract → synthesize → triangulate → prioritize → hypothesize) prevents the system from getting lost in research without reaching implementation.

### Deferred spec review items (fix during implementation)

- R3-01: `RegressionDetector.evaluate()` trigger wiring — need to specify what calls it
- R3-02: `RollbackHistory.record()` missing `target` parameter
