# Handoff — Hive Mind Epic

**Branch:** `issue-1104-hive-mind`
**Epic:** casehubio/engine#1104
**Slot:** 197
**Date:** 2026-09-20

## What This Is

Self-organizing agent coordination patterns for CaseHub — from rule-based stigmergy through LLM-enhanced swarm intelligence to fully autonomous self-evolution.

## What Happened This Session

Wrote specs and implemented #1114 (Autonomous self-improvement — Engine Foundation). The previous session completed brainstorming and research; this session took those decisions (D92–D105) through specs, plan, and full implementation.

### #1114 — Autonomous Self-Improvement (IMPLEMENTED)

**Specs written:**
- Parent vision spec: `wsp/specs/issue-1104-hive-mind/2026-09-20-cognitive-self-improvement-vision.md` — north-star across 5 epics, capability evolution protocol, design principles
- Child spec (Epic 1): `wsp/specs/issue-1104-hive-mind/2026-09-20-autonomous-self-improvement-engine-foundation.md` — detailed engineering spec for engine foundation

Key user directive incorporated: neocortex cognitive capabilities will evolve during implementation — spec references SPIs not implementations, includes a Capability Evolution Protocol.

**Implementation plan:** `wsp/plans/2026-09-20-autonomous-self-improvement.md` — 6 batches, 11 tasks. Plan review caught two compilation-blocking issues (Signal record has no properties map; SignalRegistry.deposit() signature mismatch) — fixed by introducing `ImprovementSignalContext` companion registry.

**Implementation complete — 37 tests, 0 failures:**

| Component | Module | What it does |
|-----------|--------|-------------|
| `ImprovementBudget` | api | 7-field budget record, conservative defaults |
| `ImprovementConfig` | api | Signal namespace, consensus threshold, enabled categories |
| `ImprovementRequest` / `ImprovementOutcome` / `IntrospectionResult` | api | Data records |
| `StandardGoalKind.SELF_IMPROVEMENT` | api | New goal kind → COMPLETED terminal |
| 3 new `CaseHubEventType` values | api | IMPROVEMENT_GOAL_FORMED, BUDGET_DENIED, OUTCOME |
| `StigmergyConfig.improvement` | api | New field, backward-compat 3-arg constructor |
| `ImprovementBudgetEnforcer` | runtime-core | Structural deny-list + 6 budget check layers |
| `ImprovementSignalContext` | runtime-core | Companion registry: signal name → metadata |
| `ImprovementGoalFormationStrategy` | runtime-core | Consensus detection → budget-gated goal proposals |
| `ImprovementOutcomeRecorder` | runtime-core | EventLog layer |
| `ImprovementSignalProjector` | runtime-core | Signal projection layer |
| `ImprovementCbrProjector` | runtime-core | CBR trace layer (log-only, neocortex wires storage) |
| `ImprovementOutcomeEventCapture` | runtime-core | CDI observer composing all 3 layers |
| `ImprovementIntegrationWorker` | runtime-core | Review hard gate (EventLog pre-flight check) |
| 5 operational workers | runtime-core | Dependency, lint, coverage, CI, recipe (skeletons) |
| Convergence pipeline wiring | runtime-core | Improvement detection in `CaseContextChangedEventHandler` |
| `self-improvement.yaml` | runtime | Case template with conditional bindings |

**Design decision during implementation:** `Signal` record has no metadata properties. Solved with `ImprovementSignalContext` — a companion registry that maps signal names to `ImprovementRequest` metadata. Signals carry consensus; the context registry carries details. Same separation as SwarmProvisioner (signal says "need capacity", provisioner has the details).

### Known issues (pre-existing, unchanged)

- API module checkstyle violations (pre-existing). Build passes with `-Dcheckstyle.skip=true`.
- 5 compilation errors in engine-support-core (pre-existing).
- Build command: `/opt/homebrew/bin/mvn install -pl api,schema,codegen,common-core,engine-support-core,runtime-core -am -Dcheckstyle.skip=true -Dspotless.check.skip=true -DskipTests`

## Queue (2 remaining)

Active: #1114 — implementation complete, ready for `work next` to advance to #1115

Remaining: #1115 — Continuous evolution loop

## Repos in Slot

| Repo | Path | Branch | Role |
|------|------|--------|------|
| engine | `slots/197/engine` | `issue-1104-hive-mind` | Primary |
| blocks | `slots/197/blocks` | `main` | Cognitive stack source |
| eidos | `slots/197/eidos` | `main` | Synced |
| qhorus | `slots/197/qhorus` | `main` | Synced |

## Artifacts

| Artifact | Path |
|----------|------|
| Vision spec | `wsp/specs/issue-1104-hive-mind/2026-09-20-cognitive-self-improvement-vision.md` |
| Engine foundation spec | `wsp/specs/issue-1104-hive-mind/2026-09-20-autonomous-self-improvement-engine-foundation.md` |
| Implementation plan | `wsp/plans/2026-09-20-autonomous-self-improvement.md` |
| Research document | `wsp/specs/issue-1104-hive-mind/2026-09-20-cognitive-self-improvement-research.md` |
| Adversarial review | `wsp/specs/issue-1104-hive-mind/2026-09-20-adversarial-review.md` |
| Decisions (D1–D105) | `wsp/specs/issue-1104-hive-mind/decisions.md` |
| Cognitive stack inventory | `wsp/specs/issue-1104-hive-mind/2026-09-20-cognitive-stack-inventory.md` |
| Memory stack inventory | `wsp/specs/issue-1104-hive-mind/2026-09-20-memory-stack-inventory.md` |
| All prior specs (#1105–#1113) | `wsp/specs/issue-1104-hive-mind/*.md` |
| Queue | `wsp/.plan` |

## Next Session — How to Proceed

1. **`work next`** — advances queue from #1114 to #1115 (closes #1114 on GitHub)
2. **#1115 brainstorming** — continuous evolution loop: standing directive, outcome→detection feedback, concurrent improvement prioritisation, data autophagy prevention, growth direction, rollback on regression
3. #1115 builds on #1114's foundation — all the SPI extension points (§9 of the engine foundation spec) are designed for this

### Design principles to carry forward

- **Head in the clouds, feet on the ground.** Configurable, toggleable, measurable.
- **Capability Evolution Protocol.** Before implementing cognitive integration points, check current blocks/neocortex code — new capabilities may have landed.
- **SPIs, not implementations.** The engine consumes the cognitive stack through contracts. Implementations evolve independently.
