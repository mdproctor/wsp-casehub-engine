# Handoff — 2026-06-01

**Head commit (engine):** 403dc53 — feat: CaseHub.startCase accepts Object input (6 squashed commits on main)
**Head commit (workspace):** 3d1bd1e — chore: clean up plan originals after attic move
**Both repos on:** main
**casehubio/engine:** ✅ green (manually triggered after force-push — gh workflow run required)

## What Changed This Session

**Merged PR #2 after fixing 4 CI failures in the S/XS batch:**

1. `fix(persistence)`: V1.2.0 migration missing — `persistence-hibernate` uses Flyway+validate; `tenancy_id` on 5 tables needed a migration
2. `fix(runtime)`: `CurrentPrincipal` bean missing in default profile — `DefaultTestPrincipal` only loaded in `persistence-memory` Maven profile; fixed by making it an always-on test dep
3. `fix(test)`: Two `@DefaultBean` implementations of the same SPI → "Ambiguous dependencies"; excluded `casehub-platform` mock beans in blackboard, resilience, work-adapter
4. `fix(test)`: `CaseLifecycleEvent` positional arg shift — adding `tenancyId` as 2nd field silently moved `null` from commandType to tenancyId; fixed constructors in two blackboard tests

**Branch closed and delivered:**
- `issue-408-s-xs-batch` closed, EPIC-CLOSED.md stamped
- 11 commits squashed → 6, pushed to `casehubio/engine` upstream
- CI green on both fork and blessed repo

## Immediate Next Step

Run `/work` to start engine#404 (registry lifecycle design — L·High).

## Cross-Module

**We're blocking** (tenancyId SPI changes require consumer recompile):
- `claudony` — claudony#143 · XS · Low
- `devtown` — devtown#61 · XS · Low
- `aml` — aml#47 + aml#48 (migration reconciliation) · S · Low
- `clinical` — clinical#51 · XS · Low

## What's Left

- engine#405 — @CrossTenant CDI producer · S · Low — **BLOCKED** (needs system-actor principal)
- engine#406 — DB-level RLS · M · High
- engine#411 — NOT NULL enforcement for tenancy_id columns in V1.2.0 · S · Low
- engine#410 — registry lookup root cause (defensive guard in place) · M · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | AI Fusion typed fact space implementation | XL | High | New module — own session; spec at casehubio/parent:docs/specs/2026-06-03-ai-fusion-hybrid-fact-space.md |
| engine#404 | Registry lifecycle: eviction + stateless-on-rest + Quartz restart | L | High | Design-only; groundwork done |
| engine#383 | Oversight response loop | M | Med | Unblocked |
| engine#384 | PlanItem escalation state | M | Med | Unblocked |
| engine#387 | humanTask dynamic candidateGroups | M | Med | — |

## Key References

- Blog: `blog/2026-06-01-mdp01-fixes-a-mystery-and-three-migrations.md`, `blog/2026-06-01-mdp02-what-ci-found-in-tenancy-tests.md`
- Garden entries: GE-20260601-fcf0d9 (@DefaultBean ambiguity), GE-20260601-53763c (gh log truncation), GE-20260601-6170a6 (Java record positional shift), GE-20260601-ad3154 (bytecode parser)
- Protocol: PP-20260601-70e9ea (platform mock exclusion)
- Note: Force-pushes to casehubio/engine don't auto-trigger CI — use `gh workflow run "Build and Publish" --repo casehubio/engine --ref main`
