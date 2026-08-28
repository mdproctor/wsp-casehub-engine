# CbrHumanTaskRoutingStrategy Design

Refs: engine#754 (parent: engine#797)

## Context

engine#741 shipped the `HumanTaskRoutingStrategy` SPI: interface, sealed result type,
`ExperienceAnalyser` generalisation with predicate overload, retention changes for
humanTask traces (`bindingName` as the matching key, nullable `capabilityName`), and
handler integration in `CaseContextChangedEventHandler.publishHumanTaskSchedule()`.

The `NoOpHumanTaskRoutingStrategy` (`@DefaultBean`, id `"default"`) returns `Unchanged`.

This spec covers the first concrete implementation: a CBR-based strategy that scores
candidate users using plan trace history.

## Placement

`runtime/src/main/java/io/casehub/engine/internal/routing/CbrHumanTaskRoutingStrategy.java`

**Why runtime, not blocks:**

- `HumanTaskRoutingStrategy` is an engine SPI resolved by `EngineStrategyResolver` and
  called from `CaseContextChangedEventHandler`. It is engine planning infrastructure.
- The strategy only depends on engine-api types (`ExperienceAnalyser`,
  `HumanTaskRoutingStrategy`, `HumanTaskRoutingContext`, `HumanTaskCandidates`,
  `HumanTaskRoutingResult`, `RoutingOutcome`). No eidos, ledger, or trust deps.
- `CbrAgentRoutingStrategy` is in blocks because it composes trust (ledger) + graph
  (eidos) + CBR — dependency chains that pull it out of engine. The humanTask
  strategy has no such pull.
- blocks#60 draws the line: engine owns planning/routing infrastructure, blocks owns
  problem-solving techniques. A routing strategy is engine infrastructure.
- Runtime already hosts the CBR pipeline: `CbrRetrievalService` (produces experiences),
  `EngineStrategyResolver` (discovers strategies), `NoOpHumanTaskRoutingStrategy`.
- The shared CBR scoring logic is in `ExperienceAnalyser` (engine-api) — no duplication
  between agent and humanTask strategies.

## Prerequisite: RoutingOutcome.DECLINED

`CbrCaseRetainObserver.OUTCOME_MAP` maps `TaskStatus.REJECTED → "DECLINED"` when
storing plan traces. But `RoutingOutcome` only has four values (SUCCESS, FAILURE,
GATE_REJECTED, GATE_EXPIRED). Steps with DECLINED outcomes hit `IllegalArgumentException`
in `ExperienceAnalyser.workerSuccessRates()`, are caught, and silently skipped.

For humanTasks this is a data integrity gap: DECLINED is the primary negative signal.
A human who declines 9 tasks but completes 1 would score 1.0 (perfect) because only
the SUCCESS is counted.

**Fix (part of engine#754):**

1. Add `DECLINED` to `RoutingOutcome`:
   ```java
   public enum RoutingOutcome {
     SUCCESS, FAILURE, GATE_REJECTED, GATE_EXPIRED, DECLINED
   }
   ```
2. Add `DECLINED → 0.0` to `ExperienceAnalyser.DEFAULT_OUTCOME_WEIGHTS`.

DECLINED scores 0.0 — same weight as FAILURE. From a routing perspective, a declined
task is at least as negative as a failed one: the human explicitly refused the work.

CANCELLED and OBSOLETE remain unmapped in `RoutingOutcome`. These carry no worker
signal (external cancellation, plan obsolescence) and are correctly skipped.

## Design

### Class

```java
@ApplicationScoped
@Unremovable
public class CbrHumanTaskRoutingStrategy implements HumanTaskRoutingStrategy {

  @Override
  public String id() { return "cbr"; }

  @Override
  public HumanTaskRoutingResult select(
      HumanTaskRoutingContext context, HumanTaskCandidates candidates) { ... }
}
```

No injected dependencies — stateless CDI bean. `@Unremovable` prevents Arc from
pruning the bean during build-time optimization, since it is only consumed via
`Instance<HumanTaskRoutingStrategy>` in `EngineStrategyResolver`.

### Algorithm

1. If `context.experiences()` is empty, return `Unchanged`.
2. Extract eligible user IDs from `candidates.users()`.
3. If no eligible users, return `Unchanged`.
4. Call `ExperienceAnalyser.workerSuccessRates(experiences, eligibleUsers,
   step -> context.bindingName().equals(step.bindingName()),
   ExperienceAnalyser.DEFAULT_OUTCOME_WEIGHTS)`.
5. If scores map is empty (no matching plan trace data), return `Unchanged`.
6. Return `Enriched(candidates.groups(), candidates.users(), scores)`.

**Identity invariant:** the scoring in step 4 depends on
`eligibleWorkerIds.contains(step.workerName())` in `ExperienceAnalyser`. This
assumes that `candidateUsers` (from `CandidateSetStrategy` evaluation) and
`executorName` (from `PlanItemRecord`, set via `ExecutorRef.name()` on task
completion) use the same identifier format. Both paths share the platform's
identity namespace — `ExecutorRef.name()` records the same user ID that candidate
resolution produces. No normalization is needed.

### Matching key: bindingName

Agent routing matches plan trace steps by `capabilityName`. HumanTask routing matches
by `bindingName`. This is because humanTask traces have null `capabilityName` —
humanTasks are identified by their binding name, not a capability.

The `ExperienceAnalyser` predicate overload (added in #741) supports this:
`step -> bindingName.equals(step.bindingName())`.

### Result semantics

- `Unchanged`: no CBR data available, or no matching trace data. Candidates pass through
  to `HumanTaskScheduleEvent` with empty scores. This is the safe default — human tasks
  always have recipients.
- `Enriched`: groups and users pass through unchanged (the strategy enriches, not filters).
  `candidateScores` keys are from `candidateUsers` only — group scoring requires group
  membership resolution (engine#757).
- `Escalated`: this strategy never returns `Escalated`. That variant exists for future
  strategies (e.g. the constraint-based strategy, engine#755) that may need to escalate
  when constraints are unsatisfiable.

Unlike `CbrAgentRoutingStrategy` which returns `Unresolvable` when it cannot select,
this strategy never blocks dispatch. Human tasks always proceed.

### Outcome weights

Uses `ExperienceAnalyser.DEFAULT_OUTCOME_WEIGHTS` directly (updated with DECLINED):
- SUCCESS: 1.0
- GATE_EXPIRED: 0.5
- GATE_REJECTED: 0.25
- FAILURE: 0.0
- DECLINED: 0.0

For humanTask traces, the effective outcomes are SUCCESS, FAILURE, and DECLINED.
GATE_EXPIRED and GATE_REJECTED never appear in humanTask traces (no oversight gates)
but are harmless in the map — unused keys cost nothing.

No configurable weights SPI. Blocks' `CbrOutcomeWeights` is an SPI in the blocks
module for `CbrAgentRoutingStrategy`. The humanTask strategy lives in engine and uses
the shared `DEFAULT_OUTCOME_WEIGHTS` directly. This is intentional: agent and humanTask
routing have different SPI surfaces. Agent routing composes trust + graph + CBR in
blocks with a richer configurability model. HumanTask routing is engine infrastructure
with a minimal, focused API. A `HumanTaskCbrOutcomeWeights` SPI can be added if a
domain needs custom weights — the extension point is trivial and backward-compatible.

### Activation

Case definitions activate via YAML:

```yaml
humanTaskRouting: "cbr"
```

`EngineStrategyResolver` resolves by ID. When `humanTaskRouting` is null (not set),
the `@DefaultBean` `NoOpHumanTaskRoutingStrategy` (id `"default"`) is used.

### What this strategy does NOT do

- No trust-based filtering (humans, not AI agents)
- No graph query fallback (no agent graph for humans)
- No signal assembly (engine#754 scope — signals are an agent routing concept)
- No group scoring (engine#757 — requires group membership resolution)
- No candidate filtering (enrichment only — groups and users pass through)

## Tests

### Prerequisite tests

`api/src/test/java/io/casehub/api/spi/routing/ExperienceAnalyserTest.java` — extend
existing test class with DECLINED coverage:

| Test | Assertion |
|------|-----------|
| `declinedOutcomeContributesToEvidenceMass` | DECLINED steps (weight 0.0) dilute the score — a worker with 1 SUCCESS and 1 DECLINED scores 0.5, not 1.0 |
| `frequentDeclinesProduceLowScore` | 9 DECLINED + 1 SUCCESS → score ≈ 0.1 (the spec's motivating scenario) |

### Strategy tests

`runtime/src/test/java/io/casehub/engine/internal/routing/CbrHumanTaskRoutingStrategyTest.java`

Plain JUnit 5 + AssertJ. No `@QuarkusTest` needed — no CDI injection.

| Test | Assertion |
|------|-----------|
| `idIsCbr` | `id()` returns `"cbr"` |
| `emptyExperiencesReturnsUnchanged` | No CBR data → `Unchanged` |
| `emptyUsersReturnsUnchanged` | No candidate users → `Unchanged` |
| `scoresUsersByBindingName` | Scores computed using `bindingName` match |
| `enrichesUsersWithSuccessRateScores` | All candidate scores present in `candidateScores` |
| `ignoresUsersNotInCandidateSet` | Scores only contain eligible user IDs |
| `ignoresStepsWithDifferentBindingName` | Steps for other bindings excluded |
| `addedStepsExcluded` | ADDED adaptation steps skipped |
| `substitutedStepsExcluded` | SUBSTITUTED adaptation steps skipped |
| `similarityWeightingApplied` | High-similarity experiences count more |
| `groupsPassThroughUnchanged` | `Enriched.candidateGroups()` equals input groups |
| `noMatchingTraceDataReturnsUnchanged` | Experiences exist but no steps match → `Unchanged` |

## Downstream impact

- **engine-api changes (§Prerequisite):** `RoutingOutcome` gains a `DECLINED` value
  and `ExperienceAnalyser.DEFAULT_OUTCOME_WEIGHTS` gains a `DECLINED → 0.0` entry.
  Both are additive — no existing code breaks. Callers that construct their own weights
  map are unaffected (the new enum value only matters if they choose to include it).
- No engine wiring changes. `EngineStrategyResolver` already discovers
  `HumanTaskRoutingStrategy` beans.
- No handler changes. `publishHumanTaskSchedule()` already threads strategy
  results to `HumanTaskScheduleEvent`.
- No CLAUDE.md changes needed for the strategy itself — the SPI section
  already documents `HumanTaskRoutingStrategy`.

## Future work

- engine#755: Constraint-based `HumanTaskRoutingStrategy` (separate strategy, id TBD)
- engine#757: Group scoring via group membership resolution
- engine#756: Work repo consumption of experiences and scores from `HumanTaskScheduleEvent`
