# Handoff — Hive Mind Epic

**Branch:** `issue-1104-hive-mind`
**Epic:** casehubio/engine#1104
**Slot:** 197
**Date:** 2026-09-16

## What This Is

Self-organizing agent coordination patterns for CaseHub — from rule-based stigmergy through LLM-enhanced swarm intelligence to fully autonomous self-evolution.

## What Happened This Session

Completed #1107 (Dynamic interest registration) and designed + planned #1108 (Agent discovery & neighbor awareness).

### #1107 — Dynamic Interest Registration (DONE)

Full WorkerRuntime restructure into domain-organized coordination facets. Pre-release clean break — flat methods removed, not deprecated.

Key deliverables:
- **WorkerRuntime faceting:** `signals()` returns `SignalSpace`, `interests()` returns `InterestSpace`. Flat methods (`depositSignal`, `perceiveSignals`, `registerObserver`) removed entirely.
- **InterestDeclaration sealed hierarchy:** 5 permits — `KeyThreshold`, `KeyCorrelation`, `TemporalSequence`, `SignalThreshold`, `JqInterest`. Each maps 1:1 to a classical observer. `JqInterest` gated by `ObservationConfig.allowJqInterests` (default true) for compliance.
- **InterestRegistration:** Immutable handle with engine-generated ID. Deregister by ID.
- **InterestLandscape:** Anonymous aggregate view (keyObserverCounts, signalObserverCounts, interestTypeCounts, totalObserverCount). Available on both `InterestSpace.landscape()` and `ObservationContext.interestLandscape()` (8th field).
- **DefaultSignalSpace** extracted from DefaultWorkerRuntime. **DefaultInterestSpace** creates observers from declarations.
- **ObservationRegistry extensions:** `registerObserver()` returns String instanceId (was boolean). `deregisterByInstanceId()`. `computeLandscape()`.
- **ObservationConfig** gains 4th field `allowJqInterests`. YAML: `allowJqInterests:` under `observationConfig:`.
- **Audit event types:** `INTEREST_REGISTERED`, `INTEREST_DEREGISTERED` (publishing deferred with PHEROMONE events).

5 tasks, 2 batches, 99 tests passing across api/common-core/runtime-core.

### #1108 — Agent Discovery & Neighbor Awareness (DESIGNED + PLANNED)

Design spec and implementation plan written. Not yet implemented.

Architecture: `NeighborSpace` is a pure read-only query facade over existing engine registries. No new storage. Proximity is emergent from shared activity.

Key design decisions (D32-D36):
- **NeighborSpace facet** — third WorkerRuntime facet. Four named query methods: `active()`, `withSharedInterests()`, `withSharedSignals()`, `complementary()`.
- **Neighbor record** — agentId + capabilities + status + bindingName + relations. Full identity exposed.
- **NeighborRelation enum** — COACTIVE, SHARED_INTEREST, SHARED_SIGNAL, COMPLEMENTARY.
- **Signal source tracking** — `Signal` gains `Set<String> sources` for multi-depositor tracking.
- **DefaultNeighborSpace** queries PlanItemStore, ObservationRegistry, SignalRegistry, CaseDefinition.

Implementation plan: 4 tasks in 2 batches. Ready for execution.

## Queue (8 remaining)

Active: #1108 — Agent discovery & neighbor awareness

Batch 1 remaining: #1108 (discovery)
Batches 2-6: #1109-#1115 unchanged from initial plan — see `.plan`.

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
| Design spec (#1107) | `wsp/specs/issue-1104-hive-mind/2026-09-16-dynamic-interest-registration-design.md` |
| Design spec (#1108) | `wsp/specs/issue-1104-hive-mind/2026-09-16-agent-discovery-neighbor-awareness-design.md` |
| Decisions | `wsp/specs/issue-1104-hive-mind/decisions.md` |
| Implementation plan (#1107) | `wsp/plans/2026-09-16-dynamic-interest-registration.md` |
| Implementation plan (#1108) | `wsp/plans/2026-09-16-agent-discovery-neighbor-awareness.md` |
| Design journal | `wsp/design/JOURNAL.md` |
| Diary entry | `wsp/blog/2026-09-16-mdp01-ants-dont-need-a-dispatcher.md` |
| Queue | `wsp/.plan` |

## Next Session

1. Execute #1108 implementation plan (4 tasks, 2 batches)
2. Advance queue to #1109 (Local rule evaluation)
3. Continue through the epic
