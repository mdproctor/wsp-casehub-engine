# HANDOFF — casehub-engine (Slot 160)

## Session Summary (2026-08-30)

Governed yield reconciliation (#994). The v2 branch was abandoned because main received parallel work (#995, #999, #1000) that partially covered it. This session filed and executed 5 focused issues for the first round of gaps, then did a second gap analysis that revealed the v2 branch had significantly richer types than what we ported.

## Phase 1 — Completed (5 issues, all closed)

| Issue | Repo | SHA | What |
|-------|------|-----|------|
| engine#1009 | engine | `db23e878` | CallerConfig sealed (Human/Llm/A2A/Any), typed Evidence, CallerIdentity, enriched VerificationContext/EscalationContext |
| engine#1010 | engine | `c45ab268` | JudgmentPayload sealed (BindingPayload/GatePayload), JudgmentRequest, CloudEventJudgmentScheduler, CallerRefParser.JudgmentRef |
| engine#1011 | engine | `cf3c03ea` | Escalate(CallerConfig, reason) replaces RouteHigher, DefaultJudgmentEscalator heuristic |
| blocks#219 | blocks | `55a11b9` | JudgmentPhase SPI, JudgmentContext, JudgmentDecision sealed, ExecutionModel.judgment field |
| qhorus#422 | qhorus | `f7753a47` | MessageType.JUDGMENT — commissive speech act for governed yield |

All landed on fork main (not upstream). Repos in the slot are on main at origin/main.

## Phase 2 — Completed (engine-side)

Engine issues #1012 and #1013 landed on main (2026-08-31). The governed yield engine foundation is complete.

| Issue | SHA | What |
|-------|-----|------|
| engine#1012 | `1b886952` | Enriched CallerConfig/CallerIdentity/Evidence, JudgmentTarget.maxEscalationAttempts, deprecated HumanTaskTarget/HumanTaskScheduler |
| engine#1013 | landed via #1000 | DagNode.judgment field, DagNodeSnapshot.hasJudgment, DagDriver JUDGING state |

### Remaining blocks-side work (not started):

**1. blocks#220 — Wire judgment loop in AbstractExecutionDriver** (S / Low)
   - Phase 3.5 judgment integration in the execution loop
   - **Blocked by:** nothing (engine#1012 dependency resolved)

**2. blocks#221 — Port engine-adapter judgment types** (M / Med)
   - PatternJudgmentConfig, LlmJudgmentPhase, LlmJudgmentScheduler, verification strategies
   - **Blocked by:** blocks#220

## Slot-Local M2 Issues

The slot-local `.m2` had stale SNAPSHOTs that were fixed this session:
- `casehub-worker-api` — copied from global m2 (old `inputSchema`/`outputSchema` → new `inputProjection`/`outputProjection`)
- `casehub-neocortex-cognitive-api` — was missing entirely, copied from global m2
- `casehub-neocortex-parent` — old POM lacked cognitive-api in dependencyManagement

**Pre-existing test compilation breaks (NOT fixed — out of scope):**
- engine: `AgentExperienceRecorderTest` references `Outcome.importance()` — stale neocortex SNAPSHOT
- blocks: `StrategyLearningOrchestrator` references `EngagementEvent.sentimentShift()` — same cause
- These block `mvn test` on the runtime/blocks modules. Workaround: temporarily move broken file to `/tmp/`, run specific tests, restore

## V2 Branch Reference

The abandoned branches contain the source material for cherry-picking:
- **engine:** `issue-994-governed-yield-v2` (9 commits) — CallerConfig with full fields, Evidence with ref, JudgmentTarget.maxEscalationAttempts, DagNode.judgment, CLAUDE.md updates
- **blocks:** `issue-994-governed-yield` (5 judgment commits at tip) — JudgmentPhase/Context/Decision, ExecutionModel.judgment, LlmJudgmentPhase, PatternJudgmentConfig, YAML parsing, handler wiring, 4 test classes
- **qhorus:** `issue-994-governed-yield` (1 judgment commit) — already cherry-picked as #422

**Do not delete these branches** — they're the cherry-pick source for Phase 2.

## What's Next

Engine governed yield work is complete (#1009–#1013 all landed). Remaining governed yield is blocks-side: blocks#220 (S, driver wiring) then blocks#221 (M, engine-adapter types).

**Engine priorities (updated 2026-08-31):**

| Priority | Issue | Scale | What |
|----------|-------|-------|------|
| 1 | #1015 | L / High | Adopt yaml-core record pattern — eliminate hand-coded deserializers |
| 2 | #984 | L / Med | Standalone YAML examples for all execution models |
| 3 | #987 | M / High | YAML HTN decomposition tree |
| — | #959 | S / Low | Reasoning support for persistent workers |
| — | #958 | S / Low | Independent importance weights for worker-reasoning |
| — | #867 | S / Low | Read identity from PropagationContext |

#1015 is the highest-value engine work — structural improvement that reduces maintenance burden across all future YAML features. #984 and #987 are remaining #978 (YAML DSL) children.
