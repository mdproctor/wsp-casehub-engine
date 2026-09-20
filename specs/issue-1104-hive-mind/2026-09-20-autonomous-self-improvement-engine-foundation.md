# Autonomous Self-Improvement: Engine Foundation — Design Spec

**Issue:** casehubio/engine#1114
**Epic:** casehubio/engine#1104 (Hive Mind)
**Parent spec:** `2026-09-20-cognitive-self-improvement-vision.md`
**Date:** 2026-09-20
**Decisions:** D92–D105
**Cognitive stack snapshot:** 2026-09-20 (blocks `9186d51`, neocortex — check before implementing)

## Problem

The hive mind can observe, signal, coordinate, provision, and converge — but it cannot improve itself. When the platform has a stale dependency, a lint violation, a coverage gap, or a CI failure, no agent acts on it. The infrastructure to sense these problems exists (signals, observations), and the infrastructure to execute work exists (cases, workers, routing). What's missing is the bridge: turning improvement observations into improvement actions through the goal and case lifecycle.

| Gap | What's missing |
|-----|---------------|
| Improvement sensing | No signal types for quality/stability/capability observations |
| Signal→goal bridge | No mechanism to form improvement goals from signal consensus |
| Budget enforcement | No improvement-specific budget — unbounded improvement is unsafe |
| Improvement lifecycle | No case template for the improvement cycle (introspect→implement→review→integrate) |
| Operational workers | No workers for dependency updates, lint, coverage, CI triage, recipes |
| Review gate | No mandatory code review step before improvement integration |
| Outcome tracking | No structured recording of what improved, regressed, or failed |
| Goal discrimination | No GoalKind to let evaluators apply improvement-specific logic |

**Scope boundary:** This issue delivers the complete single-shot improvement cycle. #1115 adds the continuous loop (standing directive, feedback, growth direction). Cognitive agent integration (CognitionCore, PAD, narrative) is Epic 2. Research loop is Epic 3. Full cognitive memory is Epic 4.

**Capability evolution:** The neocortex cognitive stack is under active development. This spec references cognitive SPIs, not implementations. Before implementing each integration point, check the current blocks/neocortex codebase — new capabilities may provide better integration paths than what this snapshot describes. See parent spec §Capability Evolution Protocol.

## Architecture Overview

```
┌────────────────────────────────────────────────────────────────────────────┐
│                   Autonomous Self-Improvement (Engine Foundation)          │
│                                                                            │
│  ┌────────────────────┐  ┌────────────────────┐  ┌─────────────────────┐  │
│  │ ImprovementConfig  │  │ ImprovementBudget  │  │ ImprovementBudget   │  │
│  │                    │  │                    │  │ Enforcer            │  │
│  │ • signalNamespace  │  │ • maxConcurrent    │  │                     │  │
│  │ • consensusMin     │  │ • maxPerDay        │  │ • structural deny   │  │
│  │ • improvementType  │  │ • cooldownMinutes  │  │ • path matching     │  │
│  │ • enabledCategories│  │ • allowedRepos     │  │ • budget check      │  │
│  │ • caseTemplateId   │  │ • deniedPaths      │  │ • concurrent check  │  │
│  │                    │  │ • requireReview    │  │                     │  │
│  │                    │  │ • maxPRSize        │  │                     │  │
│  └────────┬───────────┘  └────────┬───────────┘  └────────┬────────────┘  │
│           │                       │                        │               │
│           ▼                       ▼                        ▼               │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │              Goal Formation Bridge                                  │  │
│  │                                                                     │  │
│  │  Signal consensus → ImprovementGoalFormationStrategy.propose()      │  │
│  │           → GoalFormationService.propose(SELF_IMPROVEMENT goal)     │  │
│  │           → Budget check → Improvement case spawn                  │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │              Improvement Case Template                              │  │
│  │                                                                     │  │
│  │  introspect → [research → analyse] → implement → submit-pr         │  │
│  │            conditional (capability)    → review → integrate         │  │
│  │                                                                     │  │
│  │  Each step = capability-routed worker                               │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │              Operational Workers                                    │  │
│  │                                                                     │  │
│  │  DependencyUpdate │ LintFix │ CoverageGap │ CITriage │ Recipe      │  │
│  │  Worker           │ Worker  │ Worker      │ Worker   │ Worker      │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │              Outcome Tracking (3 layers)                            │  │
│  │                                                                     │  │
│  │  ImprovementOutcome → EventLog (source of truth)                   │  │
│  │                     → Signal projection (real-time notification)    │  │
│  │                     → CBR trace (historical learning)              │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │              Platform SPIs (existing)                               │  │
│  │                                                                     │  │
│  │  SignalRegistry · GoalFormationService · GoalFormationStrategy      │  │
│  │  SubCaseBinding · WorkerScope · WorkerRuntime                      │  │
│  │  EventLog · CbrCaseMemoryStore · AgentRoutingStrategy              │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────────────┘
```

**Data flow — improvement cycle:**

1. Agents deposit `improvement:*` signals when they observe quality issues (via observers or rule-based detection)
2. Signal reaches consensus (multiple independent sources reinforce — same pattern as `swarm:need-capacity`)
3. `ImprovementGoalFormationStrategy` detects consensus, reads improvement context from signal metadata
4. Strategy calls `ImprovementBudgetEnforcer.check()` — validates concurrent, daily, path, and repo constraints
5. Budget passes → `GoalFormationService.propose()` creates a SELF_IMPROVEMENT goal
6. Goal dispatched through standard goal lifecycle → spawns improvement case via SubCaseBinding
7. Case template executes: introspect → implement → submit-pr → review → integrate
8. Outcome recorded: `ImprovementOutcome` → EventLog + signal projection + CBR trace

## 1. Signal Types

Improvement signals follow a hierarchical namespace convention. All signal names are prefixed with `improvement:` and use colon-separated segments for category, subcategory, and detail.

### Signal namespace

```
improvement:{dimension}:{category}:{detail}
```

| Dimension | Categories | Example signals |
|-----------|-----------|-----------------|
| `stability` | `ci`, `test`, `build` | `improvement:stability:ci:build-failure`, `improvement:stability:test:flaky-test` |
| `quality` | `lint`, `coverage`, `complexity` | `improvement:quality:lint:checkstyle-violation`, `improvement:quality:coverage:below-threshold` |
| `dependency` | `staleness`, `security`, `compatibility` | `improvement:dependency:staleness:major-behind`, `improvement:dependency:security:cve-detected` |
| `capability` | `reasoning`, `execution`, `perception`, `tools` | `improvement:capability:execution:coordination-failure` |
| `outcome` | `positive`, `regression`, `rejected`, `failed` | `improvement:outcome:positive:pr-merged`, `improvement:outcome:regression:ci-regression` |

Signal metadata carries improvement context as signal properties (the existing `Map<String, String>` on signal entries):

| Property | Purpose | Example |
|----------|---------|---------|
| `improvementType` | `operational` or `capability` | `operational` |
| `category` | Worker category this maps to | `dependency-update` |
| `target` | What specifically needs improvement | `hibernate-core:6.6.0` |
| `severity` | `low`, `medium`, `high`, `critical` | `high` |
| `repo` | Target repository | `casehubio/engine` |
| `detail` | Human-readable description | `hibernate-core 6.6 → 6.7 available (3 months stale)` |

### Consensus model

Improvement signals use the same consensus pattern as self-provisioning: multiple independent agents must deposit the same signal type before it is acted on. The consensus threshold is configurable via `ImprovementConfig.consensusMinSources()` (default: 2).

Consensus is evaluated per signal namespace prefix. Two signals sharing `improvement:dependency:staleness` from different agents count toward consensus, even if they differ in `:detail`. This allows agents with different observation patterns to independently arrive at the same improvement category.

## 2. Configuration Model

### ImprovementConfig

Top-level configuration for the self-improvement system. Added to `StigmergyConfig` alongside `SwarmConfig`.

```java
// api/model/stigmergy
public record ImprovementConfig(
    @Nullable String signalNamespace,
    @Nullable Integer consensusMinSources,
    @Nullable List<String> enabledCategories,
    @Nullable ImprovementBudget budget,
    @Nullable String caseTemplateId) {

  public String effectiveSignalNamespace() {
    return signalNamespace != null ? signalNamespace : "improvement";
  }

  public int effectiveConsensusMinSources() {
    return consensusMinSources != null ? consensusMinSources : 2;
  }

  public List<String> effectiveEnabledCategories() {
    return enabledCategories != null ? enabledCategories
        : List.of("dependency-update", "lint-fix", "coverage-gap", "ci-triage", "recipe");
  }

  public ImprovementBudget effectiveBudget() {
    return budget != null ? budget : new ImprovementBudget(null, null, null, null, null, null, null);
  }

  public String effectiveCaseTemplateId() {
    return caseTemplateId != null ? caseTemplateId : "self-improvement";
  }
}
```

### ImprovementBudget

Improvement-specific budget record, following the `ProvisionBudget` pattern:

```java
// api/model/stigmergy
public record ImprovementBudget(
    @Nullable Integer maxConcurrent,
    @Nullable Integer maxPerDay,
    @Nullable Integer cooldownMinutes,
    @Nullable List<String> allowedRepos,
    @Nullable List<String> deniedPaths,
    @Nullable Boolean requireReview,
    @Nullable Integer maxPRSize) {

  public int effectiveMaxConcurrent() {
    return maxConcurrent != null ? maxConcurrent : 3;
  }

  public int effectiveMaxPerDay() {
    return maxPerDay != null ? maxPerDay : 10;
  }

  public int effectiveCooldownMinutes() {
    return cooldownMinutes != null ? cooldownMinutes : 30;
  }

  public List<String> effectiveAllowedRepos() {
    return allowedRepos != null ? allowedRepos : List.of();
  }

  public List<String> effectiveDeniedPaths() {
    return deniedPaths != null ? deniedPaths : List.of();
  }

  public boolean effectiveRequireReview() {
    return requireReview != null ? requireReview : true;
  }

  public int effectiveMaxPRSize() {
    return maxPRSize != null ? maxPRSize : 500;
  }
}
```

**Conservative defaults:** 3 concurrent, 10/day, 30-minute cooldown, review required, 500 lines max. The system is opt-in and constrained by default.

## 3. Budget Enforcement

### ImprovementBudgetEnforcer

`runtime-core`, `io.casehub.engine.internal.improvement`

Validates an improvement request against budget constraints before the improvement case is spawned. Returns a sealed result indicating whether the improvement is allowed.

```java
@ApplicationScoped
public class ImprovementBudgetEnforcer implements Resettable {

  private static final Set<String> STRUCTURAL_DENIED_PATHS = Set.of(
      "**/ImprovementBudget*",
      "**/ImprovementBudgetEnforcer*",
      "**/ImprovementConfig*",
      "**/SafetyConfig*",
      "**/improvement-case-template*"
  );

  public sealed interface BudgetCheck
      permits BudgetCheck.Allowed, BudgetCheck.Denied {
    record Allowed() implements BudgetCheck {}
    record Denied(String reason) implements BudgetCheck {}
  }

  public BudgetCheck check(
      UUID caseId, ImprovementBudget budget, ImprovementRequest request) { ... }
}
```

**Check layers (all must pass):**

| Layer | What it checks | Denial reason |
|-------|---------------|---------------|
| Structural deny-list | Request target paths against `STRUCTURAL_DENIED_PATHS` | `"Structural self-modification denied: <path>"` |
| User deny-list | Request target paths against `budget.effectiveDeniedPaths()` | `"Path denied by configuration: <path>"` |
| Repo allow-list | Request target repo against `budget.effectiveAllowedRepos()` (empty = allow all) | `"Repository not in allowed list: <repo>"` |
| Concurrent limit | Active improvement cases against `budget.effectiveMaxConcurrent()` | `"Concurrent improvement limit reached: <n>/<max>"` |
| Daily limit | Improvements started today against `budget.effectiveMaxPerDay()` | `"Daily improvement limit reached: <n>/<max>"` |
| Cooldown | Time since last improvement against `budget.effectiveCooldownMinutes()` | `"Cooldown active: <remaining> minutes"` |
| PR size | Estimated change size against `budget.effectiveMaxPRSize()` | `"Estimated change size <n> exceeds limit <max>"` |

**Structural self-modification denial is hardcoded and non-overridable.** User-configured `deniedPaths` are additive to structural denials. The improvement system cannot modify its own safety constraints regardless of configuration. This is the most critical safety property of the system.

### ImprovementRequest

The request object passed to the budget enforcer:

```java
// api/model/stigmergy
public record ImprovementRequest(
    String improvementType,
    String category,
    String target,
    String targetRepo,
    List<String> targetPaths,
    int estimatedSize,
    Map<String, String> metadata) {}
```

## 4. Goal Integration

### GoalKind.SELF_IMPROVEMENT

Add `SELF_IMPROVEMENT` to `StandardGoalKind`:

```java
SELF_IMPROVEMENT("self_improvement", CaseStatus.COMPLETED);
```

Terminal status is COMPLETED — a successful improvement completes the case. Failed improvements are handled through goal abandonment (separate from goal completion with FAILURE), not through the goal completing in a failed state.

### ImprovementGoalFormationStrategy

`runtime-core`, `io.casehub.engine.internal.improvement`

A `GoalFormationStrategy` that proposes SELF_IMPROVEMENT goals when improvement signal consensus is reached. Registered with `GoalFormationService` through CDI.

```java
@ApplicationScoped
public class ImprovementGoalFormationStrategy implements GoalFormationStrategy {

  @Override
  public GoalFormationProposal propose(GoalFormationContext context) {
    // 1. Scan signals for improvement consensus
    // 2. For each consensus cluster:
    //    a. Build ImprovementRequest from signal metadata
    //    b. Check budget via ImprovementBudgetEnforcer
    //    c. If allowed, propose a SELF_IMPROVEMENT goal
    // 3. Return proposal with improvement-specific attributes:
    //    - improvementType, category, target, targetRepo
  }

  @Override
  public String id() {
    return "self-improvement";
  }
}
```

**Goal attributes:** The `ProposedGoal.attributes()` map carries improvement-specific context that flows through the goal lifecycle into the improvement case:

| Attribute key | Value | Purpose |
|--------------|-------|---------|
| `improvement.type` | `operational` / `capability` | Selects case template bindings |
| `improvement.category` | Worker category slug | Routes to correct worker |
| `improvement.target` | What to improve | Worker input |
| `improvement.targetRepo` | Repository | Scoping |
| `improvement.severity` | `low`–`critical` | Priority input |
| `improvement.signalCount` | Number of consensus signals | Confidence indicator |

### Goal Evaluator Integration

The existing `GoalFormationEvaluator` already invokes registered `GoalFormationStrategy` instances. `ImprovementGoalFormationStrategy` is discovered via CDI — no evaluator changes needed.

`GoalRevisionEvaluator` and `GoalAbandonmentEvaluator` apply improvement-specific logic by checking `goal.kind() == StandardGoalKind.SELF_IMPROVEMENT`:

- **Revision:** Repeated failures in a category deprioritise it. Uses structured outcome records (§8) to detect failure patterns.
- **Abandonment:** 3 rejected PRs in the same improvement direction → abandon. Regression detected → abandon and emit rollback signal.

## 5. Improvement Case Template

The improvement lifecycle is modelled as a case template using the standard hybrid Java/YAML format. The case template ID is `self-improvement` (configurable via `ImprovementConfig.caseTemplateId`).

### Case definition (YAML)

```yaml
id: self-improvement
name: Self-Improvement
description: Autonomous improvement lifecycle — introspect, implement, review, integrate

bindings:
  - name: introspect
    trigger:
      type: on-create
    capability: improvement-introspect
    
  - name: research
    trigger:
      type: on-complete
      source: introspect
    when: "context.layer('WORKING').get('improvementType') == 'capability'"
    capability: improvement-research
    
  - name: analyse
    trigger:
      type: on-complete
      source: research
    when: "context.layer('WORKING').get('improvementType') == 'capability'"
    capability: improvement-analyse
    
  - name: implement
    trigger:
      type: on-complete
      source: introspect
    when: "context.layer('WORKING').get('improvementType') == 'operational'"
    capability: improvement-implement
    
  - name: implement-capability
    trigger:
      type: on-complete
      source: analyse
    capability: improvement-implement
    
  - name: submit-pr
    trigger:
      type: on-complete
      source: implement
    capability: improvement-submit-pr
    
  - name: submit-pr-capability
    trigger:
      type: on-complete
      source: implement-capability
    capability: improvement-submit-pr
    
  - name: review
    trigger:
      type: on-signal
      signal: "improvement:pr-submitted"
    capability: code-review
    
  - name: integrate
    trigger:
      type: on-complete
      source: review
    when: "context.layer('WORKING').get('reviewOutcome') == 'approved'"
    capability: improvement-integrate
    
  - name: record-outcome
    trigger:
      type: on-complete
      source: integrate
    capability: improvement-outcome
```

### Execution paths

**Operational improvement:**
```
introspect → implement → submit-pr → review → integrate → record-outcome
```

**Capability improvement:**
```
introspect → research → analyse → implement → submit-pr → review → integrate → record-outcome
```

The `when` conditions on the research/analyse bindings gate on `improvementType == 'capability'`. For operational improvements, those bindings don't fire and the case proceeds directly from introspect to implement. This is standard case binding behaviour — no special conditional logic needed.

## 6. Operational Workers

Five rule-based worker categories for operational improvements. Each worker implements the standard worker contract (`WorkerResult` return, `WorkerScope` parameter, cast to `WorkerRuntime` for engine methods).

All workers interact with external systems via REST/GraphQL/MCP APIs — no shell scripting, no exec calls. This is the same pattern as `WorkerProvisioner` calling external systems.

### Worker capabilities

| Worker | Capability | What it does | API surface |
|--------|-----------|-------------|-------------|
| `DependencyUpdateWorker` | `improvement-dependency-update` | Detects stale/vulnerable deps, generates update | Package manager REST APIs |
| `LintFixWorker` | `improvement-lint-fix` | Applies lint/checkstyle fixes from violation reports | Build tool output parsing |
| `CoverageGapWorker` | `improvement-coverage-gap` | Identifies uncovered code, generates test skeletons | Coverage report APIs |
| `CITriageWorker` | `improvement-ci-triage` | Diagnoses CI failure patterns, proposes fixes | CI platform APIs |
| `RecipeWorker` | `improvement-recipe` | Applies code transformation recipes | AST/OpenRewrite APIs |

### Common worker contract

Each improvement worker implements the introspect and implement capabilities. The `improvement-introspect` capability is shared — all workers can introspect their domain. The `improvement-implement` capability is per-category — routing selects the best worker based on the `improvement.category` attribute in the case context.

```java
// runtime-core, io.casehub.engine.internal.improvement.worker
public abstract class AbstractImprovementWorker {

  protected abstract String category();

  protected abstract WorkerResult introspect(
      WorkerScope scope, ImprovementRequest request);

  protected abstract WorkerResult implement(
      WorkerScope scope, IntrospectionResult introspection);
}
```

### IntrospectionResult

The output of the introspect step, carried forward as case context for implement:

```java
// api/model/stigmergy
public record IntrospectionResult(
    String category,
    String description,
    List<String> affectedPaths,
    int estimatedSize,
    String proposedChange,
    Map<String, String> metadata) {}
```

### Worker module placement

Improvement workers live in `runtime-core` under `io.casehub.engine.internal.improvement.worker`. They are `@ApplicationScoped` beans discovered via CDI. Each worker registers its capability with the routing system — the standard capability registration pattern.

### Implementation note — workers are working implementations

Per D94: engine workers do real work via APIs, not stubs. `DependencyUpdateWorker` calls package manager APIs to check versions and generate update diffs. `LintFixWorker` parses build tool lint output and generates fixes. These are the same kind of structured, API-mediated interactions that `WorkerProvisioner` already performs.

Blocks can enhance any worker with LLM-powered reasoning (e.g., LLM-based code analysis for `CoverageGapWorker`). The worker capability routing naturally selects the best implementation — engine rule-based or blocks LLM-enhanced — based on capability matching and trust.

## 7. DevTown Review Gate

DevTown is modelled as a standard worker with `code-review` capability. No dedicated SPI.

### How it works

1. The `submit-pr` binding produces a PR and deposits `improvement:pr-submitted` signal
2. The `review` binding triggers on that signal and routes to a `code-review` capability worker
3. DevTown (or any registered code-review worker) handles the review
4. Review outcome written to case context as `reviewOutcome`
5. The `integrate` binding's `when` condition checks `reviewOutcome == 'approved'`
6. If rejected, the case lifecycle handles failure (goal revision/abandonment)

### Defensive hard gate

The integration worker (`ImprovementIntegrationWorker`) performs a pre-flight check against the case's EventLog before proceeding:

```java
EventLog log = runtime.eventLog(scope.caseId());
boolean hasPassingReview = log.entries().stream()
    .anyMatch(e -> e.type() == CaseHubEventType.REVIEW_COMPLETED
        && "approved".equals(e.properties().get("verdict")));
if (!hasPassingReview) {
  throw new IllegalStateException(
      "Integration blocked: no passing review record found for improvement case "
          + scope.caseId());
}
```

This is belt-and-suspenders: even if a lifecycle bug allows the integrate binding to fire without a review, the worker refuses to execute. Not an SPI — an internal check.

## 8. Outcome Tracking

Three-layer event-sourced model following the `CaseLedgerEventCapture` pattern.

### ImprovementOutcome

Structured outcome record — the source of truth:

```java
// api/model/stigmergy
public record ImprovementOutcome(
    UUID caseId,
    UUID improvementCaseId,
    String category,
    String target,
    OutcomeStatus status,
    @Nullable String prUrl,
    @Nullable Integer ciDelta,
    @Nullable Double coverageDelta,
    @Nullable Integer lintDelta,
    Instant completedAt,
    Map<String, String> metadata) {

  public enum OutcomeStatus {
    MERGED, REJECTED, REGRESSION, FAILED, ABANDONED
  }
}
```

### Layer 1: EventLog record

The `ImprovementOutcomeRecorder` writes a typed `EventLog` entry when an improvement case completes:

```java
@ApplicationScoped
public class ImprovementOutcomeRecorder {

  public void record(UUID caseId, ImprovementOutcome outcome) {
    eventLog.append(caseId, CaseHubEventType.IMPROVEMENT_OUTCOME,
        Map.of(
            "category", outcome.category(),
            "target", outcome.target(),
            "status", outcome.status().name(),
            "prUrl", Objects.toString(outcome.prUrl(), ""),
            "ciDelta", String.valueOf(outcome.ciDelta()),
            "coverageDelta", String.valueOf(outcome.coverageDelta()),
            "lintDelta", String.valueOf(outcome.lintDelta())
        ));
  }
}
```

### Layer 2: Signal projection

From the EventLog record, project signals for real-time swarm notification:

| Outcome status | Signal projected | Purpose |
|---------------|-----------------|---------|
| MERGED | `improvement:outcome:positive:pr-merged` | Swarm knows improvement landed |
| REJECTED | `improvement:outcome:rejected` | Swarm adjusts approach for this category |
| REGRESSION | `improvement:outcome:regression` | Swarm detects and avoids similar changes |
| FAILED | `improvement:outcome:failed` | Swarm learns from failure |
| ABANDONED | `improvement:outcome:abandoned` | Swarm knows direction was abandoned |

Signals carry the outcome metadata as properties — any agent can perceive the outcome through normal signal perception.

### Layer 3: CBR trace

From the EventLog record, store a CBR trace for historical learning:

```java
CbrCase cbrCase = FeatureVectorCbrCase.builder()
    .feature("category", FeatureValue.categorical(outcome.category()))
    .feature("target", FeatureValue.text(outcome.target()))
    .feature("status", FeatureValue.categorical(outcome.status().name()))
    .feature("ciDelta", FeatureValue.numeric(outcome.ciDelta()))
    .feature("coverageDelta", FeatureValue.numeric(outcome.coverageDelta()))
    .feature("lintDelta", FeatureValue.numeric(outcome.lintDelta()))
    .build();
cbrStore.store(tenantId, "improvement-outcomes", cbrCase);
```

Future improvement cycles retrieve similar past outcomes via `CbrCaseRetriever` — "What happened last time we tried a dependency bump on this module?" — using feature vector similarity. The CBR infrastructure (neocortex) handles retrieval, temporal decay, and scope decay.

### ImprovementOutcomeEventCapture

CDI observer that listens for improvement case completion events and triggers all three projections:

```java
@ApplicationScoped
public class ImprovementOutcomeEventCapture {

  public void onImprovementComplete(
      @ObservesAsync ImprovementCaseCompleted event) {
    ImprovementOutcome outcome = buildOutcome(event);
    // Layer 1: EventLog
    outcomeRecorder.record(event.caseId(), outcome);
    // Layer 2: Signal projection
    signalProjector.project(event.caseId(), outcome);
    // Layer 3: CBR trace
    cbrProjector.project(event.tenantId(), outcome);
  }
}
```

**Testing note:** `@ObservesAsync` is unreliable in `@QuarkusTest`. The test must inject `ImprovementOutcomeEventCapture` and call `onImprovementComplete()` directly.

## 9. Extension Points for Future Epics

The engine foundation provides clean SPI extension points that future epics (2–5) and evolving cognitive capabilities can use without modifying engine code.

### For Epic 2 (Cognitive Agent — blocks)

| Extension point | How blocks uses it |
|----------------|-------------------|
| `DriveSource` SPI | Implement `ImprovementCompetenceSource`, `ImprovementCuriositySource`, etc. — feed improvement metrics into the drive system |
| `DriveGoalMapper` SPI | Map improvement drive intensity to SELF_IMPROVEMENT goals |
| `GoalFormationStrategy` SPI | LLM-enhanced goal formation using `DriveGoalFormationStrategy` |
| `CognitionTickParticipant` SPI | Custom cognition tick logic for improvement evaluation |
| `improvement:outcome:*` signals | Feed PAD mood signals from improvement outcomes |
| `ImprovementOutcome` records | Strategy learning input — what approaches work |

### For Epic 3 (Research Loop — blocks + neocortex)

| Extension point | How it's used |
|----------------|---------------|
| `improvement-research` capability | Blocks provides LLM-powered research worker |
| `improvement-analyse` capability | Blocks provides LLM-powered analysis worker |
| Case template conditional bindings | Research/analyse fire only for `capability` type |
| CBR traces | Research outcomes stored alongside operational outcomes |

### For Epic 4 (Full Cognitive Memory — neocortex)

| Extension point | How it's used |
|----------------|---------------|
| `ImprovementOutcome` records | Source for episodic memory nodes with emotional valence |
| `ConsolidationPhase` SPI | Improvement-specific consolidation from episodic to semantic |
| `CuriositySignalProvider` SPI | Improvement knowledge gaps as curiosity signals |
| `ModulationFactor` SPI | Mood-congruent retrieval of improvement memories |

### For Epic 5 (Continuous Evolution — #1115)

| Extension point | How it's used |
|----------------|---------------|
| `improvement:outcome:*` signals | Feedback loop — outcomes feed back into detection |
| `ImprovementBudget` | Budget relaxation as trust grows |
| Goal evaluator discrimination on `SELF_IMPROVEMENT` | Continuous goal revision based on outcome patterns |
| `ImprovementConfig.enabledCategories` | Growth direction — enable/disable categories dynamically |

### For evolving cognitive capabilities

| When this lands... | ...it can use |
|--------------------|---------------|
| Goal cognition (neocortex#345) | `SELF_IMPROVEMENT` goals get affective valuation, dependency graphs, multi-signal priority |
| Enhanced consolidation (neocortex#336) | Improvement episodic memories get better promotion scoring |
| Cognitive node types (neocortex#322) | Improvement knowledge nodes get typed classification |
| Relationship memory (neocortex#184, #186) | Reviewer relationship tracking enriches review outcome interpretation |
| Trust evolution (blocks#65) | Trust-gated budget relaxation |
| Structured agent invoker (blocks#287) | Typed LLM invocation for LLM-enhanced workers |

**Protocol:** When any of these land, check whether they provide a better integration path for an existing self-improvement component. If so, update the implementation. The SPI boundaries are designed for this — new implementations slot in without changing engine code.

## 10. Module Placement

| Component | Module | Package |
|-----------|--------|---------|
| `ImprovementConfig` | `api` | `io.casehub.api.model.stigmergy` |
| `ImprovementBudget` | `api` | `io.casehub.api.model.stigmergy` |
| `ImprovementRequest` | `api` | `io.casehub.api.model.stigmergy` |
| `IntrospectionResult` | `api` | `io.casehub.api.model.stigmergy` |
| `ImprovementOutcome` | `api` | `io.casehub.api.model.stigmergy` |
| `StandardGoalKind.SELF_IMPROVEMENT` | `api` | `io.casehub.api.model` |
| `CaseHubEventType.IMPROVEMENT_OUTCOME` | `api` | `io.casehub.api.model.event` |
| `ImprovementBudgetEnforcer` | `runtime-core` | `io.casehub.engine.internal.improvement` |
| `ImprovementGoalFormationStrategy` | `runtime-core` | `io.casehub.engine.internal.improvement` |
| `ImprovementOutcomeRecorder` | `runtime-core` | `io.casehub.engine.internal.improvement` |
| `ImprovementOutcomeEventCapture` | `runtime-core` | `io.casehub.engine.internal.improvement` |
| `ImprovementSignalProjector` | `runtime-core` | `io.casehub.engine.internal.improvement` |
| `ImprovementCbrProjector` | `runtime-core` | `io.casehub.engine.internal.improvement` |
| `AbstractImprovementWorker` | `runtime-core` | `io.casehub.engine.internal.improvement.worker` |
| `DependencyUpdateWorker` | `runtime-core` | `io.casehub.engine.internal.improvement.worker` |
| `LintFixWorker` | `runtime-core` | `io.casehub.engine.internal.improvement.worker` |
| `CoverageGapWorker` | `runtime-core` | `io.casehub.engine.internal.improvement.worker` |
| `CITriageWorker` | `runtime-core` | `io.casehub.engine.internal.improvement.worker` |
| `RecipeWorker` | `runtime-core` | `io.casehub.engine.internal.improvement.worker` |
| `ImprovementIntegrationWorker` | `runtime-core` | `io.casehub.engine.internal.improvement.worker` |
| Case template YAML | `runtime/src/main/resources` | `case-templates/self-improvement.yaml` |

## 11. Test Strategy

### Unit tests

| Test class | What it covers |
|------------|---------------|
| `ImprovementBudgetEnforcerTest` | All budget check layers: structural deny, user deny, repo allow, concurrent, daily, cooldown, PR size |
| `ImprovementGoalFormationStrategyTest` | Signal consensus detection, budget gating, goal proposal attributes |
| `ImprovementOutcomeRecorderTest` | EventLog entry creation with correct type and properties |
| `ImprovementSignalProjectorTest` | Correct signal types for each outcome status |
| `ImprovementCbrProjectorTest` | Feature vector construction and storage |
| Per-worker tests | Each worker's introspect and implement logic |

### Integration tests

| Test class | What it covers |
|------------|---------------|
| `SelfImprovementIntegrationTest` | Full lifecycle: signal deposit → consensus → goal formation → case spawn → worker execution → review → integration → outcome recording (all 3 layers) |
| `ImprovementBudgetIntegrationTest` | Budget enforcement under concurrent improvement load — verify limits are respected |
| `ImprovementSafetyTest` | Structural self-modification denial — verify budget enforcer rejects improvements targeting safety infrastructure |

### Critical test scenarios

1. **Structural deny-list is non-bypassable:** Configure `deniedPaths: []` in `ImprovementBudget`, attempt to improve a file matching `STRUCTURAL_DENIED_PATHS` → must be denied
2. **Consensus requirement:** Single signal deposit → no goal formed. Two independent agents → goal formed
3. **Review gate is mandatory:** Skip the review binding somehow → `ImprovementIntegrationWorker` throws `IllegalStateException`
4. **Outcome tracking completeness:** All three layers (EventLog, signal, CBR) are populated from a single improvement outcome event
5. **Conditional bindings work:** Operational improvement skips research/analyse; capability improvement includes them

## 12. YAML Record Codegen

`ImprovementConfig` and `ImprovementBudget` need YAML deserialization records for the DSL. Add entries to `yaml-record-mappings.yaml`:

```yaml
ImprovementConfig:
  source: inline
  package: io.casehub.api.model.converter.yaml
  fields:
    signalNamespace: String
    consensusMinSources: Integer
    enabledCategories: List<String>
    budget: ImprovementBudget
    caseTemplateId: String

ImprovementBudget:
  source: inline
  package: io.casehub.api.model.converter.yaml
  fields:
    maxConcurrent: Integer
    maxPerDay: Integer
    cooldownMinutes: Integer
    allowedRepos: List<String>
    deniedPaths: List<String>
    requireReview: Boolean
    maxPRSize: Integer
```

Also update `StigmergyConfig` YAML mapping to include the new `improvement` field alongside `swarm`.
