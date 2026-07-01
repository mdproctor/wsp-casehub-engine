# Bootstrap Escalation Required — Design Spec

**Issue:** casehubio/engine#415
**Branch:** `issue-415-bootstrap-fallback-type`
**Date:** 2026-06-05
**Status:** approved (revised)

---

## Problem

`TrustWeightedAgentStrategy` and `SemanticAgentRoutingStrategy` both score BOOTSTRAP candidates
by workload (`1/(1+runningJobs)`). When the available agent pool for a high-stakes capability
contains no QUALIFIED agents (e.g. all BOOTSTRAP, or BOOTSTRAP + BORDERLINE), the BOOTSTRAP
candidate wins by workload score. For `merge-executor`, `security-review`, and `architecture-review`,
assigning an unproven agent is irreversible if wrong.

The mixed-pool gap makes this worse: a pool of [BOOTSTRAP (0 jobs, score=1.0), BORDERLINE (score=0.0)]
assigns the BOOTSTRAP agent — the borderline path in `decide()` requires all candidates to score
0.0 before firing, and BOOTSTRAP's positive workload score prevents that.

`DevtownCapabilityRegistry` already declares a `fallbackType` per capability but `TrustRoutingPolicy`
carries no corresponding guard field, so enforcement is structurally impossible.

The Qhorus trust gate (`DevtownObligorTrustPolicy`) cannot fix this: it correctly exempts BOOTSTRAP
agents (no observations → permit), and `ObligorTrustContext` carries no capability tag.

---

## Design

### 1. `TrustRoutingPolicy` — add `bootstrapEscalationRequired`

```java
public record TrustRoutingPolicy(
    double threshold,
    int minimumObservations,
    double borderlineMargin,
    double blendFactor,
    Map<String, Double> qualityFloors,
    boolean bootstrapEscalationRequired   // NEW
) {
    public static final TrustRoutingPolicy DEFAULT =
        new TrustRoutingPolicy(0.7, 10, 0.1, 0.6, Map.of(), false);
    // isBootstrap/isBorderline/passesThresholdCheck unchanged
}
```

`bootstrapEscalationRequired = true` means: "this capability must never be handled by an agent
in BOOTSTRAP phase; require human oversight if no QUALIFIED agent is available."

`false` (the default) preserves the current behaviour: BOOTSTRAP candidates compete by workload
and can be assigned. Most capabilities are unaffected.

Following `trust-maturity-model.md`, this field belongs in code (a domain design decision), not
YAML config (operational tuning).

### 2. `EscalationReason` — new top-level type

A top-level enum in `api/spi/routing/` (not nested inside `AgentAssignment`), used by both
`AgentAssignment` and `AgentRoutingEscalationEvent` without coupling the event to the assignment.

```java
package io.casehub.api.spi.routing;

public enum EscalationReason {
    /** All candidates scored 0.0 and at least one was BORDERLINE. */
    BORDERLINE_STALEMATE,

    /** No QUALIFIED agent is available; only BOOTSTRAP-phase agents could be assigned.
     *  Requires human routing. Pre-screen fires before scoring, so no scoring has occurred. */
    NO_QUALIFIED_AGENT
}
```

`BORDERLINE_STALEMATE` replaces `ALL_BORDERLINE` — the existing condition is "any borderline in
an all-zero-scored pool," not "all candidates are borderline." The old name was misleading.

`NO_QUALIFIED_AGENT` names the bootstrap condition by what it actually means for the pool: there
is no trust-qualified agent available for this capability.

### 3. `AgentAssignment` — add reason to `EscalateToOversight`

```java
record EscalateToOversight(String capabilityName, EscalationReason reason)
    implements AgentAssignment {}

// Factory (replaces single-arg form — all callers updated):
static AgentAssignment escalate(String capabilityName, EscalationReason reason) {
    return new EscalateToOversight(capabilityName, reason);
}
```

`capabilityName` remains the actual capability name for both reasons.

`TrustCandidateClassifier.decide()` updated — only change is the reason label:
```java
return anyBorderline
    ? AgentAssignment.escalate(capabilityName, EscalationReason.BORDERLINE_STALEMATE)
    : AgentAssignment.unresolvable();
```

No other changes to `TrustCandidateClassifier`. No `applyBootstrapGuard()` method — the
bootstrap guard is the strategies' responsibility, not the classifier's.

### 4. Bootstrap guard in both strategies — pre-screen + stripping

Both `TrustWeightedAgentStrategy` and `SemanticAgentRoutingStrategy` get identical two-part
bootstrap logic inserted after `classify()`.

**Part 1 — pre-screen (fires before `emitOn(workerPool)` in semantic strategy):**

```java
if (policy.bootstrapEscalationRequired()) {
    boolean hasQualified = classified.stream().anyMatch(c -> c.phase() == Phase.QUALIFIED);
    boolean hasBootstrap = classified.stream().anyMatch(c -> c.phase() == Phase.BOOTSTRAP);
    if (!hasQualified && hasBootstrap) {
        return Uni.createFrom().item(
            AgentAssignment.escalate(context.capabilityName(), EscalationReason.NO_QUALIFIED_AGENT));
    }
}
```

**Part 2 — strip BOOTSTRAP from scoring (only reached if QUALIFIED exists):**

```java
final List<ClassifiedCandidate> eligible = policy.bootstrapEscalationRequired()
    ? classified.stream().filter(c -> c.phase() != Phase.BOOTSTRAP).toList()
    : classified;
// use eligible (not classified) in the scoring loop
```

The stripping approach means:
- BOOTSTRAP candidates are invisible to scoring when the flag is set
- QUALIFIED agents compete among themselves — workload comparison is only between QUALIFIED
- A busy QUALIFIED agent is preferred over an idle BOOTSTRAP agent, which is the intent of the flag
- `decide()` is called with `eligible` (not `classified`). The `anyBorderline` check inside `decide()` then only inspects candidates that were actually eligible, which is semantically correct — a BOOTSTRAP candidate that was stripped is not contributing to a borderline stalemate.

**In `SemanticAgentRoutingStrategy`:** `eligible` is computed before `emitOn(workerPool)`. The
lambda captures `eligible` and only pays embedding cost for agents that can actually win. If the
pre-screen fires, `emitOn()` is never entered.

**Pool coverage (with `bootstrapEscalationRequired = true`):**

| Pool | Pre-screen fires? | Outcome |
|------|-------------------|---------|
| [BOOTSTRAP, BOOTSTRAP] | Yes — no QUALIFIED | `NO_QUALIFIED_AGENT` |
| [BOOTSTRAP, BORDERLINE] | Yes — no QUALIFIED | `NO_QUALIFIED_AGENT` |
| [BOOTSTRAP, EXCLUDED_*] | Yes — no QUALIFIED | `NO_QUALIFIED_AGENT` |
| [BOOTSTRAP, QUALIFIED] | No — QUALIFIED exists | Strip BOOTSTRAP; QUALIFIED wins |
| [BOOTSTRAP, QUALIFIED, BORDERLINE] | No — QUALIFIED exists | Strip BOOTSTRAP; QUALIFIED wins |
| [BORDERLINE only] | No — no BOOTSTRAP | `BORDERLINE_STALEMATE` (existing path via `decide()`) |
| [QUALIFIED only] | No — no BOOTSTRAP | Existing behaviour unchanged |
| Flag = false | Pre-screen skipped | All existing behaviour unchanged |

### 5. `AgentRoutingEscalationEvent` — carry reason

```java
public record AgentRoutingEscalationEvent(
    UUID caseId, String capabilityName, String bindingName, EscalationReason reason) {}
```

Updated Javadoc: describes both escalation conditions (BORDERLINE_STALEMATE and NO_QUALIFIED_AGENT).
All three construction sites updated.

### 6. `AgentRoutingEscalationHandler` — message per reason + metric log

The `[METRIC:escalation.no-qualified-agent]` log fires unconditionally at the top of `handle()`,
before the channel lookup. This ensures the metric fires even when no oversight channel is open
(the scenario where the alert is most critical — bootstrap guard fired AND no channel to route to).

```java
@ConsumeEvent(value = EventBusAddresses.AGENT_ROUTING_ESCALATION, blocking = true)
public void handle(final AgentRoutingEscalationEvent event) {
    // Metric log fires unconditionally — before channel search
    if (event.reason() == EscalationReason.NO_QUALIFIED_AGENT) {
        LOG.warnf(
            "[METRIC:escalation.no-qualified-agent] caseId=%s capability=%s binding=%s"
                + " — bootstrap guard fired; no trust-qualified agent available.",
            event.caseId(), event.capabilityName(), event.bindingName());
    }

    final String oversightName = CaseChannel.oversightChannelName(event.caseId());
    final List<CaseChannel> channels = channelProvider.listChannels(event.caseId());
    channels.stream()
        .filter(c -> oversightName.equals(c.name()))
        .findFirst()
        .ifPresentOrElse(
            channel -> postQuery(channel, event),
            () -> LOG.warnf("[METRIC:escalation.no-oversight-channel] ..."));
}

private void postQuery(final CaseChannel channel, final AgentRoutingEscalationEvent event) {
    final String message = switch (event.reason()) {
        case BORDERLINE_STALEMATE ->
            String.format(
                "All agent candidates for capability '%s' (binding: '%s') are borderline."
                    + " Human oversight required: please select an agent or approve the next"
                    + " best available agent.",
                event.capabilityName(), event.bindingName());
        case NO_QUALIFIED_AGENT ->
            String.format(
                "No trust-qualified agent is available for capability '%s' (binding: '%s')."
                    + " Routing policy requires an agent with established trust history."
                    + " Human routing required.",
                event.capabilityName(), event.bindingName());
    };
    channelProvider.postToChannel(channel, "casehub-engine", message, MessageType.QUERY, null, null);
}
```

Existing `[METRIC:escalation.no-oversight-channel]` log unchanged.

### 7. Response path (engine#383)

`NO_QUALIFIED_AGENT` and `BORDERLINE_STALEMATE` follow identical response handling: human posts
via the oversight channel, re-triggering routing. No differentiation by reason is needed in
the response path. Engine#383 covers both.

---

## Test Plan

### `TrustWeightedAgentStrategyTest` — new cases + updates to existing

New tests:
- `bootstrap_noQualified_allBootstrap_escalatesNoQualifiedAgent()` — pool all-BOOTSTRAP, flag on → `NO_QUALIFIED_AGENT`
- `bootstrap_noQualified_bootstrapPlusBorderline_escalatesNoQualifiedAgent()` — pre-screen closes mixed-pool gap
- `bootstrap_noQualified_bootstrapPlusExcluded_escalatesNoQualifiedAgent()` — same gap, EXCLUDED variant
- `bootstrap_qualifiedExists_bootstrapStripped_qualifiedAssigned()` — BOOTSTRAP idle, QUALIFIED busy → QUALIFIED wins
- `bootstrap_qualifiedExists_bootstrapStripped_busyQualifiedWinsOverIdleBootstrap()` — explicit: flag overrides workload comparison
- `bootstrap_qualifiedExists_bootstrapPlusBorderline_qualifiedWins_noBorderlineStalemate()` — [BOOTSTRAP, QUALIFIED, BORDERLINE] with flag on → BOOTSTRAP stripped, BORDERLINE remains in eligible but QUALIFIED wins; verifies `BORDERLINE_STALEMATE` does not fire when QUALIFIED exists in stripped pool
- `bootstrap_flagFalse_allBootstrap_assignsByWorkload()` — flag off → existing behaviour preserved

Updates to existing:
- All `EscalateToOversight` assertions add `.reason() == EscalationReason.BORDERLINE_STALEMATE`
- All `new TrustRoutingPolicy(...)` constructions add `false` as 6th arg (or named test policy)

### `SemanticAgentRoutingStrategyTest` — same set of new cases

Mirror of the `TrustWeightedAgentStrategyTest` bootstrap cases.

### `TrustCandidateClassifierTest`

- Update existing `EscalateToOversight` assertion to check `reason == BORDERLINE_STALEMATE`
- All `new TrustRoutingPolicy(...)` constructions updated

### Runtime tests

- `AgentRoutingEscalationHandlerTest` — update event construction to include reason; add two
  `NO_QUALIFIED_AGENT` sub-cases:
  - Channel open: metric log fires AND QUERY is posted (message matches `NO_QUALIFIED_AGENT` text)
  - No channel open: metric log fires AND QUERY is NOT posted (this is the regression test for the metric placement decision — the log must fire even when the channel is absent)
  - Also add message assertion for `BORDERLINE_STALEMATE`
- `DefaultWorkOrchestratorTest` — `AgentAssignment.escalate("analyse", EscalationReason.BORDERLINE_STALEMATE)`
- `CaseContextChangedEventHandlerRoutingTest` — same update

### Devtown side (devtown#62)

Engine-side tests verify the mechanism in isolation. Devtown#62 must include two end-to-end tests:
- **Escalation path:** `merge-executor` with `bootstrapEscalationRequired = true`, all-BOOTSTRAP pool → `NO_QUALIFIED_AGENT` escalation fires
- **Stripping path:** `merge-executor` with `bootstrapEscalationRequired = true`, [BOOTSTRAP + QUALIFIED] pool → QUALIFIED wins, no escalation (covers the more common production scenario where a trusted agent exists)

---

## Files Changed

| File | Module | Change |
|------|--------|--------|
| `EscalationReason.java` | api | New top-level enum: `BORDERLINE_STALEMATE`, `NO_QUALIFIED_AGENT` |
| `TrustRoutingPolicy.java` | api | Add `boolean bootstrapEscalationRequired`, update DEFAULT |
| `AgentAssignment.java` | api | Add reason to `EscalateToOversight`, update factory to 2-arg |
| `AgentRoutingEscalationEvent.java` | common | Add `EscalationReason reason` field, update Javadoc |
| `TrustCandidateClassifier.java` | ledger | Update `decide()` escalate call to use `BORDERLINE_STALEMATE` |
| `TrustWeightedAgentStrategy.java` | ledger | Add pre-screen + BOOTSTRAP stripping |
| `SemanticAgentRoutingStrategy.java` | engine-ai | Add pre-screen + BOOTSTRAP stripping (before `emitOn`) |
| `CaseContextChangedEventHandler.java` | runtime | Update `AgentRoutingEscalationEvent` construction (add reason) |
| `DefaultWorkOrchestrator.java` | runtime | Update `AgentRoutingEscalationEvent` construction (add reason) |
| `AgentRoutingEscalationHandler.java` | runtime | Switch message per reason, add metric log |
| Test files (×6) | ledger, engine-ai, runtime | New bootstrap tests + reason assertion updates |