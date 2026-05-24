# Handoff — 2026-05-23

**Head commit (engine):** 7c0ba7b — fix(ledger-test): restore merkle tests with casehub-ledger-memory
**Head commit (workspace):** 5364433 — feat: promote blog entry from issue-330-humantask-scope

## What Changed This Session

Merged PR #334 (full delta from fork to upstream — CI green). Closed engine#330 (HumanTaskTarget scope), #335 (contextChange when-field), #343 (postToChannel 6-param), #341 (CapabilityHealth probe integration). Fixed PR #334 CI: ActorType import (#329), Flyway locations, RoutingCursorStore (#321), PreferenceProvider/WorkloadProvider stubs across resilience/blackboard/work-adapter, test assertions RUNNING→DELEGATED, ContextChangeWhenFilterTest inputSchema, merkle chain test frontier repo.

Updated parent deep-dive for engine (13 modules, CapabilityHealth, scope, 6-param postToChannel) and PLATFORM.md (postToChannel SPI params, deep-dive table). Fixed handover skill: git rev-parse instead of CLAUDE.md scanning (cc-praxis f21bb7b). Published 4 blog entries from other workspaces. Filed parent#47 (workspace path cleanup), engine#344 (DefaultWorkerSpiImplementationsTest beans table).

## Immediate Next Step

Check CI is stable on casehubio/engine main. Then pick next issue from What's Next.

## What's Left

- engine#344 — add NoOpCapabilityHealth to DefaultWorkerSpiImplementationsTest · XS · Low
- engine#322 — SubCase PlanItems never transition beyond PENDING · S · Med
- engine#323 — Verify openChannel idempotency in dispatchCommand · S · Med
- engine#324 — Normalise AgentTest to AssertJ style · XS · Low
- parent#47 — Remove redundant Workspace absolute paths from CLAUDE.mds · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| engine#322 | SubCase PlanItem state machine gap | S | Med | engine#312 surfaced this |
| claudony#122 | Extract correlationId + deadline from COMMAND | S | Med | Unblocked by engine#343 |
| claudony#135 | Remove content-coupling from postToChannel | S | Low | Unblocked by engine#343 |
| engine#274 | BlackboardRegistry hydration from PlanItemStore on restart | M | Med | — |

## Key References

- Blog: `blog/2026-05-23-mdp01-scope-and-the-silent-guard.md`
- PR #334: merged (delta PR)
- PR #346, #348: merged (ledger test fixes)
- Spec: `docs/specs/2026-05-23-humantask-scope-design.md`, `docs/specs/2026-05-23-capability-health-design.md`
