# Enrich Judgment Foundation Types — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #1012 — Enrich CallerConfig, CallerIdentity, Evidence — port v2 field richness
**Issue group:** #1012

**Goal:** Port the richer field sets from the abandoned v2 branch to main's judgment foundation types, clean up backward-compat constructors, and deprecate HumanTaskTarget/HumanTaskScheduler.

**Architecture:** All type changes are in `api` module (`io.casehub.api.spi.judgment` and `io.casehub.api.model`). Consumer updates in `runtime` module. No new files created — all modifications to existing records, sealed interfaces, and their tests. No backward compatibility — all consumers updated in the same commit.

**Tech Stack:** Java 21 records, sealed interfaces, jspecify nullability annotations

## Global Constraints

- Pre-release — no backward compatibility required
- All `@Nullable` annotations use `org.jspecify.annotations.Nullable`
- Defensive copying on mutable collections (Set.copyOf, List.copyOf)
- No Flyway migrations (type-only changes)

---

## Batch 1: Foundation type enrichment + all consumers

### Task 1: Enrich CallerConfig sealed interface

**Files:**
- Modify: `api/src/main/java/io/casehub/api/spi/judgment/CallerConfig.java`
- Modify: `api/src/test/java/io/casehub/api/spi/judgment/CallerConfigTest.java`

**Interfaces:**
- Produces: `CallerConfig.Human` with 12 nullable fields, `CallerConfig.Llm` with 3 fields, `CallerConfig.A2A` with 3 fields, `CallerConfig.human()` convenience factory

- [ ] **Step 1: Write failing tests for enriched CallerConfig**

Replace `CallerConfigTest.java` entirely:

```java
package io.casehub.api.spi.judgment;

import static org.junit.jupiter.api.Assertions.*;

import io.casehub.api.spi.QuorumConfig;
import io.casehub.api.spi.routing.CandidateSetSpec;
import io.casehub.api.spi.routing.StaticSetStrategy;
import java.util.Set;
import org.junit.jupiter.api.Test;

class CallerConfigTest {

  @Test
  void humanAllFields() {
    var groups = CandidateSetSpec.inline(StaticSetStrategy.of(Set.of("managers")));
    var users = CandidateSetSpec.inline(StaticSetStrategy.of(Set.of("user-1")));
    var human = new CallerConfig.Human(
        groups, users, "Review needed", null,
        Set.of("APPROVE", "REJECT"), 24, "case",
        null, "high", "tmpl-1", String.class,
        QuorumConfig.majority(3));
    assertEquals(groups, human.candidateGroups());
    assertEquals(users, human.candidateUsers());
    assertEquals("Review needed", human.title());
    assertEquals(Set.of("APPROVE", "REJECT"), human.outcomes());
    assertEquals(24, human.claimDeadlineHours());
    assertEquals("high", human.priority());
    assertEquals("tmpl-1", human.templateRef());
    assertEquals(String.class, human.payloadType());
    assertNotNull(human.quorum());
  }

  @Test
  void humanConvenienceFactory() {
    var human = CallerConfig.human();
    assertNull(human.candidateGroups());
    assertNull(human.candidateUsers());
    assertNull(human.title());
    assertNull(human.outcomes());
    assertNull(human.quorum());
  }

  @Test
  void humanOutcomesDefensiveCopy() {
    var mutable = new java.util.HashSet<>(Set.of("A", "B"));
    var human = new CallerConfig.Human(
        null, null, null, null, mutable, null, null, null, null, null, null, null);
    mutable.add("C");
    assertEquals(2, human.outcomes().size());
  }

  @Test
  void llmAllFields() {
    var llm = new CallerConfig.Llm("anthropic", "claude-sonnet-4-20250514", "You are a judge.");
    assertEquals("anthropic", llm.modelId());
    assertEquals("claude-sonnet-4-20250514", llm.modelName());
    assertEquals("You are a judge.", llm.systemPrompt());
  }

  @Test
  void llmNoArgs() {
    var llm = new CallerConfig.Llm();
    assertNull(llm.modelId());
    assertNull(llm.modelName());
    assertNull(llm.systemPrompt());
  }

  @Test
  void a2aWithStreaming() {
    var a2a = new CallerConfig.A2A("https://agent.example.com", "review", true);
    assertTrue(a2a.streaming());
  }

  @Test
  void a2aConvenienceConstructors() {
    var a2a1 = new CallerConfig.A2A("https://agent.example.com");
    assertNull(a2a1.skill());
    assertFalse(a2a1.streaming());

    var a2a2 = new CallerConfig.A2A("https://agent.example.com", "review");
    assertEquals("review", a2a2.skill());
    assertFalse(a2a2.streaming());
  }

  @Test
  void anyConfig() {
    var any = new CallerConfig.Any();
    assertInstanceOf(CallerConfig.class, any);
  }

  @Test
  void sealedTypeExhaustiveness() {
    CallerConfig config = CallerConfig.human();
    String result = switch (config) {
      case CallerConfig.Human h -> "human:" + (h.candidateGroups() == null ? "default" : "custom");
      case CallerConfig.Llm l -> "llm:" + l.modelId();
      case CallerConfig.A2A a -> "a2a:" + a.endpoint();
      case CallerConfig.Any a -> "any";
    };
    assertEquals("human:default", result);
  }
}
```

- [ ] **Step 2: Run test — verify it fails**

Run: `mvn test -pl api -Dtest=CallerConfigTest -Dsurefire.failIfNoSpecifiedTests=false -q`
Expected: compilation failure — `human()` factory and new field constructors don't exist yet

- [ ] **Step 3: Implement enriched CallerConfig**

Replace `CallerConfig.java`:

```java
package io.casehub.api.spi.judgment;

import io.casehub.api.spi.QuorumConfig;
import io.casehub.api.spi.routing.CandidateSetSpec;
import io.casehub.platform.api.expression.ExpressionEvaluator;
import java.util.Set;
import org.jspecify.annotations.Nullable;

public sealed interface CallerConfig {

  static Human human() {
    return new Human(null, null, null, null, null, null, null, null, null, null, null, null);
  }

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

  record Llm(
      @Nullable String modelId,
      @Nullable String modelName,
      @Nullable String systemPrompt) implements CallerConfig {
    public Llm() { this(null, null, null); }
  }

  record A2A(String endpoint, @Nullable String skill, boolean streaming) implements CallerConfig {
    public A2A(String endpoint) { this(endpoint, null, false); }
    public A2A(String endpoint, @Nullable String skill) { this(endpoint, skill, false); }
  }

  record Any() implements CallerConfig {}
}
```

- [ ] **Step 4: Run test — verify it passes**

Run: `mvn test -pl api -Dtest=CallerConfigTest -q`
Expected: all tests PASS

- [ ] **Step 5: Commit**

```
feat(#1012): enrich CallerConfig — Human 12 fields, Llm 3 fields, A2A streaming

Refs #1012
```

### Task 2: Enrich CallerIdentity and Evidence

**Files:**
- Modify: `api/src/main/java/io/casehub/api/spi/judgment/CallerIdentity.java`
- Modify: `api/src/main/java/io/casehub/api/spi/judgment/Evidence.java`
- Modify: `api/src/test/java/io/casehub/api/spi/judgment/CallerIdentityTest.java`
- Modify: `api/src/test/java/io/casehub/api/spi/judgment/EvidenceTypesTest.java`

**Interfaces:**
- Produces: `CallerIdentity(String callerId, String callerType, @Nullable Double trustScore)` with required callerId/callerType
- Produces: `Evidence(String name, EvidenceType type, String content, @Nullable String ref)` with required name/type/content

- [ ] **Step 1: Write failing tests for enriched CallerIdentity and Evidence**

Replace `CallerIdentityTest.java`:

```java
package io.casehub.api.spi.judgment;

import static org.junit.jupiter.api.Assertions.*;

import org.junit.jupiter.api.Test;

class CallerIdentityTest {

  @Test
  void ofFactory() {
    var id = CallerIdentity.of("user-42", "human");
    assertEquals("user-42", id.callerId());
    assertEquals("human", id.callerType());
    assertNull(id.trustScore());
  }

  @Test
  void ofFactoryWithTrustScore() {
    var id = CallerIdentity.of("agent-1", "a2a", 0.85);
    assertEquals("agent-1", id.callerId());
    assertEquals("a2a", id.callerType());
    assertEquals(0.85, id.trustScore());
  }

  @Test
  void requiredFieldsRejectNull() {
    assertThrows(NullPointerException.class, () -> new CallerIdentity(null, "human", null));
    assertThrows(NullPointerException.class, () -> new CallerIdentity("user-1", null, null));
  }
}
```

Update `EvidenceTypesTest.java` evidence factory test:

```java
// Replace the evidenceFactory test:
@Test
void evidenceFactory() {
  var ev = Evidence.of("confidence_score", EvidenceType.METRIC, "0.95");
  assertEquals("confidence_score", ev.name());
  assertEquals(EvidenceType.METRIC, ev.type());
  assertEquals("0.95", ev.content());
  assertNull(ev.ref());
}

@Test
void evidenceWithRef() {
  var ev = new Evidence("doc", EvidenceType.DOCUMENT, "report text", "https://example.com/report");
  assertEquals("doc", ev.name());
  assertEquals("report text", ev.content());
  assertEquals("https://example.com/report", ev.ref());
}

@Test
void evidenceRequiredFieldsRejectNull() {
  assertThrows(NullPointerException.class, () -> new Evidence(null, EvidenceType.METRIC, "val", null));
  assertThrows(NullPointerException.class, () -> new Evidence("key", null, "val", null));
  assertThrows(NullPointerException.class, () -> new Evidence("key", EvidenceType.METRIC, null, null));
}
```

- [ ] **Step 2: Run tests — verify they fail**

Run: `mvn test -pl api -Dtest="CallerIdentityTest,EvidenceTypesTest" -q`
Expected: compilation failure — `trustScore()`, `name()`, `content()`, `ref()` don't exist yet

- [ ] **Step 3: Implement enriched CallerIdentity**

Replace `CallerIdentity.java`:

```java
package io.casehub.api.spi.judgment;

import java.util.Objects;
import org.jspecify.annotations.Nullable;

public record CallerIdentity(String callerId, String callerType, @Nullable Double trustScore) {

  public CallerIdentity {
    Objects.requireNonNull(callerId, "callerId required");
    Objects.requireNonNull(callerType, "callerType required");
  }

  public static CallerIdentity of(String callerId, String callerType) {
    return new CallerIdentity(callerId, callerType, null);
  }

  public static CallerIdentity of(String callerId, String callerType, @Nullable Double trustScore) {
    return new CallerIdentity(callerId, callerType, trustScore);
  }
}
```

- [ ] **Step 4: Implement enriched Evidence**

Replace `Evidence.java`:

```java
package io.casehub.api.spi.judgment;

import java.util.Objects;
import org.jspecify.annotations.Nullable;

public record Evidence(String name, EvidenceType type, String content, @Nullable String ref) {

  public Evidence {
    Objects.requireNonNull(name, "name required");
    Objects.requireNonNull(type, "type required");
    Objects.requireNonNull(content, "content required");
  }

  public static Evidence of(String name, EvidenceType type, String content) {
    return new Evidence(name, type, content, null);
  }
}
```

- [ ] **Step 5: Run tests — verify they pass**

Run: `mvn test -pl api -Dtest="CallerIdentityTest,EvidenceTypesTest" -q`
Expected: all tests PASS

- [ ] **Step 6: Commit**

```
feat(#1012): enrich CallerIdentity (required fields + trustScore) and Evidence (name/content/ref)

Refs #1012
```

### Task 3: Clean VerificationContext, EscalationContext, add JudgmentTarget.maxEscalationAttempts

**Files:**
- Modify: `api/src/main/java/io/casehub/api/spi/judgment/VerificationContext.java`
- Modify: `api/src/main/java/io/casehub/api/spi/judgment/EscalationContext.java`
- Modify: `api/src/main/java/io/casehub/api/model/JudgmentTarget.java`
- Modify: `api/src/test/java/io/casehub/api/spi/judgment/VerificationContextTest.java`
- Modify: `api/src/test/java/io/casehub/api/spi/judgment/EscalationContextTest.java`
- Modify: `api/src/test/java/io/casehub/api/model/JudgmentTargetTest.java`

**Interfaces:**
- Consumes: `CallerIdentity` (required fields), `Evidence` (renamed fields)
- Produces: `VerificationContext` with `List<Evidence> evidence` (no Map, no raw strings), `EscalationContext` same cleanup, `JudgmentTarget.maxEscalationAttempts()` (int, default 3)

- [ ] **Step 1: Write failing tests**

Replace `VerificationContextTest.java`:

```java
package io.casehub.api.spi.judgment;

import static org.junit.jupiter.api.Assertions.*;

import io.casehub.api.model.JudgmentTarget;
import java.time.Duration;
import java.util.List;
import java.util.Map;
import java.util.UUID;
import org.junit.jupiter.api.Test;

class VerificationContextTest {

  private static final JudgmentTarget TARGET =
      JudgmentTarget.builder().prompt("Review this").build();

  @Test
  void fullConstructor() {
    var identity = CallerIdentity.of("agent-7", "a2a");
    var evidence = List.of(
        Evidence.of("score", EvidenceType.METRIC, "0.95"),
        Evidence.of("report", EvidenceType.DOCUMENT, "full report text"));
    var responseTime = Duration.ofSeconds(3);

    var ctx = new VerificationContext(
        UUID.randomUUID(), "tenant-1", "binding", TARGET,
        Map.of(), null, "approve", evidence,
        identity, responseTime);

    assertEquals(identity, ctx.callerIdentity());
    assertEquals(2, ctx.evidence().size());
    assertEquals("score", ctx.evidence().get(0).name());
    assertEquals(Duration.ofSeconds(3), ctx.responseTime());
  }

  @Test
  void nullCallerIdentityAndResponseTime() {
    var ctx = new VerificationContext(
        UUID.randomUUID(), "tenant-1", "binding", TARGET,
        Map.of(), null, "approve", List.of(),
        null, null);

    assertNull(ctx.callerIdentity());
    assertNull(ctx.responseTime());
    assertTrue(ctx.evidence().isEmpty());
  }
}
```

Replace `EscalationContextTest.java`:

```java
package io.casehub.api.spi.judgment;

import static org.junit.jupiter.api.Assertions.*;

import io.casehub.api.model.JudgmentTarget;
import java.time.Duration;
import java.util.List;
import java.util.UUID;
import org.junit.jupiter.api.Test;

class EscalationContextTest {

  private static final JudgmentTarget TARGET =
      JudgmentTarget.builder().prompt("Assess risk").build();

  @Test
  void fullConstructor() {
    var identity = CallerIdentity.of("llm-1", "llm");
    var evidence = List.of(Evidence.of("reasoning", EvidenceType.REASONING, "Because X"));
    var responseTime = Duration.ofMillis(450);

    var ctx = new EscalationContext(
        UUID.randomUUID(), "tenant-1", "binding", TARGET,
        "reject", evidence,
        new VerificationResult.TrustTooLow("high", "medium"),
        1, 3, null,
        identity, responseTime);

    assertEquals(identity, ctx.callerIdentity());
    assertEquals(1, ctx.evidence().size());
    assertEquals(Duration.ofMillis(450), ctx.responseTime());
    assertEquals(1, ctx.escalationCount());
    assertEquals(3, ctx.maxEscalations());
  }

  @Test
  void nullOptionalFields() {
    var ctx = new EscalationContext(
        UUID.randomUUID(), "tenant-1", "binding", TARGET,
        "approve", List.of(),
        new VerificationResult.InsufficientEvidence("missing docs", List.of("docs")),
        2, 5, null,
        null, null);

    assertNull(ctx.callerIdentity());
    assertNull(ctx.responseTime());
    assertEquals(2, ctx.escalationCount());
  }
}
```

Add test for JudgmentTarget.maxEscalationAttempts in `JudgmentTargetTest.java`:

```java
@Test
void maxEscalationAttemptsDefault() {
  var target = JudgmentTarget.builder().prompt("test").build();
  assertEquals(3, target.maxEscalationAttempts());
}

@Test
void maxEscalationAttemptsCustom() {
  var target = JudgmentTarget.builder().prompt("test").maxEscalationAttempts(10).build();
  assertEquals(10, target.maxEscalationAttempts());
}
```

- [ ] **Step 2: Run tests — verify they fail**

Run: `mvn test -pl api -Dtest="VerificationContextTest,EscalationContextTest,JudgmentTargetTest" -q`
Expected: compilation failure — new record signatures don't exist yet

- [ ] **Step 3: Implement clean VerificationContext**

Replace `VerificationContext.java`:

```java
package io.casehub.api.spi.judgment;

import io.casehub.api.model.CaseDefinition;
import io.casehub.api.model.JudgmentTarget;
import java.time.Duration;
import java.util.List;
import java.util.Map;
import java.util.UUID;
import org.jspecify.annotations.Nullable;

public record VerificationContext(
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

- [ ] **Step 4: Implement clean EscalationContext**

Replace `EscalationContext.java`:

```java
package io.casehub.api.spi.judgment;

import io.casehub.api.model.CaseDefinition;
import io.casehub.api.model.JudgmentTarget;
import java.time.Duration;
import java.util.List;
import java.util.UUID;
import org.jspecify.annotations.Nullable;

public record EscalationContext(
    UUID caseId,
    String tenancyId,
    String bindingName,
    JudgmentTarget target,
    String decision,
    List<Evidence> evidence,
    VerificationResult verificationResult,
    int escalationCount,
    int maxEscalations,
    @Nullable CaseDefinition definition,
    @Nullable CallerIdentity callerIdentity,
    @Nullable Duration responseTime) {}
```

- [ ] **Step 5: Add maxEscalationAttempts to JudgmentTarget**

Add to `JudgmentTarget.java`:

Field: `private final int maxEscalationAttempts;`
Constructor: `this.maxEscalationAttempts = builder.maxEscalationAttempts;`
Getter: `public int maxEscalationAttempts() { return maxEscalationAttempts; }`
Builder field: `private int maxEscalationAttempts = 3;`
Builder method: `public Builder maxEscalationAttempts(int max) { this.maxEscalationAttempts = max; return this; }`

- [ ] **Step 6: Run tests — verify they pass**

Run: `mvn test -pl api -Dtest="VerificationContextTest,EscalationContextTest,JudgmentTargetTest" -q`
Expected: all tests PASS

- [ ] **Step 7: Commit**

```
feat(#1012): clean VerificationContext/EscalationContext, add JudgmentTarget.maxEscalationAttempts

Remove backward-compat constructors, raw callerId/callerType fields, untyped
evidence map. List<Evidence> is the sole evidence carrier.

Refs #1012
```

### Task 4: Update runtime consumers

**Files:**
- Modify: `runtime/src/main/java/io/casehub/engine/internal/worker/EvidencePresenceVerifier.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/worker/DefaultJudgmentEscalator.java`
- Delete: `runtime/src/main/java/io/casehub/engine/internal/worker/DelegatingJudgmentScheduler.java`
- Modify: `runtime/src/test/java/io/casehub/engine/internal/worker/EvidencePresenceVerifierTest.java`
- Modify: `runtime/src/test/java/io/casehub/engine/internal/worker/DefaultJudgmentEscalatorTest.java`

**Interfaces:**
- Consumes: `VerificationContext.evidence()` → `List<Evidence>`, `CallerConfig.human()` factory, `EscalationContext` clean constructor

- [ ] **Step 1: Write failing tests for updated EvidencePresenceVerifier**

Replace `EvidencePresenceVerifierTest.java` — evidence is now `List<Evidence>`, verifier checks `Evidence.name()`:

```java
package io.casehub.engine.internal.worker;

import static org.assertj.core.api.Assertions.assertThat;

import io.casehub.api.model.JudgmentTarget;
import io.casehub.api.spi.judgment.Evidence;
import io.casehub.api.spi.judgment.EvidenceType;
import io.casehub.api.spi.judgment.VerificationContext;
import io.casehub.api.spi.judgment.VerificationResult;
import java.util.List;
import java.util.Map;
import java.util.UUID;
import org.junit.jupiter.api.Test;

class EvidencePresenceVerifierTest {

  private final EvidencePresenceVerifier verifier = new EvidencePresenceVerifier();

  @Test
  void allEvidencePresent_returnsAccepted() {
    var target = JudgmentTarget.builder()
        .prompt("test")
        .evidenceRequirements(List.of("riskScore", "rationale"))
        .build();
    var ctx = new VerificationContext(
        UUID.randomUUID(), "t", "b", target, Map.of(), null, "approve",
        List.of(
            Evidence.of("riskScore", EvidenceType.METRIC, "0.8"),
            Evidence.of("rationale", EvidenceType.REASONING, "low risk")),
        null, null);
    assertThat(verifier.verify(ctx)).isInstanceOf(VerificationResult.Accepted.class);
  }

  @Test
  void missingEvidence_returnsInsufficientEvidence() {
    var target = JudgmentTarget.builder()
        .prompt("test")
        .evidenceRequirements(List.of("riskScore", "rationale", "supportingDocs"))
        .build();
    var ctx = new VerificationContext(
        UUID.randomUUID(), "t", "b", target, Map.of(), null, "approve",
        List.of(Evidence.of("riskScore", EvidenceType.METRIC, "0.8")),
        null, null);
    var result = verifier.verify(ctx);
    assertThat(result).isInstanceOf(VerificationResult.InsufficientEvidence.class);
    var ie = (VerificationResult.InsufficientEvidence) result;
    assertThat(ie.missingKeys()).containsExactlyInAnyOrder("rationale", "supportingDocs");
  }

  @Test
  void emptyRequirements_returnsAccepted() {
    var target = JudgmentTarget.builder().prompt("test").build();
    var ctx = new VerificationContext(
        UUID.randomUUID(), "t", "b", target, Map.of(), null, "approve",
        List.of(), null, null);
    assertThat(verifier.verify(ctx)).isInstanceOf(VerificationResult.Accepted.class);
  }

  @Test
  void id_returnsEvidencePresence() {
    assertThat(verifier.id()).isEqualTo("evidence-presence");
  }
}
```

- [ ] **Step 2: Update EvidencePresenceVerifier implementation**

Replace the `verify` method in `EvidencePresenceVerifier.java`:

```java
@Override
public VerificationResult verify(VerificationContext context) {
  List<String> required = context.target().evidenceRequirements();
  if (required.isEmpty()) return new VerificationResult.Accepted();
  var presentNames = context.evidence().stream()
      .map(Evidence::name)
      .collect(java.util.stream.Collectors.toSet());
  List<String> missing = required.stream()
      .filter(key -> !presentNames.contains(key))
      .toList();
  if (missing.isEmpty()) return new VerificationResult.Accepted();
  return new VerificationResult.InsufficientEvidence(
      "Missing required evidence keys: " + missing, missing);
}
```

Add import: `import io.casehub.api.spi.judgment.Evidence;`

- [ ] **Step 3: Write failing tests for updated DefaultJudgmentEscalator**

Replace `DefaultJudgmentEscalatorTest.java`:

```java
package io.casehub.engine.internal.worker;

import static org.junit.jupiter.api.Assertions.*;

import io.casehub.api.model.JudgmentTarget;
import io.casehub.api.spi.judgment.CallerConfig;
import io.casehub.api.spi.judgment.EscalationContext;
import io.casehub.api.spi.judgment.EscalationDecision;
import io.casehub.api.spi.judgment.VerificationResult;
import java.util.List;
import java.util.UUID;
import org.junit.jupiter.api.Test;

class DefaultJudgmentEscalatorTest {

  private final DefaultJudgmentEscalator escalator = new DefaultJudgmentEscalator();

  @Test
  void idIsDefault() {
    assertEquals("default", escalator.id());
  }

  @Test
  void insufficientEvidenceReYields() {
    var ctx = buildContext(
        new VerificationResult.InsufficientEvidence("need docs", List.of("docs")), 0, 5);
    var decision = escalator.escalate(ctx);

    assertInstanceOf(EscalationDecision.ReYield.class, decision);
    assertEquals("need docs", ((EscalationDecision.ReYield) decision).feedback());
  }

  @Test
  void trustTooLowEscalates() {
    var ctx = buildContext(new VerificationResult.TrustTooLow("high", "medium"), 0, 5);
    var decision = escalator.escalate(ctx);

    assertInstanceOf(EscalationDecision.Escalate.class, decision);
    var esc = (EscalationDecision.Escalate) decision;
    assertInstanceOf(CallerConfig.Human.class, esc.newCallerConfig());
    assertTrue(esc.reason().contains("Trust level too low"));
  }

  @Test
  void rejectedFaults() {
    var ctx = buildContext(new VerificationResult.Rejected("invalid judgment"), 0, 5);
    var decision = escalator.escalate(ctx);

    assertInstanceOf(EscalationDecision.Fault.class, decision);
    assertTrue(((EscalationDecision.Fault) decision).reason().contains("rejected"));
  }

  @Test
  void maxEscalationsFaults() {
    var ctx = buildContext(
        new VerificationResult.InsufficientEvidence("need docs", List.of()), 5, 5);
    var decision = escalator.escalate(ctx);

    assertInstanceOf(EscalationDecision.Fault.class, decision);
    assertTrue(((EscalationDecision.Fault) decision).reason().contains("Max escalations"));
  }

  @Test
  void belowMaxEscalationsStillEscalates() {
    var ctx = buildContext(new VerificationResult.TrustTooLow("high", "low"), 4, 5);
    var decision = escalator.escalate(ctx);

    assertInstanceOf(EscalationDecision.Escalate.class, decision);
  }

  private static EscalationContext buildContext(
      VerificationResult result, int escalationCount, int maxEscalations) {
    return new EscalationContext(
        UUID.randomUUID(), "tenant-1", "review-binding",
        JudgmentTarget.builder().prompt("Review this").build(),
        "approve", List.of(),
        result, escalationCount, maxEscalations,
        null, null, null);
  }
}
```

- [ ] **Step 4: Update DefaultJudgmentEscalator implementation**

In `DefaultJudgmentEscalator.java`, update the TrustTooLow case to use the convenience factory:

```java
case VerificationResult.TrustTooLow ttl ->
    new EscalationDecision.Escalate(
        CallerConfig.human(),
        "Trust level too low — escalating to human with minimum trust: "
            + ttl.requiredLevel());
```

Remove the `import java.util.List;` if no longer needed.

- [ ] **Step 5: Delete DelegatingJudgmentScheduler**

Use `ide_refactor_safe_delete` to remove `runtime/src/main/java/io/casehub/engine/internal/worker/DelegatingJudgmentScheduler.java`.

- [ ] **Step 6: Run runtime tests**

Run: `mvn test -pl runtime -Dtest="EvidencePresenceVerifierTest,DefaultJudgmentEscalatorTest" -q`
Expected: all tests PASS

- [ ] **Step 7: Commit**

```
feat(#1012): update runtime consumers — EvidencePresenceVerifier, DefaultJudgmentEscalator

Delete DelegatingJudgmentScheduler (replaced by NoOpJudgmentScheduler + CloudEventJudgmentScheduler).
EvidencePresenceVerifier checks List<Evidence> by name.
DefaultJudgmentEscalator uses CallerConfig.human() convenience factory.

Refs #1012
```

### Task 5: Deprecation annotations + CLAUDE.md

**Files:**
- Modify: `api/src/main/java/io/casehub/api/model/HumanTaskTarget.java` (annotation only)
- Modify: `common/src/main/java/io/casehub/engine/common/spi/HumanTaskScheduler.java` (annotation only)
- Modify: `common/src/main/java/io/casehub/engine/common/spi/HumanTaskScheduleRequest.java` (annotation only)
- Modify: `work-cloudevent/src/main/java/io/casehub/engine/work/cloudevent/CloudEventHumanTaskScheduler.java` (annotation only)
- Modify: `CLAUDE.md`

- [ ] **Step 1: Add deprecation annotations**

Add `@Deprecated(forRemoval = true)` to class/interface declarations:
- `HumanTaskTarget` class declaration
- `HumanTaskScheduler` interface declaration
- `HumanTaskScheduleRequest` record declaration
- `CloudEventHumanTaskScheduler` class declaration

Add javadoc `@deprecated Use {@link JudgmentTarget} and {@link io.casehub.engine.common.spi.JudgmentScheduler} instead.` where appropriate.

- [ ] **Step 2: Update CLAUDE.md**

Update the "Judgment Foundation Types" section to reflect:
- CallerConfig.Human 12-field description with `CandidateSetSpec` groups/users, `human()` factory
- CallerConfig.Llm with modelId, modelName, systemPrompt
- CallerConfig.A2A with streaming
- CallerIdentity with required callerId/callerType, trustScore
- Evidence with name/type/content/ref (no backward-compat key/value)
- JudgmentTarget.maxEscalationAttempts
- VerificationContext/EscalationContext cleaned records (no raw strings, no Map evidence, no backward-compat constructors)
- HumanTaskTarget/HumanTaskScheduler/HumanTaskScheduleRequest/CloudEventHumanTaskScheduler deprecated

- [ ] **Step 3: Commit**

```
feat(#1012): deprecate HumanTaskTarget/HumanTaskScheduler, update CLAUDE.md

Refs #1012
```

## References

- [specs/issue-1012-enrich-callerconfig/2026-08-30-enrich-judgment-foundation-types-design.md] — design spec
- [api/src/main/java/io/casehub/api/spi/judgment/CallerConfig.java] — current sealed interface
- [api/src/main/java/io/casehub/api/spi/judgment/CallerIdentity.java] — current record
- [api/src/main/java/io/casehub/api/spi/judgment/Evidence.java] — current record
- [api/src/main/java/io/casehub/api/model/JudgmentTarget.java] — current target class
- [api/src/main/java/io/casehub/api/spi/judgment/VerificationContext.java:31-73] — backward-compat to remove
- [api/src/main/java/io/casehub/api/spi/judgment/EscalationContext.java:31-79] — backward-compat to remove
- [runtime/src/main/java/io/casehub/engine/internal/worker/EvidencePresenceVerifier.java] — evidence Map→List
- [runtime/src/main/java/io/casehub/engine/internal/worker/DefaultJudgmentEscalator.java] — CallerConfig.Human constructor
- [runtime/src/main/java/io/casehub/engine/internal/worker/DelegatingJudgmentScheduler.java] — to delete
- [v2 branch commit 6464141f] — CallerConfig/CallerIdentity/Evidence field source
- [GitHub #1012] — issue with field-by-field mapping
