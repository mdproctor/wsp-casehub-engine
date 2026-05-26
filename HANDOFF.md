# Handoff — 2026-05-26

**Head commit (engine):** 7333fb0 — fix(#367,#350): CaseStartedEventHandler blocking thread + openclaw dispatch
**Head commit (workspace):** 6cb9bb6 — feat: promote blog metadata sync from issue-322-engine-xs-s-batch

## What Changed This Session

*(Previous session 2026-05-25: closed engine#344, confirmed #322/#323/#324 resolved.)*

**Signal sink branch (`issue-349-case-signal-sink`) merged to main** — engine#349 shipped (CaseSignalSink SPI). Unblocks casehubio/work#225, casehubio/qhorus#200.

**Parent session (2026-05-26) made changes on branch `issue-367-350-332-engine-xs-batch`:**
- engine#367 (new): `@ConsumeEvent(blocking = true)` added to `CaseStartedEventHandler.onCaseStarted()` — handler called Quartz JTA store on Vert.x IO thread, causing BlockingOperationNotAllowedException. Now matches SubCaseExecutionHandler and HumanTaskScheduleHandler. Unblocks claudony#113.
- engine#350: openclaw added to downstream dispatch list in maven.yml alongside flow.
- engine#332: closed as already done — blackboard and platform-expression already indexed in work-adapter tests.
- engine#243: closed as already done — all module folder names already short.

## Immediate Next Step

**PR branch `issue-367-350-332-engine-xs-batch`** — 2 files changed (CaseStartedEventHandler + maven.yml), all issues closed.

After merge, proceed to claudony session to update `CaseEngineRoundTripTest` (claudony#113) — now unblocked by the `blocking = true` fix.

## What's Left

- ~~parent#47~~ — tracked in parent session (peer repo — workspace CLAUDE.md changes)
- engine#274 — BlackboardRegistry hydration from PlanItemStore on restart · M · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| claudony#113 | CaseEngineRoundTripTest: call startCase() not CONTEXT_CHANGED | S | Low | Now unblocked by engine#367 |
| claudony#122 | Extract correlationId + deadline from COMMAND | S | Med | Unblocked by engine#343 |
| claudony#135 | Remove content-coupling from postToChannel | S | Low | Unblocked by engine#343 |
| engine#274 | BlackboardRegistry hydration from PlanItemStore on restart | M | Med | — |

## Key References

- Open branch: `issue-367-350-332-engine-xs-batch` (project repo — needs PR)
- Protocols: PP-20260526-6d39e5 (opaque cross-module identifiers — new this session)
- Blog: `blog/2026-05-23-mdp01-scope-and-the-silent-guard.md`
- Branch closed: `issue-322-engine-xs-s-batch` (deletion due 2026-06-08)
