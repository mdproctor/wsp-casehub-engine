# Handoff — 2026-06-22

## What's Done

**engine#550 — add io.casehub.api.spi.mesh (parent#93 extraction)**

New package `io.casehub.api.spi.mesh` in `casehub-engine-api`. Extracts normative agent mesh primitives from claudony so any agent implementation shares canonical definitions.

Types added: `CaseChannelLayout` (SPI + `ChannelSpec` record + `named()` factory), `NormativeChannelLayout` (3-channel: work/observe/oversight per PP-20260604-a7ad99), `SimpleLayout` (2-channel: work/observe, no governance gate), `MeshParticipationStrategy` (SPI + `MeshParticipation` enum + `named()` factory), `ActiveParticipationStrategy`, `ReactiveParticipationStrategy`, `SilentParticipationStrategy`.

46 new tests green. Code review fixes: null guards on `named(null)` → IAE, contract tests cover both layouts, `containsExactlyInAnyOrder` for Set assertions.

## Immediate Next Step

Pick up new work. Branch stamped, merged to main, issue closed.

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #543 | Migrate Worker primitives to casehub-worker-api | L | High | Major refactoring |

*Updated: #554, #555 closed — removed from backlog.*
