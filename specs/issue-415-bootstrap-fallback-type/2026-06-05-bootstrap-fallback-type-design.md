# Bootstrap Fallback Type — Design Spec

**Issue:** casehubio/engine#415
**Branch:** `issue-415-bootstrap-fallback-type`
**Date:** 2026-06-05
**Status:** approved

---

## Problem

`TrustWeightedAgentStrategy` and `SemanticAgentRoutingStrategy` both score BOOTSTRAP candidates
by workload (`1/(1+runningJobs)`). When every available agent for a high-stakes capability is in
BOOTSTRAP phase (no trust history), the highest-workload agent gets assigned. For `merge-executor`,
`security-review`, and `architecture-review`, this is irreversible if wrong.

`DevtownCapabilityRegistry` already declares a `fallbackType` per capability. The devtown domain
model (`RoutingPolicy.fallbackType`) has anticipated this since Layer 2. But `TrustRoutingPolicy`
in engine-api carries no such field, so the guard can never fire.

The `DevtownObligorTrustPolicy` (Qhorus trust gate) cannot prevent this: it correctly exempts
BOOTSTRAP agents (no observations → permit). `ObligorTrustContext` carries no capability tag,
making a capability-specific bootstrap gate at the Qhorus layer structurally impossible.

---

## Design

### 1. `TrustRoutingPolicy` — add `bootstrapFallbackType`

```java
public record TrustRoutingPolicy(
    double threshold,
    int minimumObservations,
    double borderlineMargin,
    double blendFactor,
    Map<String, Double> qualityFloors,
    Optional<String> bootstrapFallbackType   // NEW: absence = no guard; presence = escalate on all-BOOTSTRAP
) {
    public static final TrustRoutingPolicy DEFAULT =
        new TrustRoutingPolicy(0.7, 10, 0.1, 0.6, Map.of(), Optional.empty());
    // isBootstrap/isBorderline/passesThresholdCheck unchanged
}
```

`bootstrapFallbackType` is `Optional<String>` — the string is the oversight type name
(e.g. `"HumanOversight.ROUTING_REVIEW"`) that devtown configures. The engine uses the field
as a guard flag (presence = escalate); the string value is domain documentation and available
for future type-aware routing. Following `trust-maturity-model.md`, this field belongs in code
(a domain design decision), not YAML config (operational tuning).

Absence of `bootstrapFallbackType` preserves the current behavior: BOOTSTRAP candidates are
scored by workload and assigned normally. Most capabilities are unaffected.

### 2. `AgentAssignment` — add `EscalationReason`

The current `EscalateToOversight(String capabilityName)` type carries no information about
_why_ escalation was triggered. The `AgentRoutingEscalationHandler` currently generates the
message "all agent candidates for capability '%s' are **borderline**" for all escalation
events — semantically wrong when the actual condition is all-BOOTSTRAP.

```java
public enum EscalationReason { ALL_BORDERLINE, ALL_BOOTSTRAP }

record EscalateToOversight(String capabilityName, EscalationReason reason)
    implements AgentAssignment {}

// Factory (replaces single-arg form — callers updated):
static AgentAssignment escalate(String capabilityName, EscalationReason reason) {
    return new EscalateToOversight(capabilityName, reason);
}
```

`capabilityName` remains the actual capability name for both reasons — consistent with the
existing borderline path and with what the escalation handler uses to format the human-readable
QUERY message. The `bootstrapFallbackType` string from the policy is NOT threaded through
`AgentAssignment` — the capability name is what identifies the routing problem.

### 3. `AgentRoutingEscalationEvent` — carry reason

```java
public record AgentRoutingEscalationEvent(
    UUID caseId, String capabilityName, String bindingName, AgentAssignment.EscalationReason reason)
{}
```

All three construction sites updated (two in `CaseContextChangedEventHandler`, one in
`DefaultWorkOrchestrator`).

### 4. `TrustCandidateClassifier` — `applyBootstrapGuard()` shared method

The guard belongs in `TrustCandidateClassifier`, not in each strategy, because:
- Both `TrustWeightedAgentStrategy` and `SemanticAgentRoutingStrategy` use the same
  classify → score → decide pipeline
- Both have the same all-BOOTSTRAP vulnerability
- The classifier is already the shared routing-decision authority
- One implementation, no duplication

```java
/**
 * Checks whether all classified candidates are BOOTSTRAP and the policy has a
 * bootstrapFallbackType configured. If so, returns an escalation assignment.
 * Call after classify(), before scoring.
 */
public Optional<AgentAssignment> applyBootstrapGuard(
    List<ClassifiedCandidate> classified,
    TrustRoutingPolicy policy,
    String capabilityName) {

    if (policy.bootstrapFallbackType().isPresent()
            && classified.stream().allMatch(c -> c.phase() == Phase.BOOTSTRAP)) {
        return Optional.of(
            AgentAssignment.escalate(capabilityName, AgentAssignment.EscalationReason.ALL_BOOTSTRAP));
    }
    return Optional.empty();
}
```

`decide()` updated for `ALL_BORDERLINE`:
```java
return anyBorderline
    ? AgentAssignment.escalate(capabilityName, EscalationReason.ALL_BORDERLINE)
    : AgentAssignment.unresolvable();
```

### 5. `TrustWeightedAgentStrategy.select()` — guard call site

```java
final List<ClassifiedCandidate> classified =
    classifier.classify(candidates, context.capabilityName(), policy, cache);

// Bootstrap guard — before scoring
Optional<AgentAssignment> guard =
    classifier.applyBootstrapGuard(classified, policy, context.capabilityName());
if (guard.isPresent()) {
    return Uni.createFrom().item(guard.get());
}

// ... existing scoring loop unchanged
```

### 6. `SemanticAgentRoutingStrategy.select()` — same guard call site

Identical pattern: after `classify()`, before `scored` list construction.

### 7. `AgentRoutingEscalationHandler` — accurate message per reason

```java
private void postQuery(final CaseChannel channel, final AgentRoutingEscalationEvent event) {
    final String message = switch (event.reason()) {
        case ALL_BORDERLINE ->
            String.format("All agent candidates for capability '%s' (binding: '%s') are borderline." +
                " Human oversight required: please select an agent or approve the next" +
                " best available agent.", event.capabilityName(), event.bindingName());
        case ALL_BOOTSTRAP ->
            String.format("All agent candidates for capability '%s' (binding: '%s') are in" +
                " bootstrap phase — no trust history exists for this capability." +
                " Human routing decision required: please designate an agent to handle this task.",
                event.capabilityName(), event.bindingName());
    };
    channelProvider.postToChannel(channel, "casehub-engine", message, MessageType.QUERY, null, null);
}
```

---

## Test Plan

### `TrustCandidateClassifierTest` — new + updated

- `applyBootstrapGuard_allBootstrap_withFallbackType_escalates()`
- `applyBootstrapGuard_allBootstrap_noFallbackType_returnsEmpty()`
- `applyBootstrapGuard_mixedBootstrapAndQualified_returnsEmpty()`
- `allBorderline_escalates_withAllBorderlineReason()` (existing test updated to check reason)

### `TrustWeightedAgentStrategyTest` — new + updated

- `bootstrapGuard_allBootstrap_withFallbackType_escalatesWithBootstrapReason()`
- `bootstrapGuard_allBootstrap_noFallbackType_assignsByWorkload()`
- `bootstrapGuard_mixedBootstrapAndQualified_qualifiedWins_guardDoesNotFire()`
- All existing `EscalateToOversight` assertions updated to check reason (`ALL_BORDERLINE`)
- All `TrustRoutingPolicy` constructions updated to include `Optional.empty()` as 6th arg

### `TrustCandidateClassifierTest` and `SemanticAgentRoutingStrategyTest`

- Update `EscalateToOversight` assertions to check `reason == ALL_BORDERLINE`

### Runtime tests updated

- `DefaultWorkOrchestratorTest` — `AgentAssignment.escalate("analyse", ALL_BORDERLINE)`
- `CaseContextChangedEventHandlerRoutingTest` — same
- `AgentRoutingEscalationHandlerTest` — update event construction, add reason-specific message assertions

---

## Why the All-BOOTSTRAP Check Is Correct

- **QUALIFIED beats BOOTSTRAP:** BOOTSTRAP candidates score by workload (> 0); QUALIFIED candidates
  score by blended trust+workload. A QUALIFIED candidate with any non-zero blended score beats a
  BOOTSTRAP candidate. The guard fires only when NO QUALIFIED candidates exist.
- **BORDERLINE + BOOTSTRAP mixed pool:** The existing `decide()` logic assigns the BOOTSTRAP
  candidate (positive workload score beats borderline's 0.0 score). The guard fires ONLY if
  `allMatch(BOOTSTRAP)` — a single non-BOOTSTRAP candidate disables it. This is correct: if a
  borderline candidate exists, the existing all-borderline escalation path handles it if all
  non-bootstrap are borderline.
- **Empty candidates:** handled before `classify()` is called; guard is never reached.

---

## Devtown Side (Separate PR — engine#415 blocks devtown#62)

`DevtownTrustRoutingPolicyProvider.forCapability()` passes `routingPolicy.fallbackType()` as
`bootstrapFallbackType` when constructing `TrustRoutingPolicy` for high-stakes capabilities
(`merge-executor`, `security-review`, `architecture-review`). No new strategy class or provider
needed in devtown.

---

## Files Changed

| File | Module | Change |
|------|--------|--------|
| `TrustRoutingPolicy.java` | api | Add `Optional<String> bootstrapFallbackType` field, update DEFAULT |
| `AgentAssignment.java` | api | Add `EscalationReason` enum, add reason to `EscalateToOversight`, update factory |
| `AgentRoutingEscalationEvent.java` | common | Add `EscalationReason reason` field |
| `TrustCandidateClassifier.java` | ledger | Add `applyBootstrapGuard()`, update `decide()` to use `ALL_BORDERLINE` |
| `TrustWeightedAgentStrategy.java` | ledger | Add guard call after `classify()` |
| `SemanticAgentRoutingStrategy.java` | engine-ai | Add guard call after `classify()` |
| `CaseContextChangedEventHandler.java` | runtime | Update `handleEscalation()` + `AgentRoutingEscalationEvent` construction |
| `DefaultWorkOrchestrator.java` | runtime | Update `AgentRoutingEscalationEvent` construction |
| `AgentRoutingEscalationHandler.java` | runtime | Switch message per reason |
| Test files (×7) | ledger, engine-ai, runtime | New tests + update existing assertions |
