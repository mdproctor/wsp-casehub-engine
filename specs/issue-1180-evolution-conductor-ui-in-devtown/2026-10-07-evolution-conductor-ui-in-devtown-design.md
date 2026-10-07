# Evolution Conductor UI in Devtown — Design Spec

**Issue:** casehubio/engine#1180
**Epic:** casehubio/engine#1149 (Phase 4: UI)
**Parent specs:** `2026-09-21-command-centre-conductor-design.md` (#1132), `2026-09-23-generalise-evolution-conductor-design.md` (#1148)
**Decisions:** D1–D7 in `decisions.md`
**Date:** 2026-10-07

## Problem

The evolution conductor (#1132) provides a 5-layer command centre (Observe, Summarize, Control, Steer, Review) with a full API surface via `DefaultEngineEvolutionApi`. The conductor was generalised (#1148) with pluggable SPIs for cross-domain use. The blocks-ui evolution components (#1149 Phase 4) — workbench shell, deny/watch/gate editors, inbox, streams — are built and tested.

But no application consumes the conductor yet. The blocks-ui components render against mock data. There is no live feedback loop: no domain-specific sensors feeding health scores, no improvement categories driving proposals, no real conductor inbox entries to resolve.

Devtown is the natural first consumer — it has a rich data surface (PR outcomes, CI results, reviewer trust, merge queue state, SLA compliance) and an existing dashboard infrastructure (`casehub-pages` with 8 tabs). Wiring devtown to the conductor produces the full feedback loop: sense → assess → propose → gate → apply → observe.

## Scope

**Three repos, one issue each (cross-linked):**

| Repo | Work | Issue |
|------|------|-------|
| `casehubio/engine` | Ensure SPIs support application-level consumers cleanly (if gaps found during implementation) | #1180 (this issue) |
| `casehubio/devtown` | Capability areas, category provider, API facade, Evolution tab, case template | New issue, refs #1180 |
| `casehubio/blocks-ui` | No changes expected — existing components sufficient (D6) | Only if gaps found |

## Design Overview

```
┌─────────────────────────────────────────────────────────┐
│                    Devtown Dashboard                     │
│  ┌─────┬──────┬─────┬────┬───┬──────┬───┬────┬─────────┐│
│  │Rev. │Queue │Rev-r│Cont│Wrk│Triage│Sys│Defs│Evolution││
│  └─────┴──────┴─────┴────┴───┴──────┴───┴────┴────┬────┘│
│                                                    │     │
│  ┌─────────────────────────────────────────────────▼────┐│
│  │          blocks-evolution-workbench                   ││
│  │  ┌─────────────────────────────────────────────────┐ ││
│  │  │ Summary: Health | Active | Inbox | CB | Enabled │ ││
│  │  └─────────────────────────────────────────────────┘ ││
│  │  Tabs: Streams | Inbox | Audit | Config | Health    ││
│  │        + [PR Impact] (custom devtown tab)            ││
│  └──────────────────────────┬───────────────────────────┘│
│                             │                            │
│                    /api/devtown/evolution                 │
│                             │                            │
│  ┌──────────────────────────▼───────────────────────────┐│
│  │           DevtownEvolutionApi (facade)                ││
│  │  delegates to EngineEvolutionApi                      ││
│  │  enriches views with PR links, CI context             ││
│  └──────────────────────────┬───────────────────────────┘│
│                             │                            │
│  ┌──────────────────────────▼───────────────────────────┐│
│  │          Engine Evolution Conductor                   ││
│  │  EvolutionTicker → HealthScoreTracker                ││
│  │  → ImprovementGoalFormationStrategy                  ││
│  │  → ConductorInboxManager                             ││
│  └──────────────────────────┬───────────────────────────┘│
│                             │                            │
│  ┌──────────────────────────▼───────────────────────────┐│
│  │        Devtown Capability Areas (CDI)                 ││
│  │  CiReliability | ReviewQuality | MergeQueueHealth    ││
│  │  ReviewerTrust | SlaCompliance                       ││
│  └──────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────┘
```

## 1. Devtown Capability Areas (D1, D2)

Five `CapabilityArea` implementations registered via CDI in devtown's `domain` module. Each has a `domainId()` of `"devtown"` and an `assess(caseId, tenancyId)` method that reads from devtown's existing beans to compute a health score.

### 1.1 CiReliabilityCapabilityArea

```java
// devtown domain, io.casehub.devtown.domain.evolution
@ApplicationScoped
public class CiReliabilityCapabilityArea extends AbstractCapabilityArea {

  @Override public String areaId() { return "ci-reliability"; }
  @Override public String domainId() { return "devtown"; }
  @Override public String name() { return "CI Reliability"; }
  @Override public double weight() { return 0.25; }

  @Override
  public CapabilityAreaAssessment assess(UUID caseId, String tenancyId) {
    // Read from CiRunnerWorker outcome history:
    // - Build pass rate over sliding window (e.g., last 50 builds)
    // - Flaky test detection (builds that fail then pass on retry)
    // - CI duration trend (is it getting slower?)
    // healthScore = weighted(passRate * 0.5 + flakyRate * 0.3 + durationTrend * 0.2)
  }
}
```

**Data sources:** `CiRunnerWorker` outcomes stored in EventLog, GitHub CI status events received via webhook.

### 1.2 ReviewQualityCapabilityArea

```java
@ApplicationScoped
public class ReviewQualityCapabilityArea extends AbstractCapabilityArea {

  @Override public String areaId() { return "review-quality"; }
  @Override public String domainId() { return "devtown"; }
  @Override public String name() { return "Review Quality"; }
  @Override public double weight() { return 0.25; }

  @Override
  public CapabilityAreaAssessment assess(UUID caseId, String tenancyId) {
    // Read from ReviewFinding outcomes:
    // - Finding accuracy: findings that led to code changes vs false positives
    // - Review cycle time: time from PR open to review complete
    // - Missed-issue rate: reverts within N days of merge (indicates missed findings)
    // healthScore = weighted(accuracy * 0.4 + cycleTime * 0.3 + missedRate * 0.3)
  }
}
```

**Data sources:** `ReviewFinding` records, case lifecycle events (`CASE_COMPLETED`), revert detection from GitHub webhooks.

### 1.3 MergeQueueHealthCapabilityArea

```java
@ApplicationScoped
public class MergeQueueHealthCapabilityArea extends AbstractCapabilityArea {

  @Override public String areaId() { return "merge-queue-health"; }
  @Override public String domainId() { return "devtown"; }
  @Override public String name() { return "Merge Queue Health"; }
  @Override public double weight() { return 0.20; }

  @Override
  public CapabilityAreaAssessment assess(UUID caseId, String tenancyId) {
    // Read from MergeQueueService state:
    // - Queue depth (lower is better, normalised against target)
    // - Batch success rate (successful batches / total batches)
    // - Throughput: merges per day against target
    // - Time-to-merge: median time from enqueue to merge
    // healthScore = weighted(depth * 0.2 + batchSuccess * 0.3 + throughput * 0.25 + ttm * 0.25)
  }
}
```

**Data sources:** `MergeQueueService` state, batch outcome history, merge timestamps.

### 1.4 ReviewerTrustCapabilityArea

```java
@ApplicationScoped
public class ReviewerTrustCapabilityArea extends AbstractCapabilityArea {

  @Override public String areaId() { return "reviewer-trust"; }
  @Override public String domainId() { return "devtown"; }
  @Override public String name() { return "Reviewer Trust"; }
  @Override public double weight() { return 0.15; }

  @Override
  public CapabilityAreaAssessment assess(UUID caseId, String tenancyId) {
    // Read from TrustRoutingPolicy data:
    // - Trust score distribution: are scores clustered or well-spread?
    // - Calibration accuracy: does high-trust correlate with better outcomes?
    // - Review decline rate: how often do routed reviewers decline?
    // - Load balance: is work concentrated or well-distributed?
    // healthScore = weighted(calibration * 0.4 + declineRate * 0.3 + loadBalance * 0.3)
  }
}
```

**Data sources:** `DevtownTrustRoutingPolicyProvider`, reviewer trust dimension scores, routing outcome history.

### 1.5 SlaComplianceCapabilityArea

```java
@ApplicationScoped
public class SlaComplianceCapabilityArea extends AbstractCapabilityArea {

  @Override public String areaId() { return "sla-compliance"; }
  @Override public String domainId() { return "devtown"; }
  @Override public String name() { return "SLA Compliance"; }
  @Override public double weight() { return 0.15; }

  @Override
  public CapabilityAreaAssessment assess(UUID caseId, String tenancyId) {
    // Read from WorkItem SLA data:
    // - On-time completion rate: work items completed within SLA
    // - Escalation rate: work items that escalated due to SLA breach
    // - Triage backlog: number of unresolved triage items
    // - SLA calibration accuracy: configured vs actual durations
    // healthScore = weighted(onTime * 0.4 + escalation * 0.3 + backlog * 0.15 + calibration * 0.15)
  }
}
```

**Data sources:** `WorkItem` completion events, `SlaCalibrationService`, triage queue state.

### Area weight distribution

| Area | Weight | Rationale |
|------|--------|-----------|
| CI Reliability | 0.25 | Foundation — if CI is broken, nothing else works |
| Review Quality | 0.25 | Primary value proposition — review accuracy drives everything |
| Merge Queue Health | 0.20 | Throughput — how efficiently work lands |
| Reviewer Trust | 0.15 | Routing quality — secondary signal |
| SLA Compliance | 0.15 | Human gates — important but less frequent |

Total: 1.0. These weights are initial defaults — configurable via `ImprovementConfig` on the evolution case.

## 2. Devtown ImprovementCategoryProvider (D5)

```java
// devtown domain, io.casehub.devtown.domain.evolution
@ApplicationScoped
public class DevtownCategoryProvider implements ImprovementCategoryProvider {

  @Override
  public String domainId() { return "devtown"; }

  @Override
  public List<CategoryDescriptor> categories() {
    return List.of(
        new CategoryDescriptor("reviewer-calibration", "Reviewer Calibration",
            "Adjust reviewer trust weights when accuracy drifts", "devtown"),
        new CategoryDescriptor("routing-adjustment", "Routing Adjustment",
            "Modify capability-to-reviewer routing rules for load/quality balance", "devtown"),
        new CategoryDescriptor("sla-tuning", "SLA Tuning",
            "Adjust SLA thresholds when completion rates are too low or high", "devtown"),
        new CategoryDescriptor("gate-tightening", "Gate Tightening",
            "Add or modify gate policies when quality signals degrade", "devtown"),
        new CategoryDescriptor("queue-optimization", "Queue Optimization",
            "Adjust merge queue batch size and priority lanes for throughput", "devtown"));
  }

  @Override
  public List<StageDescriptor> stages() {
    return List.of(
        new StageDescriptor("analyze", "Analyze", 0, false, "devtown"),
        new StageDescriptor("propose", "Propose", 1, true, "devtown"),
        new StageDescriptor("review", "Review", 2, true, "devtown"),
        new StageDescriptor("apply", "Apply", 3, false, "devtown"),
        new StageDescriptor("observe", "Observe", 4, false, "devtown"));
  }
}
```

**Stage pipeline:** 5 stages vs code-evolution's 11. Two gate checkpoints: `propose` (conductor reviews the proposed change) and `review` (conductor approves before application). `apply` and `observe` are automatic — apply makes the configuration change, observe monitors for regression.

**Gate defaults:** Both `propose` and `review` start as `GATED` — all devtown improvements require explicit conductor approval until trust builds. Users can relax to `AUTO` or `NOTIFY` via the gate policy editor.

## 3. DevtownEvolutionApi Facade (D3)

```java
// devtown app, io.casehub.devtown.app.evolution
@ApplicationScoped
public class DevtownEvolutionApi {

  @Inject EngineEvolutionApi engineApi;
  @Inject DevtownEvolutionCaseResolver caseResolver;
  @Inject DevtownEvolutionEnricher enricher;

  public DevtownEvolutionStateSnapshot getEvolutionState(String tenancyId) {
    UUID caseId = caseResolver.resolve(tenancyId);
    EvolutionStateSnapshot state = engineApi.getEvolutionState(caseId, tenancyId);
    return enricher.enrich(state);
  }

  public List<ConductorInboxEntry> getInbox(String tenancyId) {
    UUID caseId = caseResolver.resolve(tenancyId);
    return engineApi.getInbox(caseId, tenancyId);
  }

  public void resolveGate(String tenancyId, String entryId, String decision,
      Map<String, Object> payload, String reason, String feedback) {
    UUID caseId = caseResolver.resolve(tenancyId);
    engineApi.resolveGate(caseId, tenancyId, entryId, decision, payload, reason, feedback);
  }

  // ... remaining mutations delegate directly, queries get enriched
}
```

### DevtownEvolutionCaseResolver

Resolves the singleton evolution case by template ID:

```java
@ApplicationScoped
public class DevtownEvolutionCaseResolver {

  private static final String TEMPLATE_ID = "devtown-evolution";

  @Inject CaseInstanceRepository caseInstanceRepository;

  public UUID resolve(String tenancyId) {
    return caseInstanceRepository
        .findByTemplateId(tenancyId, TEMPLATE_ID)
        .map(CaseInstance::id)
        .orElseThrow(() -> new IllegalStateException(
            "No evolution case found — DevtownEvolutionBootstrap should have created one"));
  }
}
```

### DevtownEvolutionEnricher

Enriches `EvolutionStateSnapshot` with devtown context:

```java
@ApplicationScoped
public class DevtownEvolutionEnricher {

  @Inject GovernanceService governanceService;

  public DevtownEvolutionStateSnapshot enrich(EvolutionStateSnapshot state) {
    var enrichedStreams = state.activeStreams().stream()
        .map(this::enrichStream)
        .toList();
    return new DevtownEvolutionStateSnapshot(state, enrichedStreams);
  }

  private DevtownImprovementStreamView enrichStream(ImprovementStreamView stream) {
    String target = stream.target();
    // Resolve PR identifier to display string:
    // "reviewer-calibration" target might be "reviewer:agent-security"
    // → enrich to "Security Reviewer (trust: 0.82, accuracy: 74%)"
    String enrichedTarget = resolveTarget(stream.category(), target);
    return new DevtownImprovementStreamView(stream, enrichedTarget);
  }
}
```

### REST endpoint

```java
// devtown app, io.casehub.devtown.app.evolution
@Path("/api/devtown/evolution")
@ApplicationScoped
public class DevtownEvolutionResource {

  @Inject DevtownEvolutionApi api;

  @GET @Path("/state")
  public DevtownEvolutionStateSnapshot getState(@QueryParam("tenancyId") String tenancyId) {
    return api.getEvolutionState(tenancyId);
  }

  @GET @Path("/inbox")
  public List<ConductorInboxEntry> getInbox(@QueryParam("tenancyId") String tenancyId) {
    return api.getInbox(tenancyId);
  }

  @POST @Path("/inbox/{entryId}/resolve")
  public void resolveGate(@PathParam("entryId") String entryId,
      @QueryParam("tenancyId") String tenancyId,
      GateResolutionRequest request) {
    api.resolveGate(tenancyId, entryId, request.decision(),
        request.payload(), request.reason(), request.feedback());
  }

  // ... remaining endpoints follow the same pattern
}
```

## 4. Evolution Tab Integration (D4)

### devtown/app/src/main/webui/src/index.ts changes

```typescript
import "@casehubio/blocks-ui-evolution-workbench";

registerPanel("evolution-workbench", "blocks-evolution-workbench");

// In the tabs() call, add after "System":
tabs(
  ["Reviews", hostPanel("review-workbench", { endpoint: "/api/devtown/governance" })],
  ["Merge Queue", queueView],
  ["Reviewers", reviewersView],
  ["Contributors", contributorsView],
  ["Workers", hostPanel("session-workbench", { endpoint: "/api/sessions" })],
  ["Triage", triageView],
  ["System", systemView],
  ["Evolution", hostPanel("evolution-workbench", { endpoint: "/api/devtown/evolution" })],
  ["Definitions", definitionsView],
),
```

### Evolution view (devtown/app/src/main/webui/src/views/evolution.ts)

```typescript
import { hostPanel } from "@casehubio/pages-ui";
import type { TabDefinition } from "@casehubio/blocks-ui-detail-pane";

const devtownTabs: TabDefinition[] = [
  {
    id: "pr-impact",
    label: "PR Impact",
    tagName: "div",
    order: 25,
    renderContent: () => html`<devtown-pr-impact-panel></devtown-pr-impact-panel>`,
  },
];

export const evolutionView = hostPanel("evolution-workbench", {
  endpoint: "/api/devtown/evolution",
  tabs: devtownTabs,
});
```

The `pr-impact` custom tab shows how conductor-driven improvements have affected PR metrics: review cycle time trends, finding accuracy changes, merge success rate improvements — the feedback loop made visible.

## 5. Evolution Case Template (D7)

### devtown-evolution.yaml

```yaml
caseTemplateId: devtown-evolution
name: DevTown Evolution
description: Continuous improvement for devtown's PR review pipeline

improvementConfig:
  evolutionEnabled: false
  enabledCategories:
    - reviewer-calibration
    - routing-adjustment
    - sla-tuning
    - gate-tightening
    - queue-optimization
  gatePolicy:
    modes:
      propose: GATED
      review: GATED
    gateTimeoutMinutes: 1440
  escalationPolicy:
    categoryRules:
      alwaysEscalate:
        - reviewer-calibration
        - routing-adjustment
      neverEscalate: []
    confidenceThreshold: 0.7
  budget:
    maxConcurrent: 1
    maxPerDay: 3
    cooldownMinutes: 60
    maxChangeSize: 10
```

**Bootstrap starts at L0_INERT** (`evolutionEnabled: false`). The Evolution tab shows health scores and capability area assessments immediately. The user enables evolution via the Configuration sub-tab when ready, progressing through L0→L1→L2→L3.

**Conservative defaults:** `maxConcurrent: 1` (one improvement at a time), all gates GATED, `reviewer-calibration` and `routing-adjustment` always escalate. This matches a trust-building approach — the conductor earns autonomy.

### DevtownEvolutionBootstrap

```java
// devtown app, io.casehub.devtown.app.evolution
@ApplicationScoped
public class DevtownEvolutionBootstrap {

  @Inject CaseInstanceFactory caseInstanceFactory;
  @Inject CaseInstanceRepository caseInstanceRepository;

  void onStartup(@Observes StartupEvent event) {
    String tenancyId = "default";
    var existing = caseInstanceRepository.findByTemplateId(tenancyId, "devtown-evolution");
    if (existing.isEmpty()) {
      caseInstanceFactory.create(tenancyId, "devtown-evolution");
    }
  }
}
```

## 6. Devtown ImprovementProposalSource

Devtown needs a proposal source that generates improvement proposals based on capability area assessments:

```java
// devtown domain, io.casehub.devtown.domain.evolution
@ApplicationScoped
public class DevtownProposalSource implements ImprovementProposalSource {

  @Override public String sourceId() { return "devtown-assessment"; }
  @Override public String domainId() { return "devtown"; }

  @Inject ReviewerTrustCapabilityArea reviewerTrustArea;
  @Inject SlaComplianceCapabilityArea slaArea;
  @Inject MergeQueueHealthCapabilityArea queueArea;

  @Override
  public List<ImprovementRequest> propose(UUID caseId, String tenancyId,
      ImprovementConfig config) {
    List<ImprovementRequest> proposals = new ArrayList<>();

    // Example: if reviewer trust calibration is drifting
    var trustAssessment = reviewerTrustArea.assess(caseId, tenancyId);
    if (trustAssessment.healthScore() < 0.6) {
      proposals.add(new ImprovementRequest(
          "recalibrate-trust-weights",
          "reviewer-calibration",
          identifyDriftingReviewer(caseId, tenancyId),
          "devtown",
          3,
          Map.of("trigger", "trust-drift",
                 "current-score", String.valueOf(trustAssessment.healthScore()))));
    }

    // Similar logic for SLA tuning, queue optimization, etc.
    return proposals;
  }
}
```

The proposal source uses capability area assessments as triggers — when health scores drop below thresholds, it generates improvement proposals for the appropriate category.

## 7. Devtown ConflictStrategy and DenyPatternProvider

### DevtownConflictStrategy

```java
@ApplicationScoped
public class DevtownConflictStrategy implements ConflictStrategy {

  @Override public String domainId() { return "devtown"; }

  @Override
  public ConflictResult check(ImprovementRequest request,
      Map<UUID, ImprovementRequest> activeImprovements, int trivialThreshold) {
    // Devtown conflicts are category-based:
    // Two reviewer-calibration improvements can't run concurrently
    // (they'd race on trust weight updates)
    for (var entry : activeImprovements.entrySet()) {
      if (entry.getValue().category().equals(request.category())
          && entry.getValue().domainId().equals("devtown")) {
        return new ConflictResult.Conflicting(entry.getKey(),
            "Same category active: " + request.category());
      }
    }
    return new ConflictResult.Clear();
  }
}
```

### DevtownDenyPatternProvider

```java
@ApplicationScoped
public class DevtownDenyPatternProvider implements DenyPatternProvider {

  @Override public String domainId() { return "devtown"; }

  @Override
  public boolean isDenied(UUID caseId, String tenancyId, ImprovementRequest request,
      ImprovementConfig config) {
    // Devtown deny patterns protect structural invariants:
    // - Never auto-modify the case template itself
    // - Never auto-adjust trust weights below a floor (0.1)
    // - Never auto-change routing for security-review capability
    String target = request.target();
    if (target != null && target.contains("security-review")) {
      return true;
    }
    return false;
  }
}
```

## 8. Devtown RegressionEvaluator

Devtown needs a regression evaluator so the `RegressionDetector` can assess whether health degradation after an improvement constitutes a regression in devtown's domain:

```java
@ApplicationScoped
public class DevtownRegressionEvaluator implements RegressionEvaluator {

  @Override public String evaluatorId() { return "devtown-health-delta"; }
  @Override public String domainId() { return "devtown"; }

  @Override
  public RegressionVerdict evaluate(UUID caseId, HealthScoreSnapshot baseline,
      HealthScoreSnapshot current, String category) {
    double delta = current.score() - baseline.score();
    if (delta < -0.1) {
      return new RegressionVerdict.Detected(Math.abs(delta),
          "Health score dropped by " + String.format("%.1f%%", Math.abs(delta) * 100)
          + " after " + category + " improvement");
    }
    return new RegressionVerdict.NoRegression();
  }
}
```

The evaluator uses a simple delta threshold (-10% health score drop). This is intentionally conservative — devtown improvements are configuration changes, not code changes, so regressions should be rare and obvious. The threshold can be tuned via configuration.

## 9. Test Strategy

### Devtown unit tests

| Test class | What it covers |
|------------|---------------|
| `CiReliabilityCapabilityAreaTest` | Health score computation from CI pass rates, flaky detection, duration trends |
| `ReviewQualityCapabilityAreaTest` | Health score from finding accuracy, cycle time, missed-issue rate |
| `MergeQueueHealthCapabilityAreaTest` | Health score from queue depth, batch success, throughput, time-to-merge |
| `ReviewerTrustCapabilityAreaTest` | Health score from calibration accuracy, decline rate, load balance |
| `SlaComplianceCapabilityAreaTest` | Health score from on-time rate, escalation rate, backlog, calibration |
| `DevtownCategoryProviderTest` | 5 categories registered, 5 stages with correct ordering and gate checkpoints |
| `DevtownProposalSourceTest` | Proposals generated when health scores drop below thresholds, no proposals when healthy |
| `DevtownConflictStrategyTest` | Same-category conflicts detected, cross-category improvements are clear |
| `DevtownDenyPatternProviderTest` | Security-review target denied, other targets allowed |
| `DevtownRegressionEvaluatorTest` | Regression detected at -10% delta, no regression above threshold, boundary values |
| `DevtownEvolutionCaseResolverTest` | Resolves singleton case by template ID, throws if missing |
| `DevtownEvolutionEnricherTest` | Stream targets enriched with devtown context |
| `DevtownEvolutionBootstrapTest` | Creates case on first startup, skips if already exists |

### Devtown integration tests

| Test class | What it covers |
|------------|---------------|
| `DevtownEvolutionApiIntegrationTest` | Full API surface: state query returns enriched snapshot, mutations delegate correctly |
| `DevtownEvolutionTabIntegrationTest` | Evolution tab renders with workbench, fetches data from /api/devtown/evolution |
| `DevtownCapabilityAreaRegistrationTest` | All 5 devtown areas discovered by CDI alongside engine defaults, health score computed |
| `DevtownEvolutionTickTest` | Ticker runs with devtown areas, generates proposals when health degrades, respects devtown conflict strategy |

### Critical test scenarios

1. **End-to-end feedback loop:** CI pass rate drops → CiReliabilityCapabilityArea health degrades → DevtownProposalSource generates a gate-tightening proposal → proposal passes through filtering → conductor inbox entry appears → approve → configuration applied → health recovers
2. **Area coexistence:** Devtown's 5 areas + engine's 10 defaults all registered → HealthScoreTracker computes weighted aggregate → devtown areas dominate because engine areas return neutral for devtown cases
3. **Category isolation:** Devtown categories don't conflict with code-evolution categories → two domain's improvements run independently
4. **Singleton case:** Bootstrap creates case once → restart finds existing case → Evolution tab shows persisted state
5. **Enrichment correctness:** Raw ImprovementStreamView target "reviewer:agent-security" → enriched to human-readable display string with trust score context

## 10. Module Placement

### Engine (casehubio/engine) — minimal changes

| Component | Module | Notes |
|-----------|--------|-------|
| No new production code expected | — | SPIs already complete from #1148 |

### Devtown (casehubio/devtown) — main work

| Component | Module | Package |
|-----------|--------|---------|
| `CiReliabilityCapabilityArea` | `domain` | `io.casehub.devtown.domain.evolution` |
| `ReviewQualityCapabilityArea` | `domain` | `io.casehub.devtown.domain.evolution` |
| `MergeQueueHealthCapabilityArea` | `domain` | `io.casehub.devtown.domain.evolution` |
| `ReviewerTrustCapabilityArea` | `domain` | `io.casehub.devtown.domain.evolution` |
| `SlaComplianceCapabilityArea` | `domain` | `io.casehub.devtown.domain.evolution` |
| `DevtownCategoryProvider` | `domain` | `io.casehub.devtown.domain.evolution` |
| `DevtownProposalSource` | `domain` | `io.casehub.devtown.domain.evolution` |
| `DevtownConflictStrategy` | `domain` | `io.casehub.devtown.domain.evolution` |
| `DevtownDenyPatternProvider` | `domain` | `io.casehub.devtown.domain.evolution` |
| `DevtownRegressionEvaluator` | `domain` | `io.casehub.devtown.domain.evolution` |
| `DevtownEvolutionApi` | `app` | `io.casehub.devtown.app.evolution` |
| `DevtownEvolutionResource` | `app` | `io.casehub.devtown.app.evolution` |
| `DevtownEvolutionCaseResolver` | `app` | `io.casehub.devtown.app.evolution` |
| `DevtownEvolutionEnricher` | `app` | `io.casehub.devtown.app.evolution` |
| `DevtownEvolutionBootstrap` | `app` | `io.casehub.devtown.app.evolution` |
| `DevtownEvolutionStateSnapshot` | `app` | `io.casehub.devtown.app.evolution` |
| `DevtownImprovementStreamView` | `app` | `io.casehub.devtown.app.evolution` |
| `devtown-evolution.yaml` | `app` | `src/main/resources/templates/` |
| `evolution.ts` | `app/webui` | `src/views/` |
| `devtown-pr-impact-panel.ts` | `app/webui` | `src/components/` |
| `index.ts` (modified) | `app/webui` | Tab registration |

### Blocks-ui (casehubio/blocks-ui) — no changes expected

Existing `evolution-workbench` and `evolution-config` components are sufficient (D6).

## References

- `CapabilityArea.java` — existing multi-instance SPI pattern
- `AbstractCapabilityArea.java` — base class for capability areas
- `ImprovementCategoryProvider` SPI — #1148 design spec §1.1
- `ImprovementProposalSource` SPI — #1148 design spec §1.2
- `ConflictStrategy` SPI — #1148 design spec §1.4
- `DenyPatternProvider` SPI — #1148 design spec §1.5
- `DefaultEngineEvolutionApi` — engine rest module, API surface (#1132 design spec §6)
- `blocks-evolution-workbench` — blocks-ui component, TabDefinition extension point
- `blocks-evolution-config` — blocks-ui component, EvolutionApi client
- devtown `index.ts` — registerPanel/hostPanel pattern
- devtown `GovernanceService` — existing domain data source
- devtown `MergeQueueService` — queue state and batch outcomes
- devtown `DevtownTrustRoutingPolicyProvider` — trust routing data
- devtown `SlaCalibrationService` — SLA calibration data
- `2026-09-21-command-centre-conductor-design.md` — 5-layer architecture
- `2026-09-23-generalise-evolution-conductor-design.md` — pluggable SPIs
- Epic #1149 Phase 4 — UI component plan
- Decisions D1–D7 in `decisions.md`
