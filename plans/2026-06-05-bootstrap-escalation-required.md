# Bootstrap Escalation Required Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add `bootstrapEscalationRequired` to `TrustRoutingPolicy` and enforce it in both routing strategies so that BOOTSTRAP agents are never assigned to high-stakes capabilities when no QUALIFIED agent exists.

**Architecture:** New `EscalationReason` top-level enum; `TrustRoutingPolicy` gets a `boolean bootstrapEscalationRequired` field; both strategies add a two-part guard (pre-screen that short-circuits before costly work, plus BOOTSTRAP stripping from the scoring pool); `AgentRoutingEscalationEvent` and handler updated for accurate per-reason messages and unconditional metric logging.

**Tech Stack:** Java 21 records, Mutiny `Uni`, Mockito + AssertJ unit tests, Maven multi-module build. No new dependencies.

---

## File Map

| File | Module (folder) | Action |
|------|-----------------|--------|
| `api/src/main/java/io/casehub/api/spi/routing/EscalationReason.java` | `api` | **Create** — new top-level enum |
| `api/src/main/java/io/casehub/api/spi/routing/TrustRoutingPolicy.java` | `api` | Modify — add `boolean bootstrapEscalationRequired`, update `DEFAULT` |
| `api/src/main/java/io/casehub/api/spi/routing/AgentAssignment.java` | `api` | Modify — add `EscalationReason reason` to `EscalateToOversight`, update `escalate()` factory |
| `common/src/main/java/io/casehub/engine/common/internal/event/AgentRoutingEscalationEvent.java` | `common` | Modify — add `EscalationReason reason` field |
| `ledger/src/main/java/io/casehub/ledger/routing/TrustCandidateClassifier.java` | `ledger` | Modify — update `decide()` to use `EscalationReason.BORDERLINE_STALEMATE` |
| `ledger/src/main/java/io/casehub/ledger/routing/TrustWeightedAgentStrategy.java` | `ledger` | Modify — add pre-screen + BOOTSTRAP stripping |
| `engine-ai/src/main/java/io/casehub/engine/ai/routing/SemanticAgentRoutingStrategy.java` | `engine-ai` | Modify — add pre-screen + BOOTSTRAP stripping (before `emitOn`) |
| `runtime/src/main/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandler.java` | `runtime` | Modify — add `escalation.reason()` to `AgentRoutingEscalationEvent` construction |
| `runtime/src/main/java/io/casehub/engine/internal/orchestration/DefaultWorkOrchestrator.java` | `runtime` | Modify — add `e.reason()` to `AgentRoutingEscalationEvent` construction |
| `runtime/src/main/java/io/casehub/engine/internal/engine/handler/AgentRoutingEscalationHandler.java` | `runtime` | Modify — move metric log before channel search, message switch per reason |
| `ledger/src/test/java/io/casehub/ledger/routing/TrustCandidateClassifierTest.java` | `ledger` | Modify — update reason assertions, update constructor calls |
| `ledger/src/test/java/io/casehub/ledger/routing/TrustWeightedAgentStrategyTest.java` | `ledger` | Modify — add bootstrap tests, update existing assertions and constructor calls |
| `engine-ai/src/test/java/io/casehub/engine/ai/routing/SemanticAgentRoutingStrategyTest.java` | `engine-ai` | Modify — add bootstrap tests, update existing assertions and constructor calls |
| `runtime/src/test/java/io/casehub/engine/internal/engine/handler/AgentRoutingEscalationHandlerTest.java` | `runtime` | Modify — update event construction, add `NO_QUALIFIED_AGENT` sub-cases |
| `runtime/src/test/java/io/casehub/engine/internal/orchestration/DefaultWorkOrchestratorTest.java` | `runtime` | Modify — update `escalate()` call to 2-arg |
| `runtime/src/test/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandlerRoutingTest.java` | `runtime` | Modify — update `escalate()` call to 2-arg |

---

## Task 1: Create `EscalationReason` enum

**Files:**
- Create: `api/src/main/java/io/casehub/api/spi/routing/EscalationReason.java`

- [ ] **Step 1.1: Create the file**

```java
/*
 * Copyright 2026-Present The Case Hub Authors
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 * http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
package io.casehub.api.spi.routing;

/** Why agent routing escalated to human oversight. */
public enum EscalationReason {

  /**
   * All candidates scored 0.0 and at least one was BORDERLINE (score within {@code
   * borderlineMargin} of {@code threshold}). The pool has agents but none are clearly qualified.
   */
  BORDERLINE_STALEMATE,

  /**
   * No QUALIFIED agent is available; only BOOTSTRAP-phase agents could be assigned. Pre-screen
   * fires before scoring — no scoring has occurred. Requires human routing.
   */
  NO_QUALIFIED_AGENT
}
```

- [ ] **Step 1.2: Verify it compiles**

```bash
cd /Users/mdproctor/claude/casehub/engine
mvn install -DskipTests -q -pl api
```

Expected: BUILD SUCCESS

- [ ] **Step 1.3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add api/src/main/java/io/casehub/api/spi/routing/EscalationReason.java
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(engine#415): add EscalationReason enum — BORDERLINE_STALEMATE, NO_QUALIFIED_AGENT

Refs #415"
```

---

## Task 2: Update `TrustRoutingPolicy` — add `bootstrapEscalationRequired`

**Files:**
- Modify: `api/src/main/java/io/casehub/api/spi/routing/TrustRoutingPolicy.java`

- [ ] **Step 2.1: Add field to record and update DEFAULT**

Replace the entire file content:

```java
/*
 * Copyright 2026-Present The Case Hub Authors
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 * http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
package io.casehub.api.spi.routing;

import java.util.Map;

/**
 * Per-capability trust routing policy parameters.
 *
 * @param threshold minimum CAPABILITY trust score for selection (Phase 2 entry)
 * @param minimumObservations decision count below which routing falls to Phase 0/1 (availability)
 * @param borderlineMargin candidates whose score is within this margin of the threshold are
 *     excluded (score 0.0); tracked for escalation in engine#377
 * @param blendFactor weight of trust score vs workload efficiency (0.0 = pure workload, 1.0 = pure
 *     trust)
 * @param qualityFloors Phase 3: dimension name → minimum acceptable quality score; candidates
 *     failing any floor are excluded; no penalty if dimension data is absent
 * @param bootstrapEscalationRequired when true, BOOTSTRAP candidates are stripped from the scoring
 *     pool; if no QUALIFIED agent exists, escalates to {@link
 *     EscalationReason#NO_QUALIFIED_AGENT} instead of assigning an unproven agent. Set to true for
 *     high-stakes, irreversible capabilities.
 */
public record TrustRoutingPolicy(
    double threshold,
    int minimumObservations,
    double borderlineMargin,
    double blendFactor,
    Map<String, Double> qualityFloors,
    boolean bootstrapEscalationRequired) {

  /** Conservative defaults: 0.7 threshold, 10 observations, 0.1 margin, 60% trust blend, no bootstrap guard. */
  public static final TrustRoutingPolicy DEFAULT =
      new TrustRoutingPolicy(0.7, 10, 0.1, 0.6, Map.of(), false);

  /** True when an agent lacks sufficient decision history for trust-based routing (Phase 0/1). */
  public boolean isBootstrap(final int decisionCount) {
    return decisionCount < minimumObservations;
  }

  /**
   * True when the trust score is within {@code borderlineMargin} of {@code threshold}.
   *
   * <p>A borderline candidate is NOT qualified for assignment. Borderline is a distinct Phase 2a
   * state that triggers human oversight when all candidates are in this state.
   */
  public boolean isBorderline(final double score) {
    return Math.abs(score - threshold) <= borderlineMargin;
  }

  /**
   * True when the score exceeds the threshold and is not borderline.
   *
   * <p>This is a Phase 2 first-pass check only — Phase 3 quality floors may still exclude a
   * candidate that passes this check. Do not interpret as "ready to assign".
   */
  public boolean passesThresholdCheck(final double score) {
    return score >= threshold && !isBorderline(score);
  }
}
```

- [ ] **Step 2.2: Verify it compiles**

```bash
cd /Users/mdproctor/claude/casehub/engine
mvn install -DskipTests -q -pl api
```

Expected: BUILD SUCCESS

- [ ] **Step 2.3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add api/src/main/java/io/casehub/api/spi/routing/TrustRoutingPolicy.java
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(engine#415): add bootstrapEscalationRequired to TrustRoutingPolicy

Refs #415"
```

---

## Task 3: Update `AgentAssignment` — add `EscalationReason` to `EscalateToOversight`

**Files:**
- Modify: `api/src/main/java/io/casehub/api/spi/routing/AgentAssignment.java`

- [ ] **Step 3.1: Update `EscalateToOversight` and `escalate()` factory**

Replace the entire file:

```java
/*
 * Copyright 2026-Present The Case Hub Authors
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 * http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
package io.casehub.api.spi.routing;

/**
 * Result of {@link AgentRoutingStrategy#select}. A sealed type with three distinct outcomes:
 *
 * <ul>
 *   <li>{@link Assigned} — a specific worker was selected
 *   <li>{@link Unresolvable} — no candidate passed trust filters (none were borderline)
 *   <li>{@link EscalateToOversight} — routing cannot proceed automatically; human oversight
 *       required. The {@link EscalationReason} indicates why.
 * </ul>
 *
 * <p>Callers must switch exhaustively on the sealed type. The three outcomes are semantically
 * distinct and the engine's response to each differs.
 */
public sealed interface AgentAssignment
    permits AgentAssignment.Assigned,
        AgentAssignment.Unresolvable,
        AgentAssignment.EscalateToOversight {

  /** A specific worker was selected for the capability. */
  record Assigned(String workerId) implements AgentAssignment {}

  /**
   * No candidate passed trust filters. None were borderline — the pool simply lacks qualified
   * agents. Engine falls to {@code tryProvision()}.
   */
  record Unresolvable() implements AgentAssignment {}

  /**
   * Routing cannot proceed automatically. See {@link EscalationReason} for why. Engine must route
   * to human oversight via the oversight channel.
   */
  record EscalateToOversight(String capabilityName, EscalationReason reason)
      implements AgentAssignment {}

  static AgentAssignment assign(final String workerId) {
    return new Assigned(workerId);
  }

  static AgentAssignment unresolvable() {
    return new Unresolvable();
  }

  static AgentAssignment escalate(final String capabilityName, final EscalationReason reason) {
    return new EscalateToOversight(capabilityName, reason);
  }
}
```

- [ ] **Step 3.2: Fix compile errors in all callers — DO NOT build yet, first fix all files**

The `escalate(String)` factory is removed. Find all call sites:

```bash
grep -rn "AgentAssignment.escalate\|\.escalate(" /Users/mdproctor/claude/casehub/engine --include="*.java" | grep -v "Binary"
```

Production call sites to fix (change to 2-arg form):

**`ledger/src/main/java/io/casehub/ledger/routing/TrustCandidateClassifier.java`** — line with `AgentAssignment.escalate(capabilityName)`:
```java
// Old:
return anyBorderline
    ? AgentAssignment.escalate(capabilityName)
    : AgentAssignment.unresolvable();
// New (add import: import io.casehub.api.spi.routing.EscalationReason;):
return anyBorderline
    ? AgentAssignment.escalate(capabilityName, EscalationReason.BORDERLINE_STALEMATE)
    : AgentAssignment.unresolvable();
```

Add import to `TrustCandidateClassifier.java`:
```java
import io.casehub.api.spi.routing.EscalationReason;
```

Test call sites to fix:

**`runtime/src/test/java/io/casehub/engine/internal/orchestration/DefaultWorkOrchestratorTest.java`** — find line with `AgentAssignment.escalate("analyse")`:
```java
// Old:
.thenReturn(Uni.createFrom().item(AgentAssignment.escalate("analyse")));
// New:
.thenReturn(Uni.createFrom().item(AgentAssignment.escalate("analyse", EscalationReason.BORDERLINE_STALEMATE)));
```
Add import: `import io.casehub.api.spi.routing.EscalationReason;`

**`runtime/src/test/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandlerRoutingTest.java`** — find line with `AgentAssignment.escalate("research")`:
```java
// Old:
.thenReturn(Uni.createFrom().item(AgentAssignment.escalate("research")));
// New:
.thenReturn(Uni.createFrom().item(AgentAssignment.escalate("research", EscalationReason.BORDERLINE_STALEMATE)));
```
Add import: `import io.casehub.api.spi.routing.EscalationReason;`

- [ ] **Step 3.3: Build `api` + verify downstream compiles**

```bash
cd /Users/mdproctor/claude/casehub/engine
mvn install -DskipTests -q
```

Expected: BUILD SUCCESS across all modules (all compile-time call sites fixed)

- [ ] **Step 3.4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add \
  api/src/main/java/io/casehub/api/spi/routing/AgentAssignment.java \
  ledger/src/main/java/io/casehub/ledger/routing/TrustCandidateClassifier.java \
  runtime/src/test/java/io/casehub/engine/internal/orchestration/DefaultWorkOrchestratorTest.java \
  runtime/src/test/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandlerRoutingTest.java
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(engine#415): add EscalationReason to EscalateToOversight; update escalate() factory

Refs #415"
```

---

## Task 4: Update `AgentRoutingEscalationEvent` and its construction sites

**Files:**
- Modify: `common/src/main/java/io/casehub/engine/common/internal/event/AgentRoutingEscalationEvent.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandler.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/orchestration/DefaultWorkOrchestrator.java`

- [ ] **Step 4.1: Update `AgentRoutingEscalationEvent`**

Read the current file first. Replace it with:

```java
/*
 * Copyright 2026-Present The Case Hub Authors
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 * http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
package io.casehub.engine.common.internal.event;

import io.casehub.api.spi.routing.EscalationReason;
import java.util.UUID;

/**
 * Published when agent routing cannot proceed automatically and human oversight is required.
 * The {@link EscalationReason} indicates whether the trigger was a borderline stalemate or a
 * pool with no trust-qualified agents.
 */
public record AgentRoutingEscalationEvent(
    UUID caseId, String capabilityName, String bindingName, EscalationReason reason) {}
```

- [ ] **Step 4.2: Update `CaseContextChangedEventHandler` — `handleEscalation()`**

Find the `handleEscalation` method. The `AgentRoutingEscalationEvent` construction needs the reason. Change:

```java
// Old:
new AgentRoutingEscalationEvent(
    caseInstance.getUuid(), escalation.capabilityName(), binding.getName())
// New:
new AgentRoutingEscalationEvent(
    caseInstance.getUuid(), escalation.capabilityName(), binding.getName(), escalation.reason())
```

- [ ] **Step 4.3: Update `DefaultWorkOrchestrator` — `EscalateToOversight` handling**

Find the `case AgentAssignment.EscalateToOversight e ->` block. Change:

```java
// Old:
new AgentRoutingEscalationEvent(
    instance.getUuid(), e.capabilityName(), "(direct-orchestration)")
// New:
new AgentRoutingEscalationEvent(
    instance.getUuid(), e.capabilityName(), "(direct-orchestration)", e.reason())
```

- [ ] **Step 4.4: Update handler test to add reason to existing event constructions**

In `AgentRoutingEscalationHandlerTest`, all three `new AgentRoutingEscalationEvent(...)` calls need the reason arg:

```java
// Old:
new AgentRoutingEscalationEvent(caseId, "research", "research-binding")
// New:
new AgentRoutingEscalationEvent(caseId, "research", "research-binding", EscalationReason.BORDERLINE_STALEMATE)
```

Add import to the test: `import io.casehub.api.spi.routing.EscalationReason;`

- [ ] **Step 4.5: Build and run runtime tests**

```bash
cd /Users/mdproctor/claude/casehub/engine
mvn install -DskipTests -q
TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl runtime
```

Expected: All existing tests pass.

- [ ] **Step 4.6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add \
  common/src/main/java/io/casehub/engine/common/internal/event/AgentRoutingEscalationEvent.java \
  runtime/src/main/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandler.java \
  runtime/src/main/java/io/casehub/engine/internal/orchestration/DefaultWorkOrchestrator.java \
  runtime/src/test/java/io/casehub/engine/internal/engine/handler/AgentRoutingEscalationHandlerTest.java
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(engine#415): add EscalationReason to AgentRoutingEscalationEvent; update construction sites

Refs #415"
```

---

## Task 5: Update `TrustCandidateClassifier` tests and fix constructor calls

**Files:**
- Modify: `ledger/src/test/java/io/casehub/ledger/routing/TrustCandidateClassifierTest.java`

The `decide()` method already received the `BORDERLINE_STALEMATE` change in Task 3. Now update the test to assert the reason.

- [ ] **Step 5.1: Write failing tests — update `decide_allZeroScores_someBorderline_returnsEscalate`**

Add `reason` assertion to the existing test and update `TrustRoutingPolicy` constructor calls.

In `TrustCandidateClassifierTest.java`:

1. Add import: `import io.casehub.api.spi.routing.EscalationReason;`

2. Update `POLICY` constant:
```java
// Old:
private static final io.casehub.api.spi.routing.TrustRoutingPolicy POLICY =
    new io.casehub.api.spi.routing.TrustRoutingPolicy(0.7, 5, 0.1, 0.6, Map.of());
// New:
private static final io.casehub.api.spi.routing.TrustRoutingPolicy POLICY =
    new io.casehub.api.spi.routing.TrustRoutingPolicy(0.7, 5, 0.1, 0.6, Map.of(), false);
```

3. Update `policyWithFloor` in `classify_qualityFloorFailed_isExcludedPhase3`:
```java
// Old:
new io.casehub.api.spi.routing.TrustRoutingPolicy(
    0.7, 5, 0.1, 0.6, Map.of("thoroughness", 0.75));
// New:
new io.casehub.api.spi.routing.TrustRoutingPolicy(
    0.7, 5, 0.1, 0.6, Map.of("thoroughness", 0.75), false);
```

4. Update `policyWithFloor` in `classify_qualityFloorMissing_isQualified`:
```java
// Old:
new io.casehub.api.spi.routing.TrustRoutingPolicy(
    0.7, 5, 0.1, 0.6, Map.of("thoroughness", 0.75));
// New:
new io.casehub.api.spi.routing.TrustRoutingPolicy(
    0.7, 5, 0.1, 0.6, Map.of("thoroughness", 0.75), false);
```

5. Update `decide_allZeroScores_someBorderline_returnsEscalate` — add reason assertion:
```java
@Test
void decide_allZeroScores_someBorderline_returnsEscalate() {
  final ClassifiedCandidate cand = classified("worker-1", Phase.BORDERLINE, 0.65, 1.0);
  final ScoredCandidate scored = new ScoredCandidate(cand, 0.0);

  final AgentAssignment result = classifier.decide(List.of(cand), List.of(scored), CAP);

  assertThat(result).isInstanceOf(AgentAssignment.EscalateToOversight.class);
  assertThat(((AgentAssignment.EscalateToOversight) result).capabilityName()).isEqualTo(CAP);
  assertThat(((AgentAssignment.EscalateToOversight) result).reason())
      .isEqualTo(EscalationReason.BORDERLINE_STALEMATE);
}
```

6. Update `decide_mixedBorderlineAndExcluded_allZero_returnsEscalate` — add reason assertion:
```java
@Test
void decide_mixedBorderlineAndExcluded_allZero_returnsEscalate() {
  final ClassifiedCandidate border = classified("w1", Phase.BORDERLINE, 0.65, 1.0);
  final ClassifiedCandidate excluded = classified("w2", Phase.EXCLUDED_PHASE2B, 0.5, 1.0);
  final List<ScoredCandidate> scored =
      List.of(new ScoredCandidate(border, 0.0), new ScoredCandidate(excluded, 0.0));

  final AgentAssignment result = classifier.decide(List.of(border, excluded), scored, CAP);

  assertThat(result).isInstanceOf(AgentAssignment.EscalateToOversight.class);
  assertThat(((AgentAssignment.EscalateToOversight) result).reason())
      .isEqualTo(EscalationReason.BORDERLINE_STALEMATE);
}
```

- [ ] **Step 5.2: Run failing tests**

```bash
cd /Users/mdproctor/claude/casehub/engine
mvn install -DskipTests -q
TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl ledger -Dtest=TrustCandidateClassifierTest
```

Expected: Tests fail with "expected BORDERLINE_STALEMATE but was..." or compilation error if `reason()` is unknown. If they PASS (because Task 3 already fixed `decide()`), proceed — the assertions are confirming the existing correct behavior.

- [ ] **Step 5.3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add \
  ledger/src/test/java/io/casehub/ledger/routing/TrustCandidateClassifierTest.java
git -C /Users/mdproctor/claude/casehub/engine commit -m "test(engine#415): assert BORDERLINE_STALEMATE reason in TrustCandidateClassifierTest

Refs #415"
```

---

## Task 6: Bootstrap guard in `TrustWeightedAgentStrategy` — TDD

**Files:**
- Modify: `ledger/src/test/java/io/casehub/ledger/routing/TrustWeightedAgentStrategyTest.java`
- Modify: `ledger/src/main/java/io/casehub/ledger/routing/TrustWeightedAgentStrategy.java`

- [ ] **Step 6.1: Write failing bootstrap tests**

In `TrustWeightedAgentStrategyTest.java`, add the following changes:

1. Add imports:
```java
import io.casehub.api.spi.routing.EscalationReason;
```

2. Update `DEFAULT_POLICY` to 6-arg:
```java
private static final TrustRoutingPolicy DEFAULT_POLICY =
    new TrustRoutingPolicy(0.7, 5, 0.1, 0.6, Map.of(), false);
```

3. Add `BOOTSTRAP_GUARD_POLICY` constant:
```java
private static final TrustRoutingPolicy BOOTSTRAP_GUARD_POLICY =
    new TrustRoutingPolicy(0.7, 5, 0.1, 0.6, Map.of(), true);
```

4. Update all inline `new TrustRoutingPolicy(...)` constructions in existing tests:
```java
// In phase3 tests — add false as 6th arg:
new TrustRoutingPolicy(0.7, 5, 0.1, 0.6, Map.of("thoroughness", 0.75), false)
// In blendFactor tests:
new TrustRoutingPolicy(0.7, 5, 0.1, 1.0, Map.of(), false)  // pureTrust
new TrustRoutingPolicy(0.7, 5, 0.1, 0.0, Map.of(), false)  // pureWorkload
```

5. Update existing escalation assertions to add reason check:
```java
// In phase2a_singleBorderlineCandidate_escalates and similar:
assertThat(result).isInstanceOf(AgentAssignment.EscalateToOversight.class);
assertThat(((AgentAssignment.EscalateToOversight) result).capabilityName())
    .isEqualTo("research");
assertThat(((AgentAssignment.EscalateToOversight) result).reason())
    .isEqualTo(EscalationReason.BORDERLINE_STALEMATE);
```

(Apply the reason assertion to all 4 existing `EscalateToOversight` assertions in the file.)

6. Add new bootstrap test section after `// ---- All-excluded edge case ---`:

```java
// ---- Bootstrap guard (bootstrapEscalationRequired = true) -----------------------

@Test
void bootstrap_noQualified_allBootstrap_escalatesNoQualifiedAgent() {
  when(policyProvider.forCapability("research")).thenReturn(BOOTSTRAP_GUARD_POLICY);
  // All candidates: no trust score → BOOTSTRAP

  final AgentAssignment result = select(List.of(candidate("agent-1", 0), candidate("agent-2", 1)));

  assertThat(result).isInstanceOf(AgentAssignment.EscalateToOversight.class);
  assertThat(((AgentAssignment.EscalateToOversight) result).reason())
      .isEqualTo(EscalationReason.NO_QUALIFIED_AGENT);
  assertThat(((AgentAssignment.EscalateToOversight) result).capabilityName())
      .isEqualTo("research");
}

@Test
void bootstrap_noQualified_bootstrapPlusBorderline_escalatesNoQualifiedAgent() {
  // Closes the mixed-pool gap: BOOTSTRAP (workload>0) + BORDERLINE (score=0) → BOOTSTRAP would
  // win without the guard. Pre-screen: no QUALIFIED → escalate.
  when(policyProvider.forCapability("research")).thenReturn(BOOTSTRAP_GUARD_POLICY);
  when(cache.getCapabilityScore("agent-border", "research")).thenReturn(OptionalDouble.of(0.65));
  when(cache.getDecisionCount("agent-border", "research")).thenReturn(10);
  // agent-new: BOOTSTRAP (no score)

  final AgentAssignment result =
      select(List.of(candidate("agent-border", 0), candidate("agent-new", 0)));

  assertThat(result).isInstanceOf(AgentAssignment.EscalateToOversight.class);
  assertThat(((AgentAssignment.EscalateToOversight) result).reason())
      .isEqualTo(EscalationReason.NO_QUALIFIED_AGENT);
}

@Test
void bootstrap_noQualified_bootstrapPlusExcluded_escalatesNoQualifiedAgent() {
  // EXCLUDED_PHASE2B (score<threshold, score=0) + BOOTSTRAP → BOOTSTRAP would win by workload.
  when(policyProvider.forCapability("research")).thenReturn(BOOTSTRAP_GUARD_POLICY);
  when(cache.getCapabilityScore("agent-low", "research")).thenReturn(OptionalDouble.of(0.5));
  when(cache.getDecisionCount("agent-low", "research")).thenReturn(10);
  // agent-new: BOOTSTRAP

  final AgentAssignment result =
      select(List.of(candidate("agent-low", 0), candidate("agent-new", 0)));

  assertThat(result).isInstanceOf(AgentAssignment.EscalateToOversight.class);
  assertThat(((AgentAssignment.EscalateToOversight) result).reason())
      .isEqualTo(EscalationReason.NO_QUALIFIED_AGENT);
}

@Test
void bootstrap_qualifiedExists_bootstrapStripped_qualifiedAssigned() {
  // QUALIFIED agent exists → pre-screen skips, BOOTSTRAP stripped from scoring pool.
  when(policyProvider.forCapability("research")).thenReturn(BOOTSTRAP_GUARD_POLICY);
  when(cache.getCapabilityScore("agent-qualified", "research")).thenReturn(OptionalDouble.of(0.85));
  when(cache.getDecisionCount("agent-qualified", "research")).thenReturn(10);
  // agent-new: BOOTSTRAP, 0 jobs (would outscore QUALIFIED by workload without flag)

  final AgentAssignment result =
      select(List.of(candidate("agent-qualified", 2), candidate("agent-new", 0)));

  assertThat(result).isInstanceOf(AgentAssignment.Assigned.class);
  assertThat(((AgentAssignment.Assigned) result).workerId()).isEqualTo("agent-qualified");
}

@Test
void bootstrap_qualifiedExists_bootstrapStripped_busyQualifiedWinsOverIdleBootstrap() {
  // Explicit: flag overrides workload comparison. Busy QUALIFIED beats idle BOOTSTRAP.
  // Without flag: BOOTSTRAP workload=1.0 > QUALIFIED blended≈0.55 (5 jobs) → BOOTSTRAP would win.
  when(policyProvider.forCapability("research")).thenReturn(BOOTSTRAP_GUARD_POLICY);
  when(cache.getCapabilityScore("agent-qualified", "research")).thenReturn(OptionalDouble.of(0.85));
  when(cache.getDecisionCount("agent-qualified", "research")).thenReturn(10);

  final AgentAssignment result =
      select(List.of(candidate("agent-qualified", 5), candidate("agent-new", 0)));

  assertThat(result).isInstanceOf(AgentAssignment.Assigned.class);
  assertThat(((AgentAssignment.Assigned) result).workerId()).isEqualTo("agent-qualified");
}

@Test
void bootstrap_qualifiedExists_bootstrapPlusBorderline_qualifiedWins_noBorderlineStalemate() {
  // [BOOTSTRAP, QUALIFIED, BORDERLINE] with flag → BOOTSTRAP stripped, eligible=[QUALIFIED,BORDERLINE].
  // QUALIFIED scores positively; BORDERLINE_STALEMATE must NOT fire.
  when(policyProvider.forCapability("research")).thenReturn(BOOTSTRAP_GUARD_POLICY);
  when(cache.getCapabilityScore("agent-qualified", "research")).thenReturn(OptionalDouble.of(0.85));
  when(cache.getDecisionCount("agent-qualified", "research")).thenReturn(10);
  when(cache.getCapabilityScore("agent-border", "research")).thenReturn(OptionalDouble.of(0.65));
  when(cache.getDecisionCount("agent-border", "research")).thenReturn(10);
  // agent-new: BOOTSTRAP

  final List<AgentCandidate> candidates = List.of(
      candidate("agent-qualified", 0),
      candidate("agent-border", 0),
      candidate("agent-new", 0));

  final AgentAssignment result = select(candidates);

  assertThat(result).isInstanceOf(AgentAssignment.Assigned.class);
  assertThat(((AgentAssignment.Assigned) result).workerId()).isEqualTo("agent-qualified");
}

@Test
void bootstrap_flagFalse_allBootstrap_assignsByWorkload() {
  // DEFAULT_POLICY has bootstrapEscalationRequired = false; existing behaviour preserved.
  when(policyProvider.forCapability("research")).thenReturn(DEFAULT_POLICY);

  final AgentAssignment result = select(List.of(candidate("agent-busy", 5), candidate("agent-idle", 0)));

  assertThat(result).isInstanceOf(AgentAssignment.Assigned.class);
  assertThat(((AgentAssignment.Assigned) result).workerId()).isEqualTo("agent-idle");
}
```

- [ ] **Step 6.2: Run failing tests — verify they fail**

```bash
cd /Users/mdproctor/claude/casehub/engine
mvn install -DskipTests -q
TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl ledger -Dtest=TrustWeightedAgentStrategyTest
```

Expected: New bootstrap tests fail ("expected NO_QUALIFIED_AGENT but was Assigned" or similar). Existing borderline tests may fail if reason assertion is new. The constructor compile error (5 args) would surface first if you missed any constructor call.

- [ ] **Step 6.3: Implement pre-screen + stripping in `TrustWeightedAgentStrategy`**

Full updated `select()` method:

```java
@Override
public Uni<AgentAssignment> select(
    final AgentRoutingContext context, final List<AgentCandidate> candidates) {
  if (candidates.isEmpty()) {
    return Uni.createFrom().item(AgentAssignment.unresolvable());
  }

  final TrustRoutingPolicy policy = policyProvider.forCapability(context.capabilityName());
  final List<ClassifiedCandidate> classified =
      classifier.classify(candidates, context.capabilityName(), policy, cache);

  // Bootstrap guard: pre-screen before scoring
  if (policy.bootstrapEscalationRequired()) {
    final boolean hasQualified =
        classified.stream().anyMatch(c -> c.phase() == Phase.QUALIFIED);
    final boolean hasBootstrap =
        classified.stream().anyMatch(c -> c.phase() == Phase.BOOTSTRAP);
    if (!hasQualified && hasBootstrap) {
      return Uni.createFrom()
          .item(
              AgentAssignment.escalate(
                  context.capabilityName(), EscalationReason.NO_QUALIFIED_AGENT));
    }
  }

  // Strip BOOTSTRAP from scoring when guard is active (reached only if QUALIFIED exists)
  final List<ClassifiedCandidate> eligible =
      policy.bootstrapEscalationRequired()
          ? classified.stream().filter(c -> c.phase() != Phase.BOOTSTRAP).toList()
          : classified;

  final List<ScoredCandidate> scored = new ArrayList<>(eligible.size());
  for (final ClassifiedCandidate cc : eligible) {
    scored.add(new ScoredCandidate(cc, score(cc, policy)));
  }

  return Uni.createFrom().item(classifier.decide(eligible, scored, context.capabilityName()));
}
```

Add import to `TrustWeightedAgentStrategy.java`:
```java
import io.casehub.api.spi.routing.EscalationReason;
```

- [ ] **Step 6.4: Run all ledger tests — verify pass**

```bash
cd /Users/mdproctor/claude/casehub/engine
TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl ledger
```

Expected: ALL tests pass including all new bootstrap tests and updated borderline reason assertions.

- [ ] **Step 6.5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add \
  ledger/src/main/java/io/casehub/ledger/routing/TrustWeightedAgentStrategy.java \
  ledger/src/test/java/io/casehub/ledger/routing/TrustWeightedAgentStrategyTest.java
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(engine#415): add bootstrap pre-screen + stripping to TrustWeightedAgentStrategy

When bootstrapEscalationRequired=true and no QUALIFIED agent exists, escalates with
NO_QUALIFIED_AGENT instead of assigning BOOTSTRAP agent. Strips BOOTSTRAP from
scoring pool when QUALIFIED exists, so busy QUALIFIED beats idle BOOTSTRAP.

Refs #415"
```

---

## Task 7: Bootstrap guard in `SemanticAgentRoutingStrategy` — TDD

**Files:**
- Modify: `engine-ai/src/test/java/io/casehub/engine/ai/routing/SemanticAgentRoutingStrategyTest.java`
- Modify: `engine-ai/src/main/java/io/casehub/engine/ai/routing/SemanticAgentRoutingStrategy.java`

- [ ] **Step 7.1: Write failing bootstrap tests**

In `SemanticAgentRoutingStrategyTest.java`:

1. Add import: `import io.casehub.api.spi.routing.EscalationReason;`

2. Update `POLICY` constant to 6-arg:
```java
private static final TrustRoutingPolicy POLICY =
    new TrustRoutingPolicy(0.7, 5, 0.1, 0.6, Map.of(), false);
```

3. Add `BOOTSTRAP_GUARD_POLICY` constant:
```java
private static final TrustRoutingPolicy BOOTSTRAP_GUARD_POLICY =
    new TrustRoutingPolicy(0.7, 5, 0.1, 0.6, Map.of(), true);
```

4. Update `allBorderlineCandidates_escalates` — add reason assertion:
```java
assertThat(result).isInstanceOf(AgentAssignment.EscalateToOversight.class);
assertThat(((AgentAssignment.EscalateToOversight) result).reason())
    .isEqualTo(EscalationReason.BORDERLINE_STALEMATE);
```

5. Add new bootstrap test section. Note: tests where the pre-screen fires (no QUALIFIED) do NOT require embedding mocks — the guard returns before entering `emitOn(workerPool)`.

```java
// ---- Bootstrap guard (bootstrapEscalationRequired = true) -----------------------

@Test
void bootstrap_noQualified_allBootstrap_escalatesNoQualifiedAgent() {
  // Pre-screen fires BEFORE emitOn(workerPool) — no embedding cost.
  when(policyProvider.forCapability("research")).thenReturn(BOOTSTRAP_GUARD_POLICY);
  // All candidates: no trust score → BOOTSTRAP

  final List<AgentCandidate> candidates = List.of(
      new AgentCandidate("agent-1", Set.of("research"), 0, AgentHealth.READY, null),
      new AgentCandidate("agent-2", Set.of("research"), 1, AgentHealth.READY, null));

  final AgentAssignment result = strategy.select(ctx(), candidates).await().indefinitely();

  assertThat(result).isInstanceOf(AgentAssignment.EscalateToOversight.class);
  assertThat(((AgentAssignment.EscalateToOversight) result).reason())
      .isEqualTo(EscalationReason.NO_QUALIFIED_AGENT);
  assertThat(((AgentAssignment.EscalateToOversight) result).capabilityName())
      .isEqualTo("research");
}

@Test
void bootstrap_noQualified_bootstrapPlusBorderline_escalatesNoQualifiedAgent() {
  // Mixed-pool gap: BOOTSTRAP + BORDERLINE → pre-screen fires, no embedding cost.
  when(policyProvider.forCapability("research")).thenReturn(BOOTSTRAP_GUARD_POLICY);
  when(cache.getCapabilityScore("agent-border", "research")).thenReturn(OptionalDouble.of(0.65));
  when(cache.getDecisionCount("agent-border", "research")).thenReturn(10);

  final List<AgentCandidate> candidates = List.of(
      candidateWithDescriptor("agent-border", 0, "agent-b"),
      new AgentCandidate("agent-new", Set.of("research"), 0, AgentHealth.READY, null));

  final AgentAssignment result = strategy.select(ctx(), candidates).await().indefinitely();

  assertThat(result).isInstanceOf(AgentAssignment.EscalateToOversight.class);
  assertThat(((AgentAssignment.EscalateToOversight) result).reason())
      .isEqualTo(EscalationReason.NO_QUALIFIED_AGENT);
}

@Test
void bootstrap_noQualified_bootstrapPlusExcluded_escalatesNoQualifiedAgent() {
  when(policyProvider.forCapability("research")).thenReturn(BOOTSTRAP_GUARD_POLICY);
  when(cache.getCapabilityScore("agent-low", "research")).thenReturn(OptionalDouble.of(0.5));
  when(cache.getDecisionCount("agent-low", "research")).thenReturn(10);

  final List<AgentCandidate> candidates = List.of(
      candidateWithDescriptor("agent-low", 0, "agent-l"),
      new AgentCandidate("agent-new", Set.of("research"), 0, AgentHealth.READY, null));

  final AgentAssignment result = strategy.select(ctx(), candidates).await().indefinitely();

  assertThat(result).isInstanceOf(AgentAssignment.EscalateToOversight.class);
  assertThat(((AgentAssignment.EscalateToOversight) result).reason())
      .isEqualTo(EscalationReason.NO_QUALIFIED_AGENT);
}

@Test
void bootstrap_qualifiedExists_bootstrapStripped_qualifiedAssigned() {
  // QUALIFIED exists → pre-screen skips, enters worker pool, BOOTSTRAP stripped from eligible.
  // Embedding mocks needed because QUALIFIED candidate goes through semantic scoring.
  when(policyProvider.forCapability("research")).thenReturn(BOOTSTRAP_GUARD_POLICY);
  when(cache.getCapabilityScore("agent-qualified", "research")).thenReturn(OptionalDouble.of(0.85));
  when(cache.getDecisionCount("agent-qualified", "research")).thenReturn(10);
  when(jqEvaluator.eval(anyString(), any()))
      .thenReturn(ValidationResult.ok(List.of(MAPPER.createObjectNode().textNode("research"))));
  when(embeddingProvider.embed(any()))
      .thenReturn(new float[]{1.0f, 0.0f})    // query vector
      .thenReturn(new float[]{0.9f, 0.1f});   // agent-qualified descriptor

  final List<AgentCandidate> candidates = List.of(
      candidateWithDescriptor("agent-qualified", 2, "agent-q"),
      new AgentCandidate("agent-new", Set.of("research"), 0, AgentHealth.READY, null));

  final AgentAssignment result = strategy.select(ctx(), candidates).await().indefinitely();

  assertThat(result).isInstanceOf(AgentAssignment.Assigned.class);
  assertThat(((AgentAssignment.Assigned) result).workerId()).isEqualTo("agent-qualified");
}

@Test
void bootstrap_qualifiedExists_bootstrapStripped_busyQualifiedWinsOverIdleBootstrap() {
  // Explicit: flag overrides workload comparison.
  when(policyProvider.forCapability("research")).thenReturn(BOOTSTRAP_GUARD_POLICY);
  when(cache.getCapabilityScore("agent-qualified", "research")).thenReturn(OptionalDouble.of(0.85));
  when(cache.getDecisionCount("agent-qualified", "research")).thenReturn(10);
  when(jqEvaluator.eval(anyString(), any()))
      .thenReturn(ValidationResult.ok(List.of(MAPPER.createObjectNode().textNode("research"))));
  when(embeddingProvider.embed(any()))
      .thenReturn(new float[]{1.0f, 0.0f})
      .thenReturn(new float[]{0.9f, 0.1f});

  final List<AgentCandidate> candidates = List.of(
      candidateWithDescriptor("agent-qualified", 5, "agent-q"),  // 5 running jobs
      new AgentCandidate("agent-new", Set.of("research"), 0, AgentHealth.READY, null));

  final AgentAssignment result = strategy.select(ctx(), candidates).await().indefinitely();

  assertThat(result).isInstanceOf(AgentAssignment.Assigned.class);
  assertThat(((AgentAssignment.Assigned) result).workerId()).isEqualTo("agent-qualified");
}

@Test
void bootstrap_qualifiedExists_bootstrapPlusBorderline_qualifiedWins_noBorderlineStalemate() {
  // [BOOTSTRAP, QUALIFIED, BORDERLINE]: BOOTSTRAP stripped; eligible=[QUALIFIED, BORDERLINE].
  // QUALIFIED wins; BORDERLINE_STALEMATE must NOT fire.
  when(policyProvider.forCapability("research")).thenReturn(BOOTSTRAP_GUARD_POLICY);
  when(cache.getCapabilityScore("agent-qualified", "research")).thenReturn(OptionalDouble.of(0.85));
  when(cache.getDecisionCount("agent-qualified", "research")).thenReturn(10);
  when(cache.getCapabilityScore("agent-border", "research")).thenReturn(OptionalDouble.of(0.65));
  when(cache.getDecisionCount("agent-border", "research")).thenReturn(10);
  when(jqEvaluator.eval(anyString(), any()))
      .thenReturn(ValidationResult.ok(List.of(MAPPER.createObjectNode().textNode("research"))));
  when(embeddingProvider.embed(any()))
      .thenReturn(new float[]{1.0f, 0.0f})
      .thenReturn(new float[]{0.9f, 0.1f});  // agent-qualified descriptor only (BORDERLINE scores 0.0)

  final List<AgentCandidate> candidates = List.of(
      candidateWithDescriptor("agent-qualified", 0, "agent-q"),
      candidateWithDescriptor("agent-border", 0, "agent-b"),
      new AgentCandidate("agent-new", Set.of("research"), 0, AgentHealth.READY, null));

  final AgentAssignment result = strategy.select(ctx(), candidates).await().indefinitely();

  assertThat(result).isInstanceOf(AgentAssignment.Assigned.class);
  assertThat(((AgentAssignment.Assigned) result).workerId()).isEqualTo("agent-qualified");
}

@Test
void bootstrap_flagFalse_allBootstrap_assignsByWorkload() {
  // POLICY has bootstrapEscalationRequired = false; pre-screen skipped.
  when(jqEvaluator.eval(anyString(), any()))
      .thenReturn(ValidationResult.ok(List.of(MAPPER.createObjectNode().textNode("research"))));
  when(embeddingProvider.embed(any())).thenReturn(new float[]{1.0f, 0.0f});

  final List<AgentCandidate> candidates = List.of(
      new AgentCandidate("agent-busy", Set.of("research"), 5, AgentHealth.READY, null),
      new AgentCandidate("agent-idle", Set.of("research"), 0, AgentHealth.READY, null));

  final AgentAssignment result = strategy.select(ctx(), candidates).await().indefinitely();

  assertThat(result).isInstanceOf(AgentAssignment.Assigned.class);
  assertThat(((AgentAssignment.Assigned) result).workerId()).isEqualTo("agent-idle");
}
```

- [ ] **Step 7.2: Run failing tests**

```bash
cd /Users/mdproctor/claude/casehub/engine
mvn install -DskipTests -q
TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl engine-ai -Dtest=SemanticAgentRoutingStrategyTest
```

Expected: New bootstrap tests fail. Existing `allBorderlineCandidates_escalates` fails on reason assertion.

- [ ] **Step 7.3: Implement pre-screen + stripping in `SemanticAgentRoutingStrategy`**

The critical constraint: the pre-screen MUST fire BEFORE `emitOn(Infrastructure.getDefaultWorkerPool())`. Compute `eligible` before the lambda.

Updated `select()` method in `SemanticAgentRoutingStrategy.java`:

```java
@Override
public Uni<AgentAssignment> select(
    final AgentRoutingContext context, final List<AgentCandidate> candidates) {
  if (candidates.isEmpty()) {
    return Uni.createFrom().item(AgentAssignment.unresolvable());
  }

  final TrustRoutingPolicy policy = policyProvider.forCapability(context.capabilityName());
  final List<ClassifiedCandidate> classified =
      classifier.classify(candidates, context.capabilityName(), policy, cache);

  // Bootstrap guard: pre-screen before entering worker pool — avoids embedding cost
  if (policy.bootstrapEscalationRequired()) {
    final boolean hasQualified =
        classified.stream().anyMatch(c -> c.phase() == Phase.QUALIFIED);
    final boolean hasBootstrap =
        classified.stream().anyMatch(c -> c.phase() == Phase.BOOTSTRAP);
    if (!hasQualified && hasBootstrap) {
      return Uni.createFrom()
          .item(
              AgentAssignment.escalate(
                  context.capabilityName(), EscalationReason.NO_QUALIFIED_AGENT));
    }
  }

  // Strip BOOTSTRAP before entering worker pool; lambda captures eligible (not classified)
  final List<ClassifiedCandidate> eligible =
      policy.bootstrapEscalationRequired()
          ? classified.stream().filter(c -> c.phase() != Phase.BOOTSTRAP).toList()
          : classified;

  return Uni.createFrom()
      .voidItem()
      .emitOn(Infrastructure.getDefaultWorkerPool())
      .map(
          ignored -> {
            final String queryText =
                extractQueryText(context.caseContext(), context.capabilityName());
            final float[] queryVector = embeddingCache.getOrCompute(queryText, embeddingProvider);

            final List<ScoredCandidate> scored = new ArrayList<>(eligible.size());
            for (final ClassifiedCandidate cc : eligible) {
              scored.add(new ScoredCandidate(cc, score(cc, queryVector, policy)));
            }

            return classifier.decide(eligible, scored, context.capabilityName());
          });
}
```

Add import to `SemanticAgentRoutingStrategy.java`:
```java
import io.casehub.api.spi.routing.EscalationReason;
```

- [ ] **Step 7.4: Run all engine-ai tests — verify pass**

```bash
cd /Users/mdproctor/claude/casehub/engine
TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl engine-ai
```

Expected: ALL tests pass.

- [ ] **Step 7.5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add \
  engine-ai/src/main/java/io/casehub/engine/ai/routing/SemanticAgentRoutingStrategy.java \
  engine-ai/src/test/java/io/casehub/engine/ai/routing/SemanticAgentRoutingStrategyTest.java
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(engine#415): add bootstrap pre-screen + stripping to SemanticAgentRoutingStrategy

Pre-screen fires before emitOn(workerPool) — avoids embedding cost when guard fires.
Eligible list computed before lambda; BOOTSTRAP candidates excluded from embedding.

Refs #415"
```

---

## Task 8: Update `AgentRoutingEscalationHandler` — metric log placement and message switch

**Files:**
- Modify: `runtime/src/test/java/io/casehub/engine/internal/engine/handler/AgentRoutingEscalationHandlerTest.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/AgentRoutingEscalationHandler.java`

- [ ] **Step 8.1: Write failing handler tests**

In `AgentRoutingEscalationHandlerTest.java`:

1. Add imports:
```java
import io.casehub.api.spi.routing.EscalationReason;
import static org.mockito.ArgumentMatchers.contains;
```

2. Add two new test methods:

```java
@Test
void noQualifiedAgent_channelFound_postsQueryWithNoQualifiedMessage() {
  final UUID caseId = UUID.randomUUID();
  final String oversightName = CaseChannel.oversightChannelName(caseId);
  final CaseChannel oversight = channel(oversightName);
  when(channelProvider.listChannels(caseId)).thenReturn(List.of(oversight));

  handler.handle(new AgentRoutingEscalationEvent(
      caseId, "merge-executor", "merge-binding", EscalationReason.NO_QUALIFIED_AGENT));

  verify(channelProvider)
      .postToChannel(
          eq(oversight),
          eq("casehub-engine"),
          contains("No trust-qualified agent"),
          eq(MessageType.QUERY),
          eq(null),
          eq(null));
}

@Test
void noQualifiedAgent_noChannel_doesNotPostQueryButHandlesGracefully() {
  // Regression test: metric log fires unconditionally before channel search.
  // Even with no channel, handle() completes without error and posts nothing.
  final UUID caseId = UUID.randomUUID();
  when(channelProvider.listChannels(caseId)).thenReturn(List.of());

  handler.handle(new AgentRoutingEscalationEvent(
      caseId, "merge-executor", "merge-binding", EscalationReason.NO_QUALIFIED_AGENT));

  verify(channelProvider, never()).postToChannel(any(), any(), any(), any(), any(), any());
}
```

- [ ] **Step 8.2: Run failing tests**

```bash
cd /Users/mdproctor/claude/casehub/engine
mvn install -DskipTests -q
TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl runtime -Dtest=AgentRoutingEscalationHandlerTest
```

Expected: New tests fail (constructor mismatch on `AgentRoutingEscalationEvent` or incorrect message). Existing tests also fail on constructor arity (already fixed in Task 4). If Task 4 is done, only the new 2 tests fail on logic.

- [ ] **Step 8.3: Implement updated handler**

Replace the `handle()` and `postQuery()` methods in `AgentRoutingEscalationHandler.java`:

```java
@ConsumeEvent(value = EventBusAddresses.AGENT_ROUTING_ESCALATION, blocking = true)
public void handle(final AgentRoutingEscalationEvent event) {
  // Metric log fires unconditionally — before channel search
  // This ensures the alert fires even when no oversight channel is open
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
          () ->
              LOG.warnf(
                  "[METRIC:escalation.no-oversight-channel] caseId=%s capability=%s binding=%s"
                      + " — escalation absorbed; no oversight channel open."
                      + " PlanItem stays PENDING indefinitely. engine#383 tracks response handling.",
                  event.caseId(), event.capabilityName(), event.bindingName()));
}

private void postQuery(final CaseChannel channel, final AgentRoutingEscalationEvent event) {
  final String message =
      switch (event.reason()) {
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

  LOG.infof(
      "Agent routing escalation: QUERY posted to oversight channel '%s' for"
          + " caseId=%s capability=%s reason=%s",
      channel.name(), event.caseId(), event.capabilityName(), event.reason());
}
```

Add import to `AgentRoutingEscalationHandler.java`:
```java
import io.casehub.api.spi.routing.EscalationReason;
```

- [ ] **Step 8.4: Run all runtime tests**

```bash
cd /Users/mdproctor/claude/casehub/engine
TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl runtime
```

Expected: ALL tests pass.

- [ ] **Step 8.5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add \
  runtime/src/main/java/io/casehub/engine/internal/engine/handler/AgentRoutingEscalationHandler.java \
  runtime/src/test/java/io/casehub/engine/internal/engine/handler/AgentRoutingEscalationHandlerTest.java
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(engine#415): update AgentRoutingEscalationHandler — per-reason messages, unconditional metric log

[METRIC:escalation.no-qualified-agent] now fires before channel lookup so it fires
even when no oversight channel is open. Message text accurate per EscalationReason.

Refs #415"
```

---

## Task 9: Full build verification

- [ ] **Step 9.1: Run full build with all tests**

```bash
cd /Users/mdproctor/claude/casehub/engine
mvn install -DskipTests -q
TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl api,common,ledger,engine-ai,runtime
```

Expected: BUILD SUCCESS, all tests green across all five modules.

- [ ] **Step 9.2: If any test fails, read the failure output and fix before proceeding**

The most likely failures:
- Constructor arity mismatch on `TrustRoutingPolicy` — search for remaining 5-arg calls: `grep -rn "new TrustRoutingPolicy(" . --include="*.java"`
- Constructor arity mismatch on `AgentRoutingEscalationEvent` — search: `grep -rn "new AgentRoutingEscalationEvent(" . --include="*.java"`
- Single-arg `escalate()` call remaining — search: `grep -rn "\.escalate(\"" . --include="*.java"`

Fix and commit any remaining issues with message `"fix(engine#415): fix remaining constructor call sites"`.

- [ ] **Step 9.3: Push branch**

```bash
git -C /Users/mdproctor/claude/casehub/engine push -u origin issue-415-bootstrap-fallback-type
```
