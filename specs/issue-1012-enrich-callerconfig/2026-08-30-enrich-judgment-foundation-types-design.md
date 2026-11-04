# Enrich Judgment Foundation Types

**Issue:** engine#1012
**Date:** 2026-08-30
**Status:** Approved

## Context

The judgment foundation types landed in #1009 with minimal fields. The abandoned v2 branch (`issue-994-governed-yield-v2`, commit `6464141f`) had significantly richer versions needed for real escalation and judgment scenarios. This issue ports v2's field richness to main's types, with no backward compatibility — pre-release, no external consumers.

## Changes

### 1. CallerConfig (sealed interface, `api/spi/judgment/`)

**CallerConfig.Human** — 2 fields → 12:

```java
record Human(
    @Nullable CandidateSetSpec candidateGroups,
    @Nullable CandidateSetSpec candidateUsers,
    @Nullable String title,
    @Nullable ExpressionEvaluator titleExpression,
    @Nullable Set<String> outcomes,
    @Nullable Integer claimDeadlineHours,
    @Nullable String scope,
    @Nullable ExpressionEvaluator scopeExpression,
    @Nullable String priority,
    @Nullable String templateRef,
    @Nullable Class<?> payloadType,
    @Nullable QuorumConfig quorum) implements CallerConfig {
  public Human {
    if (outcomes != null) outcomes = Set.copyOf(outcomes);
  }
}
```

Removes `minimumTrustLevel` (trust threshold lives on `JudgmentTarget.trustThreshold()`). Replaces `List<String> candidateGroups` with `CandidateSetSpec` for dynamic evaluation support. All fields nullable — escalation may only need a subset.

**CallerConfig.Llm** — 1 field → 3:

```java
record Llm(
    @Nullable String modelId,
    @Nullable String modelName,
    @Nullable String systemPrompt) implements CallerConfig {}
```

Keeps `modelId` (main's name, more self-documenting than v2's `model`). Adds `modelName` (specific model identifier, e.g. `claude-sonnet-4-20250514`) and `systemPrompt` (needed by `LlmJudgmentPhase` in blocks#221).

**CallerConfig.A2A** — 2 fields → 3:

```java
record A2A(String endpoint, @Nullable String skill, boolean streaming) implements CallerConfig {
  public A2A(String endpoint) { this(endpoint, null, false); }
  public A2A(String endpoint, @Nullable String skill) { this(endpoint, skill, false); }
}
```

Adds `streaming` boolean (default false). Convenience constructors preserve existing 1-arg and 2-arg call sites.

**CallerConfig.Any** — unchanged.

### 2. CallerIdentity (`api/spi/judgment/`)

Required fields, add `trustScore`:

```java
record CallerIdentity(String callerId, String callerType, @Nullable Double trustScore) {
  CallerIdentity {
    Objects.requireNonNull(callerId, "callerId required");
    Objects.requireNonNull(callerType, "callerType required");
  }
  static CallerIdentity of(String callerId, String callerType) {
    return new CallerIdentity(callerId, callerType, null);
  }
  static CallerIdentity of(String callerId, String callerType, @Nullable Double trustScore) {
    return new CallerIdentity(callerId, callerType, trustScore);
  }
}
```

Removes `anonymous()`. If no caller identity is known, the field is null at the container level (VerificationContext, EscalationContext).

### 3. Evidence (`api/spi/judgment/`)

Full v2 alignment:

```java
record Evidence(String name, EvidenceType type, String content, @Nullable String ref) {
  Evidence {
    Objects.requireNonNull(name, "name required");
    Objects.requireNonNull(type, "type required");
    Objects.requireNonNull(content, "content required");
  }
  static Evidence of(String name, EvidenceType type, String content) {
    return new Evidence(name, type, content, null);
  }
}
```

Renames: `key`→`name`, `value:Object`→`content:String`. Adds `ref` for external evidence references. Removes the 3-arg `of(key, type, value)` factory — replaced by the new signature.

### 4. JudgmentTarget (`api/model/`)

Add `maxEscalationAttempts`:

```java
private final int maxEscalationAttempts;  // default 3 in Builder
```

Builder:
```java
private int maxEscalationAttempts = 3;
public Builder maxEscalationAttempts(int max) { this.maxEscalationAttempts = max; return this; }
```

Per-target escalation budget — different judgment bindings may need different limits. Complements `CaseDefinition.maxEscalations` (case-level default).

### 5. VerificationContext (`api/spi/judgment/`)

Clean record — remove raw string identity fields and untyped evidence map:

```java
record VerificationContext(
    UUID caseId,
    String tenancyId,
    String bindingName,
    JudgmentTarget target,
    Map<String, Object> inputData,
    @Nullable CaseDefinition definition,
    String decision,
    List<Evidence> evidence,
    @Nullable CallerIdentity callerIdentity,
    @Nullable Duration responseTime) {}
```

Removes: `callerId` (String), `callerType` (String), `typedEvidence` rename to `evidence`. The untyped `Map<String, Object> evidence` is replaced by `List<Evidence>`.

### 6. EscalationContext (`api/spi/judgment/`)

Same cleanup:

```java
record EscalationContext(
    UUID caseId,
    String tenancyId,
    String bindingName,
    JudgmentTarget target,
    String decision,
    List<Evidence> evidence,
    VerificationResult verificationResult,
    int escalationCount,
    int maxEscalations,
    CaseDefinition definition,
    @Nullable CallerIdentity callerIdentity,
    @Nullable Duration responseTime) {}
```

Removes: `callerId`, `callerType`, backward-compat 12-arg constructor. Replaces `Map<String, Object> evidence` with `List<Evidence>`.

### 7. Deprecation annotations

Add `@Deprecated(forRemoval = true)` to:
- `HumanTaskTarget` class
- `HumanTaskScheduler` interface
- `HumanTaskScheduleRequest` record
- `CloudEventHumanTaskScheduler` class

Javadoc `@deprecated Use {@link JudgmentTarget} and {@link JudgmentScheduler} instead.`

### 8. Consumer updates

All construction sites must be updated. Key files:

| File | Module | Change |
|------|--------|--------|
| `EvidencePresenceVerifier` | runtime | Evidence field renames (`name`, `content`) |
| `DefaultJudgmentEscalator` | runtime | CallerIdentity required fields, maxEscalationAttempts from target |
| `JudgmentCompletedHandler` | runtime | VerificationContext/EscalationContext new signatures |
| `CloudEventJudgmentScheduler` | work-cloudevent | Evidence serialization field names |
| `CallerConfigTest` | api/test | New Human fields, Llm fields, A2A streaming |
| `EvidenceTypesTest` | api/test | Evidence field renames |
| `CallerIdentityTest` | api/test | Required fields, trustScore |
| `JudgmentTargetTest` | api/test | maxEscalationAttempts |
| `JudgmentTargetDispatchTest` | runtime/test | maxEscalationAttempts in fixtures |
| `EvidencePresenceVerifierTest` | runtime/test | Evidence field renames |

### 9. CLAUDE.md update

Update the "Judgment Foundation Types" section to reflect:
- CallerConfig.Human field list
- CallerConfig.Llm fields
- CallerConfig.A2A streaming
- CallerIdentity required fields + trustScore
- Evidence renamed fields + ref
- JudgmentTarget.maxEscalationAttempts
- VerificationContext/EscalationContext cleaned records
- HumanTaskTarget/HumanTaskScheduler deprecation

## Testing

- Existing `CallerConfigTest`, `CallerIdentityTest`, `EvidenceTypesTest` updated for new fields
- Existing `JudgmentTargetTest` updated for `maxEscalationAttempts`
- Existing `EvidencePresenceVerifierTest` updated for Evidence field renames
- Existing `JudgmentTargetDispatchTest` — verify dispatch still works with enriched target
- No new test classes needed — all changes are field additions/renames on existing types

## References

- engine#1012 — issue body with field-by-field mapping
- v2 branch `issue-994-governed-yield-v2`, commit `6464141f` — source for CallerConfig, CallerIdentity, Evidence field designs
- `api/src/main/java/io/casehub/api/spi/judgment/CallerConfig.java` — current sealed interface
- `api/src/main/java/io/casehub/api/spi/judgment/CallerIdentity.java` — current record
- `api/src/main/java/io/casehub/api/spi/judgment/Evidence.java` — current record
- `api/src/main/java/io/casehub/api/model/JudgmentTarget.java` — current target class
- `api/src/main/java/io/casehub/api/spi/judgment/VerificationContext.java` — backward-compat constructor to remove
- `api/src/main/java/io/casehub/api/spi/judgment/EscalationContext.java` — backward-compat constructor to remove
