# Design: AgentRoutingStrategy SPI + Trust-Weighted Routing
**Branch:** `issue-371-337-336-spi-trust`
**Issues:** engine#371 (CaseLifecycleEvent), engine#337 (AgentRoutingStrategy), engine#336 (TrustWeightedAgentStrategy)
**Date:** 2026-05-27

---

## Summary

Three related improvements that collectively clean up a borrowed abstraction, establish an
engine-owned routing SPI, and connect trust scoring from the ledger to agent selection.

1. **#371** — Promote `CaseLifecycleEvent` from `internal.event` to `spi.event` (package rename only).
2. **#337** — Replace borrowed `LeastLoadedStrategy`/`WorkBroker` from casehub-work with an
   engine-owned `AgentRoutingStrategy` SPI. Removes `casehub-work-api` and `casehub-work-core`
   from the engine runtime dependency graph.
3. **#336** — Implement `TrustWeightedAgentStrategy` in `casehub-engine-ledger` using a
   `TrustScoreCache` fed by existing ledger CDI events. Full trust maturity model phases 0–3
   (ledger#76 closed, CAPABILITY_DIMENSION scores available).

---

## Issue #371 — CaseLifecycleEvent Package Promotion

### Decision

Move `CaseLifecycleEvent` from `io.casehub.engine.internal.event` to
`io.casehub.engine.spi.event`, staying in the `common` module. The `spi.*` namespace
already exists in `common` (persistence SPIs). No module change, no API change — pure rename.

### Why `common` not `api`

`common` already has `io.casehub.engine.spi.*` packages. `CaseLifecycleEvent` is a pure Java
record with no framework annotations (CDI, JPA, or Quarkus). Staying in `common` avoids
touching module POMs and is consistent with other event types in that module (`EventLog`).

### Affected files

| File | Change |
|------|--------|
| `common/.../internal/event/CaseLifecycleEvent.java` | Moved (package rename) |
| `runtime/.../handler/CaseStartedEventHandler.java` | Import updated |
| `runtime/.../handler/CaseStatusChangedHandler.java` | Import updated |
| `runtime/.../handler/GoalReachedEventHandler.java` | Import updated |
| `runtime/.../handler/MilestoneReachedEventHandler.java` | Import updated |
| `runtime/.../handler/WorkflowExecutionCompletedHandler.java` | Import updated |
| `runtime/.../handler/SignalReceivedEventHandler.java` | Import updated |
| `scheduler-quartz/.../QuartzWorkerExecutionJobListener.java` | Import updated |
| `ledger/.../service/CaseLedgerEventCapture.java` | Import updated |
| `ledger/.../CaseLedgerEventCaptureTest.java` | Import updated |

`casehub-clinical` imports the internal package (clinical#28) — their import fix is mechanical
once this ships.

**Build note (GE-20260526-43a51d):** After the move, use `mvn clean test-compile -pl
common,runtime,scheduler-quartz,ledger -am` — not incremental — to catch any stale class
files that miss the package change.

---

## Issue #337 — AgentRoutingStrategy SPI

### Root cause

`WorkOrchestrator` and `CaseContextChangedEventHandler` borrow seven types from
`casehub-work-api`/`casehub-work-core` for agent routing: `WorkBroker`, `WorkloadProvider`,
`LeastLoadedStrategy` (concrete — the interface `WorkerSelectionStrategy` is not directly
imported), `WorkerCandidate`, `SelectionContext`, `AssignmentDecision`, `AssignmentTrigger`.
All are designed for human task (WorkItem) routing. Evidence of the mismatch:

- `SelectionContext` is constructed with 7 of 8 fields null in both call sites, and the two
  call sites pass `capability.getName()` in **different** fields (category vs requiredCapabilities)
- `AssignmentTrigger.CREATED` is semantically wrong — agent workers are triggered by context change
- `WorkerCandidate.activeWorkItemCount` uses WorkItem counts for Quartz job slots
- `CasehubWorkloadProvider` wraps `WorkerExecutionManager` (Quartz job counts) in a
  `WorkloadProvider` interface from casehub-work — correct semantics, wrong abstraction
- CDI ambiguity: `JpaWorkloadProvider` (from casehub-work-core on transitive classpath) plus
  `CasehubWorkloadProvider` both implement `WorkloadProvider` → `StubWorkloadProvider` stubs
  in both `resilience/test` and `work-adapter/test` exist solely to suppress this

### New types in `casehub-engine-api`

```java
public interface AgentRoutingStrategy {
    AgentAssignment select(AgentRoutingContext context, List<AgentCandidate> candidates);
}

// caseId per spi-case-id-parameter.md protocol
// caseContext omitted — neither current strategy uses it; add JsonNode when engine#376 is implemented
public record AgentRoutingContext(
    UUID caseId,
    String capabilityName)

// AgentHealth is engine-api — no casehub-eidos-api leak into api tier
public enum AgentHealth { READY, EPISTEMICALLY_WEAK, DEGRADED }

// Unavailable workers are filtered before candidate list is built; never appear here
// capabilities = worker's full capability set (all declared capabilities, not just matched one)
public record AgentCandidate(
    String workerId,
    Set<String> capabilities,
    int runningJobs,       // Quartz job count via WorkerExecutionManager — semantically correct
    AgentHealth health)    // pre-probed; EPISTEMICALLY_WEAK/DEGRADED available for scoring

public record AgentAssignment(String workerId) {
    public static AgentAssignment noOp() { return new AgentAssignment(null); }
    public boolean isNoOp() { return workerId == null; }
}
```

`AgentHealth` is an engine-api enum so `casehub-engine-api` has no dependency on
`casehub-eidos-api`. The runtime caller maps `CapabilityStatus` → `AgentHealth` before
building candidates.

### Default implementation in engine runtime

```java
@DefaultBean
@ApplicationScoped
public class LeastLoadedAgentStrategy implements AgentRoutingStrategy {
    public AgentAssignment select(AgentRoutingContext ctx, List<AgentCandidate> candidates) {
        return candidates.stream()
            .min(Comparator.comparingInt(AgentCandidate::runningJobs))
            .map(c -> new AgentAssignment(c.workerId()))
            .orElse(AgentAssignment.noOp());
    }
}
```

Ties broken by list order (deterministic). Health-based preference demotion is left to
`TrustWeightedAgentStrategy` via the `health` field on `AgentCandidate`.

Per `engine-spi-noops-defaultbean.md`: add `LeastLoadedAgentStrategy` to the beans table
in that protocol at implementation time.

### Candidate construction

**Current state before this change:**
- `WorkOrchestrator.buildCandidates()` calls `capabilityHealth.probe()` with `ProbeContext.of(null)`
  and filters `Unavailable` workers.
- `CaseContextChangedEventHandler.publishWorkerSchedule()` does NOT call `capabilityHealth.probe()`
  at all — candidates are built with no health check.

**After this change — both call sites use the same pattern:**

```java
List<AgentCandidate> candidates = workers.stream()
    .filter(w -> w.hasCapability(capabilityName))
    .map(w -> {
        // Workers without an AgentDescriptor skip the probe — NoOpCapabilityHealth returns Ready
        CapabilityStatus cs = capabilityHealth.probe(
            w.agentDescriptor(), capabilityName, ProbeContext.of(caseInstance.getUuid()));
        // Intentional improvement: pass caseId to ProbeContext (was null in WorkOrchestrator)
        return switch (cs) {
            case Unavailable u        -> null;  // hard filter — excluded from candidates
            case Ready r              -> new AgentCandidate(w.getName(), w.capabilityNames(), runningJobs(w), READY);
            case EpistemicallyWeak ew -> new AgentCandidate(w.getName(), w.capabilityNames(), runningJobs(w), EPISTEMICALLY_WEAK);
            case Degraded d           -> new AgentCandidate(w.getName(), w.capabilityNames(), runningJobs(w), DEGRADED);
        };
    })
    .filter(Objects::nonNull)
    .toList();
```

`w.capabilityNames()` = the worker's full set of declared capabilities (not just the one being
matched). `runningJobs(w)` calls `executionManager.getActiveWorkCount(w.getName())` directly.

Adding `capabilityHealth.probe()` to `CaseContextChangedEventHandler` is a **behaviour change**,
not a pure refactor. Workers that are Unavailable will now be excluded from choreography routing
for the first time. This is the correct behaviour — it closes the parity gap between the two
routing paths.

### Files removed

| File | Reason |
|------|--------|
| `CasehubWorkloadProvider.java` (runtime) | `WorkerExecutionManager` injected directly in `LeastLoadedAgentStrategy` |
| `StubWorkloadProvider.java` (resilience/test) | CDI ambiguity gone; `CasehubWorkloadProvider` deleted |
| `StubWorkloadProvider.java` (work-adapter/test) | Same reason — `CasehubWorkloadProvider` deleted eliminates the ambiguity |

### pom.xml changes

| Module | Change |
|--------|--------|
| `runtime/pom.xml` | Remove `casehub-work-api`, `casehub-work-core` |
| `resilience/pom.xml` | Remove `casehub-work-testing` (test scope) |
| `ledger/pom.xml` | Add `casehub-engine-api` — `TrustWeightedAgentStrategy` implements `AgentRoutingStrategy` |
| `work-adapter/pom.xml` | Unchanged — legitimate bridge to casehub-work |

---

## Issue #336 — TrustWeightedAgentStrategy

### TrustScoreCache

Lives in `casehub-engine-ledger`. Annotated `@Startup` (Quarkus eager init) to force
`@PostConstruct` hydration at application startup on the main thread — not on first routing
call on a Vert.x IO thread, which would block the event loop.

`TrustScoreRoutingPublisher` fires events synchronously (`fire()`, not `fireAsync()`). Its
javadoc is explicit: "TrustScoreFullPayload and TrustScoreDeltaPayload use only fire() because
no async observers exist for them." Additionally, `resolveObserverMethods()` returns only sync
observers, so `@ObservesAsync` cache methods would not be detected by the publisher's observer
check → `hasFullObservers = false` → publisher skips firing entirely. Cache observers must use
`@Observes` (synchronous).

`TrustScoreDeltaPayload` carries `(actorId, previousScore, newScore, previousGlobalScore,
newGlobalScore)` — GLOBAL scores only, no `capabilityKey`/`dimensionKey`. Only
`TrustScoreFullPayload` carries structured `ActorTrustScore` records needed for CAPABILITY
and CAPABILITY_DIMENSION updates. `onDelta` is a no-op.

```java
@Startup
@ApplicationScoped
public class TrustScoreCache {

    // key: "actorId:capabilityKey"
    private final ConcurrentHashMap<String, CachedCapabilityScore> capabilityScores =
        new ConcurrentHashMap<>();

    // key: "actorId:capabilityKey:dimensionKey"
    private final ConcurrentHashMap<String, Double> capabilityDimensionScores =
        new ConcurrentHashMap<>();

    record CachedCapabilityScore(double trustScore, int decisionCount) {}

    @PostConstruct
    void hydrate() {
        trustRepo.findAll().forEach(this::index);
    }

    void onFull(@Observes TrustScoreFullPayload p) {
        p.scores().forEach(this::index);
    }

    // Delta carries only GLOBAL scores (no capabilityKey/dimensionKey) — no-op for cache
    void onDelta(@Observes TrustScoreDeltaPayload p) { /* no-op */ }

    private void index(ActorTrustScore s) {
        if (s.scoreType == CAPABILITY && s.capabilityKey != null) {
            capabilityScores.put(
                s.actorId + ":" + s.capabilityKey,
                new CachedCapabilityScore(s.trustScore, s.decisionCount));
        } else if (s.scoreType == CAPABILITY_DIMENSION
                   && s.capabilityKey != null && s.dimensionKey != null) {
            capabilityDimensionScores.put(
                s.actorId + ":" + s.capabilityKey + ":" + s.dimensionKey, s.trustScore);
        }
    }

    public OptionalDouble getCapabilityScore(String actorId, String capabilityKey) {
        CachedCapabilityScore s = capabilityScores.get(actorId + ":" + capabilityKey);
        return s != null ? OptionalDouble.of(s.trustScore()) : OptionalDouble.empty();
    }

    public int getDecisionCount(String actorId, String capabilityKey) {
        CachedCapabilityScore s = capabilityScores.get(actorId + ":" + capabilityKey);
        return s != null ? s.decisionCount() : 0;
    }

    public OptionalDouble getCapabilityDimensionScore(
            String actorId, String capabilityKey, String dimensionKey) {
        Double v = capabilityDimensionScores.get(
            actorId + ":" + capabilityKey + ":" + dimensionKey);
        return v != null ? OptionalDouble.of(v) : OptionalDouble.empty();
    }
}
```

**Identity assumption — must validate at implementation time:** `TrustScoreCache` looks up
scores using `workerId` (= `Worker.getName()` from the YAML definition) as the `actorId` key.
This assumes that `ActorTrustScore.actorId` values in the ledger match worker names exactly.
Before wiring trust routing, verify that `TrustScoreJob` populates `actorId` with the same
string format as worker names. If the namespaces diverge, every lookup returns `empty()` and
the system silently falls to Phase 0 with no error.

### TrustRoutingPolicy

```java
public record TrustRoutingPolicy(
    double threshold,               // minimum CAPABILITY trust score for selection
    int minimumObservations,        // Phase 0/1 boundary — below = bootstrap routing
    double borderlineMargin,        // |score - threshold| ≤ margin → excluded (score 0.0)
    double blendFactor,             // trust weight (0.0 = pure workload, 1.0 = pure trust)
    Map<String, Double> qualityFloors)  // Phase 3: dimension → minimum quality score; empty = no floor checks
```

```java
public interface TrustRoutingPolicyProvider {
    TrustRoutingPolicy forCapability(String capabilityName);
}

@DefaultBean @ApplicationScoped
public class DefaultTrustRoutingPolicyProvider implements TrustRoutingPolicyProvider {
    // threshold=0.7, minimumObservations=10, borderlineMargin=0.1,
    // blendFactor=0.6, qualityFloors=empty
    public TrustRoutingPolicy forCapability(String capabilityName) { return DEFAULT; }
}
```

Deployments override with `@Alternative @Priority(1)`. Devtown's `DevtownCapabilityRegistry`
can implement this interface to expose its per-capability policies.

### TrustWeightedAgentStrategy — scoring model

```java
@ApplicationScoped @Alternative @Priority(1)
public class TrustWeightedAgentStrategy implements AgentRoutingStrategy {

    public AgentAssignment select(AgentRoutingContext ctx, List<AgentCandidate> candidates) {
        TrustRoutingPolicy policy = policyProvider.forCapability(ctx.capabilityName());

        // ScoredCandidate is a private record: (AgentCandidate candidate, double score)
        return candidates.stream()
            .map(c -> new ScoredCandidate(c, score(c, ctx.capabilityName(), policy)))
            .max(Comparator.comparingDouble(ScoredCandidate::score))
            .filter(sc -> sc.score() > 0.0)
            .map(sc -> new AgentAssignment(sc.candidate().workerId()))
            .orElse(AgentAssignment.noOp());
    }

    private double score(AgentCandidate c, String capability, TrustRoutingPolicy policy) {
        OptionalDouble capScore = cache.getCapabilityScore(c.workerId(), capability);

        // Phase 0: no CAPABILITY history → availability routing (Gastown parity)
        if (capScore.isEmpty()) return availabilityScore(c);

        // Phase 1: insufficient observations → availability routing
        if (cache.getDecisionCount(c.workerId(), capability) < policy.minimumObservations())
            return availabilityScore(c);

        double t = capScore.getAsDouble();

        // Phase 2a: borderline → excluded (score 0.0)
        if (Math.abs(t - policy.threshold()) <= policy.borderlineMargin()) return 0.0;

        // Phase 2b: below threshold → excluded
        if (t < policy.threshold()) return 0.0;

        // Phase 3: quality floor checks (CAPABILITY_DIMENSION)
        for (Map.Entry<String, Double> floor : policy.qualityFloors().entrySet()) {
            OptionalDouble quality =
                cache.getCapabilityDimensionScore(c.workerId(), capability, floor.getKey());
            // Data exists and below floor → excluded
            if (quality.isPresent() && quality.getAsDouble() < floor.getValue()) return 0.0;
            // No data yet for this dimension → graceful Phase 0; not penalised
        }

        // All checks passed: blend trust score with workload efficiency
        return t * policy.blendFactor() + workloadScore(c) * (1.0 - policy.blendFactor());
    }

    private double availabilityScore(AgentCandidate c) { return workloadScore(c); }
    private double workloadScore(AgentCandidate c) { return 1.0 / (1.0 + c.runningJobs()); }
}
```

**Mixed Phase 0 / Phase 2 pool policy (explicit):** a Phase 0 candidate (no trust history,
`availabilityScore > 0`) will always outscore a Phase 2 borderline candidate (score 0.0). This
is intentional: a new agent with no track record is preferable to one the system has identified
as borderline. New agents start with a clean slate; borderline agents have demonstrated
uncertainty. Phase 1 candidates (sparse history) also score > 0 via `availabilityScore` and
will outcompete borderline Phase 2 candidates by the same logic.

**Full-exclusion path:** when all candidates score 0.0 (all borderline or all below threshold),
`noOp()` is returned. The engine falls to its no-worker path. Escalation to human oversight
when all candidates are borderline is tracked in engine#377.

---

## Deferred Issues Filed

| Issue | Repo | What |
|-------|------|------|
| casehub-work#231 | casehub-work | `ClaimFirstStrategy` → `@Alternative @Priority(0)` |
| engine#376 | engine | `SemanticAgentRoutingStrategy` (add `JsonNode caseContext` to `AgentRoutingContext` at that point) |
| engine#377 | engine | Borderline agent escalation |
| parent#80 | parent | Engine deep-dive doc sync |

---

## Protocol Compliance

| Protocol | Status |
|----------|--------|
| `spi-case-id-parameter.md` | ✅ `AgentRoutingContext` carries `UUID caseId` |
| `engine-spi-noops-defaultbean.md` | ✅ `LeastLoadedAgentStrategy` is `@DefaultBean`; add to beans table at implementation |
| `trust-maturity-model.md` | ✅ Phases 0–3 implemented; Phase 0 = Gastown parity |
| `module-tier-structure.md` | ✅ `AgentRoutingStrategy` types in `api` (Tier 1); cache in `ledger` (Tier 3) |
| `no-workarounds-fix-the-design.md` | ✅ Root cause fixed (borrowed abstraction), not patched |
| CDI `@Alternative @Priority(1)` | ✅ Standard CDI priority pattern: `@DefaultBean` yields to `@ApplicationScoped`; `@Alternative @Priority(1)` beats both |