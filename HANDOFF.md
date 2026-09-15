# Handoff — Hive Mind Epic

**Branch:** `issue-1104-hive-mind`
**Epic:** casehubio/engine#1104
**Slot:** 197
**Date:** 2026-09-16

## What This Is

Self-organizing agent coordination patterns for CaseHub — from rule-based stigmergy through LLM-enhanced swarm intelligence to fully autonomous self-evolution. No competing framework implements true self-organization; this is novel territory.

## Repos in Slot

| Repo | Path | Role |
|------|------|------|
| engine | `slots/197/engine` | Primary — foundation SPIs, execution models, autonomous loop |
| blocks | `slots/197/blocks` | LLM-enhanced observation, swarm coordination |
| eidos | `slots/197/eidos` | Proximity/topology query for agent discovery |
| qhorus | `slots/197/qhorus` | Topic-based broadcast for swarm coordination |

## Queue (15 issues, 6 batches)

### Batch 1 — Foundation SPIs (parallel, no dependencies)
| # | Title | Repo | Scale | Cplx |
|---|-------|------|-------|------|
| 1105 | Environment observation SPI | engine | M | Med |
| 1106 | Signal/pheromone model with decay & reinforcement | engine | M | Med |
| 1108 | Agent discovery & neighbor awareness | engine | M | Med |

### Batch 2 — Dynamic coordination (depends on batch 1)
| # | Title | Repo | Scale | Cplx |
|---|-------|------|-------|------|
| 1107 | Dynamic interest registration | engine | M | High |
| 1109 | Local rule evaluation | engine | L | High |
| 1110 | Convergence detection & termination | engine | M | High |

### Batch 3 — Cross-repo enablers (parallel with batch 2)
| # | Title | Repo | Scale | Cplx |
|---|-------|------|-------|------|
| eidos#178 | Proximity/topology query | eidos | M | Med |
| qhorus#442 | Topic-based broadcast | qhorus | M | Med |

### Batch 4 — Execution models (depends on batches 2-3)
| # | Title | Repo | Scale | Cplx |
|---|-------|------|-------|------|
| 1111 | Stigmergy execution model | engine | L | High |
| 1112 | Swarm execution model | engine | L | High |

### Batch 5 — LLM-enhanced hive mind (depends on batch 4)
| # | Title | Repo | Scale | Cplx |
|---|-------|------|-------|------|
| blocks#284 | LLM-driven observation & local rules | blocks | L | High |
| blocks#285 | LLM swarm coordination with experience replay | blocks | L | High |

### Batch 6 — Autonomous (depends on batch 5)
| # | Title | Repo | Scale | Cplx |
|---|-------|------|-------|------|
| 1113 | Self-provisioning swarm | engine | L | High |
| 1114 | Autonomous self-improvement | engine | XL | High |
| 1115 | Continuous evolution loop | engine | XL | High |

## Research Foundation

Key papers that shaped the design — read these before starting implementation:

1. **"Drop the Hierarchy and Roles"** (arXiv:2603.28990) — minimal scaffolding + role autonomy beats hierarchy. Give agents a mission + protocol + capable model, not pre-assigned roles.
2. **SwarmWorld** (arXiv:2608.26081) — validates engine/LLM split (cognition separated from consequence).
3. **Emergent Collective Memory** (arXiv:2512.10166) — memory-first principle: individual memory prerequisite before environmental traces work. At agent density >0.20, stigmergic coordination dominates.
4. **SwarmExp** (arXiv:2608.30661, EMNLP 2026) — experience extraction/replay = CBR plan traces. CaseHub competitive advantage.
5. **AgentNet** (arXiv:2504.00587) — decentralized via dynamic DAG + RAG memory.
6. **SwarmSys** (arXiv:2510.10047) — Explorer/Worker/Validator cycle with pheromone reinforcement.
7. **Darwin Godel Machine** (Sakana AI) — self-modifying agents. Only works with verifiable outcomes.
8. **Self-Evolving Agents Survey** (arXiv:2507.21046) — data autophagy risk in self-improvement loops.

## Design Principles

1. **Memory-first** — neocortex memory is prerequisite before environmental traces become useful
2. **Decay-first** — pheromone signals must decay; stale state misleads future agents
3. **Verification-first** — self-improvement only works with verifiable outcomes (devtown review gate)
4. **Engine mechanics, blocks intelligence** — rule-based hive mind fully works in engine without LLMs; blocks adds LLM for advanced patterns

## Cross-Repo Verified Status (Code-Verified)

| Repo | Status | Evidence |
|------|--------|----------|
| neocortex | Ready as-is | `Memory` has `principalId` + `sharedWith` for permissions-based visibility. `Subject(type, id)` supports `type="swarm"`. `MemoryQuery.forSubjects()` for multi-subject queries. |
| eidos | One gap | `AgentQuery` filters by exact capability/slot/goal. No proximity or activity-based query. `toBuilder()` enables runtime descriptor updates. |
| ledger | Ready as-is | `TrustScoreSource` has per-actor, per-capability, per-dimension, batch scoring. `AttestorCredibilityPolicy` for credibility assessment. |
| work | Ready as-is | Judgment bindings + `WorkItemSpawnGroup` for M-of-N oversight. |
| qhorus | One gap | `EPHEMERAL` semantic perfect for pheromone hints. `PresenceTracker` for discovery. No topic-based broadcast without pre-enrolled named channels. |

## Existing Engine Infrastructure to Build On

- `CaseContext.onChange(key, listener)` / `onAnyChange(listener)` — per-key change listeners
- `ContextChangeTrigger` — static JQ trigger conditions (additive, not replacement)
- `ScopedWorkerRegistry` — PERSISTENT (mailbox) and REINVOKED (accumulated state) execution modes
- `CaseEvaluationSerializer` — per-case serialization of CONTEXT_CHANGED evaluation
- `Watchdog` conditions — LOOP_DETECTED, ECHO_CHAMBER, CONVERSATION_STALL, CIRCULAR_DELEGATION
- `WatchdogRecoveryBridge` — translates alerts to synthetic WorkerOutcome.Expired
- `GoalBasedCompletion` — goal satisfaction detection
- `CbrRetrievalService` — CBR experience retrieval (SwarmExp foundation)
- `ExperienceAnalyser` — worker success rates from plan traces
- Blocks `ExecutionModel` — composable SPIs (RoutingStrategy, ActivationRule, AggregationStrategy, TerminationCondition)

## What's Next

Start with Batch 1 (foundation SPIs). The three issues are independent — can be worked in parallel or sequentially. First active issue is #1105 (Environment observation SPI).

Open a CLI in `slots/197/engine` and run `work`.
