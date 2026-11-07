# Conductor → SUPERVISOR Overlap Resolution — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #1294 — resolve Conductor → PlatformRole.SUPERVISOR overlap
**Issue group:** #1294 (part of #1281 epic)

**Goal:** Formalise the relationship between the conductor pattern and PlatformRole.SUPERVISOR via a shared marker interface, making the conceptual overlap explicit in types and Javadoc.

**Architecture:** Add a bare `SupervisorGateBehavior` marker interface in `api/model/coordination` alongside `PlatformRole`. `OversightGateService` extends it; `ConductorInboxManager` implements it. Javadoc cross-references tie the three approval-gating mechanisms together. No behavioral changes.

**Tech Stack:** Java 21, Quarkus 3.32.2

## Global Constraints

- No behavioral changes — only type hierarchy additions and Javadoc updates
- No new dependencies between modules
- `SupervisorGateBehavior` must be visible from both `api` and `runtime-core`

---

## Batch 1: Marker interface and type hierarchy

### Task 1: Create SupervisorGateBehavior and wire OversightGateService

**Files:**
- Create: `api/src/main/java/io/casehub/api/model/coordination/SupervisorGateBehavior.java`
- Modify: `api/src/main/java/io/casehub/api/spi/OversightGateService.java:20`
- Test: `api/src/test/java/io/casehub/api/model/coordination/SupervisorGateBehaviorTest.java`

**Interfaces:**
- Consumes: `PlatformRole.SUPERVISOR` (existing)
- Produces: `SupervisorGateBehavior` marker interface — consumed by Task 2

- [ ] **Step 1: Write the failing test**

Create `SupervisorGateBehaviorTest.java`:

```java
package io.casehub.api.model.coordination;

import static org.assertj.core.api.Assertions.assertThat;

import io.casehub.api.spi.OversightGateService;
import org.junit.jupiter.api.Test;

class SupervisorGateBehaviorTest {

  @Test
  void oversightGateService_extends_supervisor_gate_behavior() {
    assertThat(SupervisorGateBehavior.class)
        .isAssignableFrom(OversightGateService.class);
  }

  @Test
  void actionRiskClassifier_does_not_extend_supervisor_gate_behavior() {
    assertThat(SupervisorGateBehavior.class)
        .isNotAssignableFrom(io.casehub.api.spi.ActionRiskClassifier.class);
  }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=SupervisorGateBehaviorTest -q`
Expected: FAIL — `SupervisorGateBehavior` does not exist

- [ ] **Step 3: Create SupervisorGateBehavior marker interface**

Use `ide_create_file` or write to `api/src/main/java/io/casehub/api/model/coordination/SupervisorGateBehavior.java`:

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
package io.casehub.api.model.coordination;

/**
 * Marker for components that implement {@link PlatformRole#SUPERVISOR} gating behavior — the
 * ability to pause execution, wait for approval, and resume with a decision.
 *
 * <p>Implementors operate at different granularities:
 *
 * <ul>
 *   <li>{@link io.casehub.api.spi.OversightGateService} — worker-action level, gates individual
 *       {@code PlannedAction}s
 *   <li>{@code ConductorInboxManager} — evolution-pipeline level, gates research pipeline stages
 * </ul>
 *
 * <p>Classification components that decide <em>whether</em> gating is needed (such as {@link
 * io.casehub.api.spi.ActionRiskClassifier}) are not gate implementations and do not carry this
 * marker. They are the triage step that precedes the gate.
 *
 * @see PlatformRole#SUPERVISOR
 */
public interface SupervisorGateBehavior {}
```

- [ ] **Step 4: Add extends SupervisorGateBehavior to OversightGateService**

In `api/src/main/java/io/casehub/api/spi/OversightGateService.java`, change:

```java
// Before:
public interface OversightGateService {

// After:
import io.casehub.api.model.coordination.SupervisorGateBehavior;

public interface OversightGateService extends SupervisorGateBehavior {
```

Also update the Javadoc on `OversightGateService`:

```java
/**
 * Engine SPI for approval-gating agent actions. Implements {@link SupervisorGateBehavior} — the
 * {@link io.casehub.api.model.coordination.PlatformRole#SUPERVISOR} gating contract at the
 * worker-action level.
 *
 * <p>{@link ActionRiskClassifier} determines whether a {@link io.casehub.worker.api.PlannedAction}
 * requires gating. When gating is required, this service manages the gate lifecycle.
 */
```

- [ ] **Step 5: Run test to verify it passes**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=SupervisorGateBehaviorTest -q`
Expected: PASS — both assertions hold

- [ ] **Step 6: Verify NoOpOversightGateService inherits transitively**

Add to `SupervisorGateBehaviorTest.java`:

```java
import io.casehub.engine.runtime.worker.NoOpOversightGateService;

@Test
void noOpOversightGateService_inherits_marker_transitively() {
  assertThat(SupervisorGateBehavior.class)
      .isAssignableFrom(NoOpOversightGateService.class);
}
```

This test lives in the api module but references a runtime-core class. If that cross-module reference is not on the test classpath, move this assertion into the runtime-core test in Task 2 instead.

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=SupervisorGateBehaviorTest -q`
Expected: PASS (or compile error if cross-module — handle in Task 2)

- [ ] **Step 7: Commit**

```bash
git add api/src/main/java/io/casehub/api/model/coordination/SupervisorGateBehavior.java api/src/main/java/io/casehub/api/spi/OversightGateService.java api/src/test/java/io/casehub/api/model/coordination/SupervisorGateBehaviorTest.java
git commit -m "feat: add SupervisorGateBehavior marker, wire OversightGateService

Refs #1294"
```

### Task 2: Wire ConductorInboxManager and add runtime-core tests

**Files:**
- Modify: `runtime-core/src/main/java/io/casehub/engine/runtime/improvement/ConductorInboxManager.java:27`
- Test: `runtime-core/src/test/java/io/casehub/engine/runtime/improvement/ConductorInboxManagerTest.java` (modify existing)

**Interfaces:**
- Consumes: `SupervisorGateBehavior` (from Task 1)
- Produces: nothing — terminal task in batch

- [ ] **Step 1: Write the failing test**

Add to existing `ConductorInboxManagerTest.java`:

```java
import io.casehub.api.model.coordination.SupervisorGateBehavior;

@Test
void implements_supervisor_gate_behavior() {
  assertThat(manager).isInstanceOf(SupervisorGateBehavior.class);
}
```

- [ ] **Step 2: Run test to verify it fails**

First install the api module with the new marker:
Run: `mvn install -DskipTests -q`

Then run the test:
Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime-core -Dtest=ConductorInboxManagerTest#implements_supervisor_gate_behavior -q`
Expected: FAIL — `ConductorInboxManager` does not implement `SupervisorGateBehavior`

- [ ] **Step 3: Add implements SupervisorGateBehavior to ConductorInboxManager**

In `runtime-core/src/main/java/io/casehub/engine/runtime/improvement/ConductorInboxManager.java`, change:

```java
// Before:
public class ConductorInboxManager {

// After:
import io.casehub.api.model.coordination.SupervisorGateBehavior;

public class ConductorInboxManager implements SupervisorGateBehavior {
```

Also update/add Javadoc:

```java
/**
 * Manages the evolution conductor's approval inbox — the {@link SupervisorGateBehavior}
 * implementation for evolution-pipeline gates.
 *
 * <p>Operates at the pipeline-stage level (research scope, hypothesis approval, PR review) with
 * evolution-specific context: escalation triggers, watch patterns, confidence scores, categories,
 * and area IDs.
 *
 * <p>Complementary to {@link io.casehub.api.spi.OversightGateService}, which provides {@link
 * SupervisorGateBehavior} at the worker-action level for individual {@code PlannedAction}s.
 *
 * @see io.casehub.api.model.coordination.PlatformRole#SUPERVISOR
 */
```

- [ ] **Step 4: Run test to verify it passes**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime-core -Dtest=ConductorInboxManagerTest#implements_supervisor_gate_behavior -q`
Expected: PASS

- [ ] **Step 5: Add NoOp transitive inheritance test (if not handled in Task 1)**

If the cross-module test in Task 1 Step 6 failed to compile, add it here in a new or existing runtime-core test:

```java
import io.casehub.api.model.coordination.SupervisorGateBehavior;
import io.casehub.engine.runtime.worker.NoOpOversightGateService;

@Test
void noOpOversightGateService_inherits_marker_transitively() {
  assertThat(SupervisorGateBehavior.class)
      .isAssignableFrom(NoOpOversightGateService.class);
}
```

- [ ] **Step 6: Run full test suite for both modules**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api,runtime-core -q`
Expected: All tests PASS — no regressions

- [ ] **Step 7: Commit**

```bash
git add runtime-core/src/main/java/io/casehub/engine/runtime/improvement/ConductorInboxManager.java runtime-core/src/test/java/io/casehub/engine/runtime/improvement/ConductorInboxManagerTest.java
git commit -m "feat: wire ConductorInboxManager to SupervisorGateBehavior

Refs #1294"
```

## Batch 2: Javadoc cross-references

### Task 3: Update Javadoc on PlatformRole and ActionRiskClassifier

**Files:**
- Modify: `api/src/main/java/io/casehub/api/model/coordination/PlatformRole.java:18-22` (Javadoc only)
- Modify: `api/src/main/java/io/casehub/api/spi/ActionRiskClassifier.java:21-33` (Javadoc only)

**Interfaces:**
- Consumes: `SupervisorGateBehavior` (from Task 1) — referenced in Javadoc
- Produces: nothing — documentation-only changes

- [ ] **Step 1: Add Javadoc to PlatformRole enum and SUPERVISOR constant**

In `api/src/main/java/io/casehub/api/model/coordination/PlatformRole.java`, add class-level and constant-level Javadoc:

```java
/**
 * Intrinsic behavioral contracts — the platform implements specific behavior for these roles.
 *
 * @see AgentRole
 * @see StructuralRole
 */
public enum PlatformRole {
  COORDINATOR(StructuralRole.INITIATOR),

  /**
   * Oversees, approves/rejects, gates progression. Implemented at two granularities via {@link
   * SupervisorGateBehavior}:
   *
   * <ul>
   *   <li>{@link io.casehub.api.spi.OversightGateService} — worker-action gates for {@code
   *       PlannedAction} approval
   *   <li>{@code ConductorInboxManager} — evolution-pipeline gates for research stage approval
   * </ul>
   *
   * <p>Classification (deciding <em>whether</em> gating is needed) is handled by {@link
   * io.casehub.api.spi.ActionRiskClassifier}, which precedes the gate.
   */
  SUPERVISOR(StructuralRole.INITIATOR),

  JUDGE(StructuralRole.PEER),
  MODERATOR(StructuralRole.PEER);
```

- [ ] **Step 2: Update ActionRiskClassifier Javadoc**

In `api/src/main/java/io/casehub/api/spi/ActionRiskClassifier.java`, replace the existing Javadoc:

```java
/**
 * Classifies a worker's planned action to determine whether it may proceed autonomously or must be
 * gated for human approval.
 *
 * <p>This is the classification step in the {@link
 * io.casehub.api.model.coordination.PlatformRole#SUPERVISOR} pipeline — it decides <em>whether</em>
 * gating is needed. The gate itself is managed by {@link OversightGateService}, which implements
 * {@link io.casehub.api.model.coordination.SupervisorGateBehavior}.
 *
 * <p>Implementations must be annotated {@link RiskClassifier} and {@code @ApplicationScoped}. The
 * engine chains all registered implementations — the most restrictive result wins. Multiple
 * consumer repos can provide classifiers simultaneously without conflict.
 *
 * <p>The engine chains all registered implementations via {@link ChainedActionRiskClassifier},
 * which applies most-restrictive-wins semantics.
 *
 * <p>If {@code classify} throws, the engine applies a fail-safe {@link RiskDecision.GateRequired}
 * requiring manual review. Do not throw to bypass the gate — the fail-safe will catch it.
 */
```

- [ ] **Step 3: Verify compilation**

Run: `mvn compile -pl api -q`
Expected: Compiles with no errors — Javadoc references resolve

- [ ] **Step 4: Commit**

```bash
git add api/src/main/java/io/casehub/api/model/coordination/PlatformRole.java api/src/main/java/io/casehub/api/spi/ActionRiskClassifier.java
git commit -m "docs: add SUPERVISOR cross-references to PlatformRole and ActionRiskClassifier

Refs #1294"
```

- [ ] **Step 5: Run full test suite**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api,runtime-core -q`
Expected: All tests PASS — Javadoc-only changes have no behavioral impact

## References

- [2026-11-07-conductor-supervisor-overlap-design.md] — design spec this plan implements
- `api/src/main/java/io/casehub/api/model/coordination/PlatformRole.java:18-33` — SUPERVISOR constant
- `api/src/main/java/io/casehub/api/spi/OversightGateService.java:20-24` — generic gate SPI
- `runtime-core/src/main/java/io/casehub/engine/runtime/improvement/ConductorInboxManager.java:27-90` — evolution gate
- `api/src/main/java/io/casehub/api/spi/ActionRiskClassifier.java:34-37` — risk classifier
- `runtime-core/src/main/java/io/casehub/engine/runtime/worker/NoOpOversightGateService.java:24` — NoOp impl
- `api/src/test/java/io/casehub/api/model/coordination/AgentRoleTest.java` — existing coordination tests
- `runtime-core/src/test/java/io/casehub/engine/runtime/improvement/ConductorInboxManagerTest.java` — existing conductor tests
- GitHub #1294 — focal issue
- GitHub #1281 — parent epic
- GitHub #1282 — coordination vocabulary spec
