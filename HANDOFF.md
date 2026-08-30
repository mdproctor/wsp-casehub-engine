# HANDOFF — casehub-engine

## Last Session (2026-08-30)

Governed yield reconciliation (#994) — complete. The v2 branch was abandoned; main received parallel work (#995, #999, #1000) that partially covered it. Filed 5 focused issues for remaining gaps and executed all 5 as small branches from main across 3 repos.

## Completed This Session

| Issue | Repo | SHA | What |
|-------|------|-----|------|
| #1009 | engine | `db23e878` | CallerConfig sealed type (Human/Llm/A2A/Any), typed Evidence, CallerIdentity, enriched VerificationContext/EscalationContext |
| #1010 | engine | `c45ab268` | JudgmentPayload sealed (BindingPayload/GatePayload), JudgmentRequest, CloudEventJudgmentScheduler, CallerRefParser.JudgmentRef |
| #1011 | engine | `cf3c03ea` | Escalate(CallerConfig, reason) replaces RouteHigher, DefaultJudgmentEscalator heuristic |
| blocks#219 | blocks | `55a11b9` | JudgmentPhase SPI, JudgmentContext, JudgmentDecision sealed type, ExecutionModel.judgment field |
| qhorus#422 | qhorus | `f7753a47` | MessageType.JUDGMENT — commissive speech act for governed yield |

## Slot-Local M2 Fix

The slot-local `.m2` had stale SNAPSHOTs that blocked compilation:
- `casehub-worker-api` — old `inputSchema()`/`outputSchema()` names, updated to `inputProjection()`/`outputProjection()`
- `casehub-neocortex-cognitive-api` — missing entirely, copied from global m2
- `casehub-neocortex-parent` — old POM missing cognitive-api in dependencyManagement

Pre-existing test compilation breaks on main (both engine and blocks): `AgentExperienceRecorderTest` references `Outcome.importance()` and `StrategyLearningOrchestrator` references `EngagementEvent.sentimentShift()` — both from stale neocortex SNAPSHOTs. Not fixed (out of scope).

## What's Left

All governed yield reconciliation gaps are closed. No remaining work items from this epic.
