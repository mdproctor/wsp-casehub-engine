# Handoff — 2026-05-26

**Head commit (engine):** 7333fb0 — fix(#367,#350): CaseStartedEventHandler blocking thread + openclaw dispatch
**Head commit (workspace):** f288dcf — docs: update handoff (parent session, 2026-05-26)

## What Changed This Session

**engine#349 complete — PR#366 open, awaiting upstream merge.**
Five signal gaps closed: Qhorus human message bridge (`QhorusMessageSignalBridge`), WAITING case signal support (LoopControl dedup refactor + guard relaxed to `RUNNING||WAITING`), ESCALATED mapping fix (engine#338 closed), M-of-N group lifecycle observer (engine#339 closed), `CaseChannel` naming constant.

**engine#367 + #350 also on main** (parent session, 2026-05-26) — `@ConsumeEvent(blocking=true)` on `CaseStartedEventHandler`; openclaw in maven.yml. Needs its own PR (branch `issue-367-350-332-engine-xs-batch`).

## Immediate Next Step

Merge PR#366 (casehubio/engine#366) — clean 2-commit history, all tests pass.

## What's Left

- engine PR#366 — open, awaiting review/merge · XS · Low
- PR needed for `issue-367-350-332-engine-xs-batch` (parent session) · XS · Low
- engine#274 — BlackboardRegistry hydration from PlanItemStore on restart · M · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| claudony#113 | CaseEngineRoundTripTest: call startCase() not CONTEXT_CHANGED | S | Low | Unblocked by engine#367 |
| claudony#122 | Extract correlationId + deadline from COMMAND | S | Med | Unblocked by engine#343 |
| claudony#135 | Remove content-coupling from postToChannel | S | Low | Unblocked by engine#343 |
| claudony#139 | Use CaseChannel.channelName() in ClaudonyReactiveCaseChannelProvider | XS | Low | Filed this session |
| engine#274 | BlackboardRegistry hydration from PlanItemStore on restart | M | Med | — |

## Key References

- PR open: casehubio/engine#366 (engine#349 — signal bridge, 2 commits)
- Garden: GE-20260526-fa0e3e — `getPlanItemByBindingName` silently excludes terminal PlanItems
- Protocol: PP-20260526-case-channel-message-signal — `channelMessage` context path
- Blog: `blog/2026-05-26-mdp01-guard-that-did-too-much.md`
- Parent#75 filed — engine deep-dive doc sync for #349
- Branch closed: `issue-349-case-signal-sink` (deletion due 2026-06-09)
