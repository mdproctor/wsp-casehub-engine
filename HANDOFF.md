# Handoff — Hive Mind Epic

**Branch:** `issue-1104-hive-mind`
**Epic:** casehubio/engine#1104
**Slot:** 197
**Date:** 2026-09-16

## What This Is

Self-organizing agent coordination patterns for CaseHub — from rule-based stigmergy through LLM-enhanced swarm intelligence to fully autonomous self-evolution.

## What Happened This Session

Designed and implemented #1106 (Signal/pheromone model) — the temporal signal model for stigmergic coordination. Full cycle: brainstorming (9 decisions D10-D18, all quick picks), design spec (light review, 10 findings incorporated), implementation plan (5 tasks, 2 batches), execution (all tasks complete, 42 tests passing).

Key design choices: signals live in a dedicated `SignalRegistry` (not CaseContext — avoids feedback loops per #1105 D7); exponential decay computed lazily at read time (`effectiveStrength = strength * e^(-λ * elapsed)`); name-keyed signals with max-reinforcement; `WorkerRuntime.depositSignal()`/`perceiveSignals()` as the worker API; `ObservationContext.signals()` for observer perception; signal expiry detection runs before observer-count guard (R1-04 fix); event types named `PHEROMONE_*` to avoid collision with existing `SIGNAL_*` types (R1-08).

Deferred: EventLog publishing for `PHEROMONE_DEPOSITED`/`PHEROMONE_EXPIRED` (needs event bus plumbing into `DefaultWorkerRuntime`).

Queue advanced to #1107 (Dynamic interest registration).

## Queue (9 remaining)

Active: #1107 — Dynamic interest registration

Batch 1 remaining: #1107 (interests), #1108 (agent discovery)
Batches 2-6: unchanged from initial plan — see `.plan`.

## Repos in Slot

| Repo | Path | Role |
|------|------|------|
| engine | `slots/197/engine` | Primary |
| blocks | `slots/197/blocks` | LLM-enhanced observation (future) |
| eidos | `slots/197/eidos` | Proximity/topology query (future) |
| qhorus | `slots/197/qhorus` | Topic-based broadcast (future) |

## Artifacts

| Artifact | Path |
|----------|------|
| Design spec (#1105) | `wsp/specs/issue-1104-hive-mind/2026-09-16-environment-observation-spi-design.md` |
| Design spec (#1106) | `wsp/specs/issue-1104-hive-mind/2026-09-16-signal-pheromone-model-design.md` |
| Decisions | `wsp/specs/issue-1104-hive-mind/decisions.md` |
| Implementation plan (#1106) | `wsp/plans/2026-09-16-signal-pheromone-model.md` |
| Design journal | `wsp/design/JOURNAL.md` |
| Diary entry | `wsp/blog/2026-09-16-mdp01-ants-dont-need-a-dispatcher.md` |
| Queue | `wsp/.plan` |

## Open Question

Whether the signal model should live in a separate module (e.g. `casehub-engine-ras`) rather than engine-api/common-core. Current placement follows the `ObservationRegistry` precedent — foundation coordination primitives in the core modules. To be discussed.

## Research Foundation

*Unchanged — see git show HEAD~15:HANDOFF.md §Research Foundation*
