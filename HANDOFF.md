# HANDOFF — casehub-engine

## Last Session

Completed all 4 batches of the evolution conductor UI integration (#1180, devtown#230). 64 Java tests green. Launched the server with Playwright and verified the Evolution tab renders in the dashboard tab bar. Then explored a design pivot for how evolution should be presented.

**Batch 2 — Filtering SPIs (27 tests):**
- DevtownConflictStrategy, DevtownDenyPatternProvider, DevtownRegressionEvaluator, DevtownProposalSource

**Batch 3 — API + Bootstrap (15 tests):**
- evolution.yaml case template (L0_INERT, all gates GATED, maxConcurrent 1)
- DevtownEvolutionCaseHub, DevtownEvolutionBootstrap, DevtownEvolutionCaseResolver
- DevtownEvolutionEnricher, DevtownEvolutionApi facade, DevtownEvolutionResource (15 REST endpoints)
- View records: DevtownEvolutionStateSnapshot, DevtownImprovementStreamView, GateResolutionRequest

**Batch 4 — Frontend:**
- evolution.ts view + index.ts wiring — Evolution tab between System and Definitions
- blocks-ui-evolution-workbench import commented out (package not yet packed)

All files in devtown `app` module at `io.casehub.devtown.app.evolution`. 64 total evolution tests green.

## Design Pivot — Evolution as Separate App

During server launch and UI review, we explored whether evolution should be a tab in the review dashboard or its own experience. Conclusion:

**Two peer apps in the devtown repo, one Quarkus backend:**
- **DevTown Review** — operational. Day-to-day PR flow. The existing 8-tab dashboard.
- **DevTown Evolution** — strategic. Conductor steering: inbox, streams, health, configuration.

**Key design decisions:**
1. Evolution is a separate page/app, not a tab in review. Different audiences, different cadence.
2. Review doesn't need evolution widgets — it stays clean as an operational tool.
3. Evolution pulls review context in via contextual panels (reviewer detail, PR history, routing state) — not the other way around.
4. When a PR is conductor-driven, the review app shows a small origin indicator linking back to the evolution stream. That's the only touch point.
5. The dependency is one-way: evolution depends on review (reads its event log), review is self-contained.
6. Same Quarkus backend, same database — the split is frontend routing, not JVM separation.

**This means Batch 3+4 need restructuring:** The API facade and REST resource stay in `app/` but the Evolution tab wiring in index.ts should be replaced with a separate page route. New issue needed.

## Open Items

1. **blocks-ui packaging:** `@casehubio/blocks-ui-evolution-workbench` + `evolution-config` need to be packed from blocks-ui via `pack-all.sh` and added to devtown's `.casehub-packages/tarballs/` + `package.json`
2. **Bootstrap fix:** `DevtownEvolutionBootstrap` startup observer didn't create the case — need to debug (the YAML registered but `startCase()` may have failed silently)
3. **Platform dependency:** `casehub-platform-agent-session-core` is an optional dep of `agent-router` — devtown needs it explicitly (added to pom.xml during this session but not committed)
4. **App split issue:** Create new issue for restructuring evolution as a separate page/app within devtown

## Immediate Next Step

Create a new issue for the evolution app split. Then: fix the bootstrap, pack blocks-ui, verify the full evolution workbench renders.

## References

- `wsp/specs/issue-1180-evolution-conductor-ui-in-devtown/2026-10-07-evolution-conductor-ui-in-devtown-design.md`
- `wsp/specs/issue-1180-evolution-conductor-ui-in-devtown/decisions.md`
- `wsp/plans/2026-10-07-evolution-conductor-ui-in-devtown.md`
- `wsp/JOURNAL.md`
- `wsp/blog/2026-10-07-mdp01-the-conductor-finds-its-audience.md`
