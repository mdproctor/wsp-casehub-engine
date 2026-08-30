# HANDOFF — casehub-engine

## Last Session (2026-08-30)

Governed yield reconciliation (#994). The v2 branch was abandoned — main received parallel work (#995, #999, #1000) that partially covered it. Filed 5 focused issues for remaining gaps and executed the 3 engine issues as small branches from main.

## Completed This Session

| Issue | SHA | What |
|-------|-----|------|
| #1009 | `db23e878` | CallerConfig sealed type (Human/Llm/A2A/Any), typed Evidence (EvidenceType, EvidenceRequirement, Evidence), CallerIdentity, enriched VerificationContext/EscalationContext |
| #1010 | `c45ab268` | JudgmentPayload sealed (BindingPayload/GatePayload), JudgmentRequest, CloudEventJudgmentScheduler, CallerRefParser.JudgmentRef, deprecated ActionGateScheduler/ActionGateScheduleRequest |
| #1011 | `cf3c03ea` | Escalate(CallerConfig, reason) replaces RouteHigher, DefaultJudgmentEscalator (@DefaultBean, heuristic), FaultEscalator demoted |

## Slot-Local M2 Fix

The slot-local `.m2` had stale SNAPSHOTs that blocked compilation:
- `casehub-worker-api` — old `inputSchema()`/`outputSchema()` names, updated to `inputProjection()`/`outputProjection()`
- `casehub-neocortex-cognitive-api` — missing entirely, copied from global m2
- `casehub-neocortex-parent` — old POM missing cognitive-api in dependencyManagement

Pre-existing test compilation break on main: `AgentExperienceRecorderTest` references `Outcome.importance()` which doesn't exist in the installed neocortex SNAPSHOT. Not fixed (out of scope).

## What's Left

| # | Repo | What | Scale | Status |
|---|------|------|-------|--------|
| blocks#219 | blocks | Port judgment types to blocks — adapted to enriched engine types | M | Ready (engine deps landed) |
| qhorus#422 | qhorus | Cherry-pick JUDGMENT message type from abandoned branch | S | Ready |

## What's Next

Blocks#219 and qhorus#422 are independent and can be done in any order. Both repos are on main at origin/main. Engine SNAPSHOTs need to be installed to the slot-local `.m2` before blocks can compile against the new types.
