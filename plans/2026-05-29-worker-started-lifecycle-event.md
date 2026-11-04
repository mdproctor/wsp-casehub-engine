# WorkerStarted CaseLifecycleEvent + ProvisionResult Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fire a `WorkerStarted` `CaseLifecycleEvent` from `tryProvision()` when an external provisioner succeeds, using a new `ProvisionResult` SPI return type that keeps `Worker` as a pure definition artifact.

**Architecture:** Introduce `ProvisionResult(UUID causedByEntryId)` as the provisioner SPI return type (replacing `Worker`). `CaseContextChangedEventHandler.tryProvision()` captures `traceId` before the reactive chain, calls `provision()`, and fires `CaseLifecycleEvent("WorkerStarted")` on success. `CaseLifecycleEvent` stays at 7 fields — no claudony-specific fields on the shared event.

**Tech Stack:** Java 21, Quarkus 3.32.x, Mutiny (Uni), CDI (`Event.fireAsync`), Mockito 5, JUnit 5, AssertJ.

---

## File Map

| Action | File |
|--------|------|
| Create | `api/src/main/java/io/casehub/api/spi/ProvisionResult.java` |
| Modify | `api/src/main/java/io/casehub/api/spi/WorkerProvisioner.java` |
| Modify | `api/src/main/java/io/casehub/api/spi/ReactiveWorkerProvisioner.java` |
| Modify | `runtime/src/main/java/io/casehub/engine/internal/worker/NoOpWorkerProvisioner.java` |
| Modify | `runtime/src/main/java/io/casehub/engine/internal/worker/NoOpReactiveWorkerProvisioner.java` |
| Modify | `runtime/src/main/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandler.java` |
| Modify | `api/src/test/java/io/casehub/api/spi/WorkerProvisionerContractTest.java` |
| Modify | `api/src/test/java/io/casehub/api/spi/ReactiveWorkerProvisionerContractTest.java` |
| Modify | `runtime/src/test/java/io/casehub/engine/internal/worker/DefaultWorkerSpiImplementationsTest.java` |
| Modify | `runtime/src/test/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandlerRoutingTest.java` |

---

## Task 1: Create `ProvisionResult` + unit test

**Files:**
- Create: `api/src/main/java/io/casehub/api/spi/ProvisionResult.java`
- Create: `api/src/test/java/io/casehub/api/spi/ProvisionResultTest.java`

- [ ] **Step 1.1: Write the failing test**

Create `api/src/test/java/io/casehub/api/spi/ProvisionResultTest.java`:

```java
package io.casehub.api.spi;

import static org.assertj.core.api.Assertions.assertThat;
import java.util.UUID;
import org.junit.jupiter.api.Test;

class ProvisionResultTest {

  @Test
  void empty_causedByEntryId_isNull() {
    assertThat(ProvisionResult.empty().causedByEntryId()).isNull();
  }

  @Test
  void withId_roundtrips() {
    UUID id = UUID.randomUUID();
    assertThat(new ProvisionResult(id).causedByEntryId()).isEqualTo(id);
  }

  @Test
  void empty_isEquivalentToNullConstructor() {
    assertThat(ProvisionResult.empty()).isEqualTo(new ProvisionResult(null));
  }
}
```

- [ ] **Step 1.2: Run test — expect compilation failure**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest="ProvisionResultTest" -q 2>&1 | tail -5
```

Expected: `COMPILATION ERROR` — `ProvisionResult` does not exist yet.

- [ ] **Step 1.3: Create `ProvisionResult`**

Create `api/src/main/java/io/casehub/api/spi/ProvisionResult.java`:

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
package io.casehub.api.spi;

import java.util.UUID;

/**
 * Outcome of a successful {@link WorkerProvisioner#provision} or {@link
 * ReactiveWorkerProvisioner#provision} call.
 *
 * <p>{@code causedByEntryId} is the ledger entry ID of the COMMAND that triggered provisioning.
 * Provisioner implementations that can resolve this ID (e.g. by correlating against a Qhorus
 * message ledger) set it here; implementations that cannot leave it {@code null}. The engine passes
 * it through to the audit event so that ledger observers can establish causal linkage without
 * round-tripping through the engine's internal state.
 *
 * <p>Will be non-null only after engine#231 threads Qhorus trigger context (channelId +
 * correlationId) through {@link io.casehub.api.model.ProvisionContext}.
 */
public record ProvisionResult(UUID causedByEntryId) {

  /** Convenience factory for provisioners that do not resolve a causal ledger entry. */
  public static ProvisionResult empty() {
    return new ProvisionResult(null);
  }
}
```

- [ ] **Step 1.4: Run test — expect green**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest="ProvisionResultTest" -q 2>&1 | tail -5
```

Expected: `BUILD SUCCESS`, `Tests run: 3, Failures: 0, Errors: 0`.

---

## Task 2: Update provisioner SPI interfaces

**Files:**
- Modify: `api/src/main/java/io/casehub/api/spi/WorkerProvisioner.java`
- Modify: `api/src/main/java/io/casehub/api/spi/ReactiveWorkerProvisioner.java`

This task intentionally breaks compilation — every `provision()` implementor and call site will fail until Tasks 3–6 fix them.

- [ ] **Step 2.1: Update `WorkerProvisioner.provision()` return type**

In `api/src/main/java/io/casehub/api/spi/WorkerProvisioner.java`, replace:

```java
import io.casehub.api.model.Worker;
```

with no import change needed (remove `Worker` import if it becomes unused after this edit), and replace the method signature and Javadoc:

```java
  /**
   * Provision a new worker with the given capabilities.
   *
   * @param capabilities required capability set for the PlanItem
   * @param context case context, pre-built worker context, and propagation
   * @return the provisioning outcome, including optional causal ledger entry linkage
   * @throws ProvisioningException if the worker cannot be started
   */
  ProvisionResult provision(Set<String> capabilities, ProvisionContext context);
```

Remove the `import io.casehub.api.model.Worker;` line (it is no longer used in this interface).

- [ ] **Step 2.2: Update `ReactiveWorkerProvisioner.provision()` return type**

In `api/src/main/java/io/casehub/api/spi/ReactiveWorkerProvisioner.java`, replace the `provision` method signature:

```java
  /**
   * Provision a new worker reactively.
   *
   * @param capabilities required capability set
   * @param context case context, pre-built worker context, and propagation
   * @return {@code Uni} emitting the provisioning outcome on success, or failing with
   *     {@link ProvisioningException}
   */
  Uni<ProvisionResult> provision(Set<String> capabilities, ProvisionContext context);
```

Remove `import io.casehub.api.model.Worker;` if it is no longer used.

- [ ] **Step 2.3: Verify compilation fails predictably**

```bash
mvn compile -pl api,runtime,scheduler-quartz,ledger 2>&1 | grep -E "error:|BUILD" | head -20
```

Expected: compilation errors in `NoOpWorkerProvisioner`, `NoOpReactiveWorkerProvisioner`, the contract test stubs, and `CaseContextChangedEventHandler`. This is the intended state — proceed to Task 3.

---

## Task 3: Fix no-op default implementations

**Files:**
- Modify: `runtime/src/main/java/io/casehub/engine/internal/worker/NoOpWorkerProvisioner.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/worker/NoOpReactiveWorkerProvisioner.java`

The no-ops still throw `ProvisioningException` on `provision()` — that behaviour is correct and unchanged. Only the declared return type changes.

- [ ] **Step 3.1: Fix `NoOpWorkerProvisioner`**

Replace the `provision` method (remove the `Worker` import and update the return type):

```java
  @Override
  public ProvisionResult provision(Set<String> capabilities, ProvisionContext context) {
    throw new ProvisioningException(
        "No WorkerProvisioner configured — add an @ApplicationScoped @Priority(1) WorkerProvisioner implementation");
  }
```

Add import at the top of the file:
```java
import io.casehub.api.spi.ProvisionResult;
```

Remove `import io.casehub.api.model.Worker;` if no longer needed.

- [ ] **Step 3.2: Fix `NoOpReactiveWorkerProvisioner`**

In `runtime/src/main/java/io/casehub/engine/internal/worker/NoOpReactiveWorkerProvisioner.java`, replace the `provision` method:

```java
  @Override
  public Uni<ProvisionResult> provision(Set<String> capabilities, ProvisionContext context) {
    return Uni.createFrom()
        .failure(
            new ProvisioningException(
                "No ReactiveWorkerProvisioner configured — add an @ApplicationScoped @Priority(1) ReactiveWorkerProvisioner implementation"));
  }
```

Add import: `import io.casehub.api.spi.ProvisionResult;`. Remove `import io.casehub.api.model.Worker;` if unused.

---

## Task 4: Fix contract tests + no-op SPI tests

**Files:**
- Modify: `api/src/test/java/io/casehub/api/spi/WorkerProvisionerContractTest.java`
- Modify: `api/src/test/java/io/casehub/api/spi/ReactiveWorkerProvisionerContractTest.java`
- Modify: `runtime/src/test/java/io/casehub/engine/internal/worker/DefaultWorkerSpiImplementationsTest.java`

- [ ] **Step 4.1: Fix `WorkerProvisionerContractTest.NoOpStub`**

In `WorkerProvisionerContractTest`, update the inner stub class `NoOpStub.provision()`:

```java
    @Override
    public ProvisionResult provision(Set<String> capabilities, ProvisionContext context) {
      throw new ProvisioningException("NoOp — no provisioner configured");
    }
```

Remove `import io.casehub.api.model.Worker;`. Add nothing — `ProvisionResult` is in the same package.

The existing test `noOp_provision_throwsProvisioningException` does not need its assertion changed — it still asserts `ProvisioningException` is thrown.

- [ ] **Step 4.2: Fix `ReactiveWorkerProvisionerContractTest.NoOpStub`**

Update the `NoOpStub.provision()` in `ReactiveWorkerProvisionerContractTest`:

```java
    @Override
    public Uni<ProvisionResult> provision(Set<String> capabilities, ProvisionContext context) {
      return Uni.createFrom().failure(new ProvisioningException("NoOp"));
    }
```

Remove `import io.casehub.api.model.Worker;`. Add `import io.casehub.api.spi.ProvisionResult;`.

The existing test `noOp_provision_uniFails_withProvisioningException` still asserts `ProvisioningException` — no assertion change needed.

- [ ] **Step 4.3: Fix `DefaultWorkerSpiImplementationsTest`**

`DefaultWorkerSpiImplementationsTest.noOpProvisioner_provision_throwsProvisioningException` instantiates `NoOpWorkerProvisioner` and asserts it throws. The assertion is unchanged. Only compilation needs fixing — no import or assertion changes are needed since the test doesn't capture the return value.

Verify this file compiles as-is after Tasks 3.1. If there are any lingering `Worker` imports, remove them.

- [ ] **Step 4.4: Run api + runtime tests — expect green**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api,runtime 2>&1 | grep -E "Tests run:|BUILD" | tail -10
```

Expected: `BUILD SUCCESS`, all tests passing. The `CaseContextChangedEventHandler` still uses the old `replaceWithVoid()` call — the routing test will compile once Task 5 fixes the mocks.

---

## Task 5: Update `CaseContextChangedEventHandlerRoutingTest` (mocks + `WorkerStarted` test)

**Files:**
- Modify: `runtime/src/test/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandlerRoutingTest.java`

- [ ] **Step 5.1: Add missing mock fields**

Add these two `@Mock` fields immediately after the existing `@Mock ReactiveWorkerProvisioner reactiveWorkerProvisioner;` declaration:

```java
  @Mock jakarta.enterprise.event.Event<io.casehub.engine.common.spi.event.CaseLifecycleEvent> lifecycleEvents;
  @Mock io.casehub.ledger.api.spi.LedgerTraceIdProvider traceIdProvider;
```

- [ ] **Step 5.2: Add traceId stub in `@BeforeEach`**

At the end of the `setUp()` method (before the closing brace), add:

```java
    when(traceIdProvider.currentTraceId()).thenReturn(java.util.Optional.empty());
```

- [ ] **Step 5.3: Add `WorkerStarted` event test**

Add this test after the existing `routing_escalateToOversight_publishesEscalationEvent` test (before the closing `}`):

```java
  @Test
  void tryProvision_provisionerHasCapability_firesWorkerStartedLifecycleEvent() {
    // Routing: no pre-defined workers match, falls through to tryProvision
    when(agentRoutingStrategy.select(any(), any()))
        .thenReturn(Uni.createFrom().item(AgentAssignment.unresolvable()));

    // Provisioner claims the "research" capability — tryProvision will call provision()
    when(reactiveWorkerProvisioner.getCapabilities())
        .thenReturn(Uni.createFrom().item(java.util.Set.of("research")));

    when(reactiveWorkerContextProvider.buildContext(any(), any(), any()))
        .thenReturn(
            Uni.createFrom()
                .item(
                    new io.casehub.api.model.WorkerContext(
                        "desc",
                        null,
                        null,
                        java.util.List.of(),
                        io.casehub.api.context.PropagationContext.createRoot(),
                        java.util.Map.of())));

    when(reactiveWorkerProvisioner.provision(any(), any()))
        .thenReturn(Uni.createFrom().item(io.casehub.api.spi.ProvisionResult.empty()));

    handler
        .onCaseStateContextChangedEventHandler(
            new CaseContextChangedEvent(caseInstance, NullNode.instance))
        .await()
        .indefinitely();

    verify(lifecycleEvents)
        .fireAsync(
            argThat(
                e ->
                    caseInstance.getUuid().equals(e.caseId())
                        && "WorkerStarted".equals(e.eventType())
                        && "ProvisionWorker".equals(e.commandType())
                        && "RUNNING".equals(e.caseStatus())
                        && "System".equals(e.actorRole())));
    verify(eventBus, never())
        .publish(eq(EventBusAddresses.WORKER_SCHEDULE), any(WorkerScheduleEvent.class));
  }
```

Add missing import at the top of the test file:
```java
import static org.mockito.ArgumentMatchers.argThat;
```

- [ ] **Step 5.4: Run the new test — expect failure (handler not yet updated)**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest="CaseContextChangedEventHandlerRoutingTest#tryProvision_provisionerHasCapability_firesWorkerStartedLifecycleEvent" -q 2>&1 | tail -10
```

Expected: `FAILED` — `lifecycleEvents.fireAsync()` was never called (the handler still calls `.replaceWithVoid()`). If it fails with a compilation error instead, fix any import issues and re-run.

---

## Task 6: Update `CaseContextChangedEventHandler`

**Files:**
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandler.java`

- [ ] **Step 6.1: Add two `@Inject` fields**

After the existing `@Inject ReactiveWorkerProvisioner reactiveWorkerProvisioner;` field, add:

```java
  @Inject Event<CaseLifecycleEvent> lifecycleEvents;

  @Inject LedgerTraceIdProvider traceIdProvider;
```

Add imports at the top of the file:
```java
import io.casehub.engine.common.spi.event.CaseLifecycleEvent;
import io.casehub.ledger.api.spi.LedgerTraceIdProvider;
import jakarta.enterprise.event.Event;
import io.casehub.api.spi.ProvisionResult;
```

- [ ] **Step 6.2: Rewrite `tryProvision()`**

Replace the entire `tryProvision` method with:

```java
  private Uni<Void> tryProvision(final CaseInstance caseInstance, final Capability capability) {
    final String traceId = traceIdProvider.currentTraceId().orElse(null);
    return reactiveWorkerProvisioner
        .getCapabilities()
        .flatMap(
            caps -> {
              if (!caps.contains(capability.getName())) {
                return Uni.createFrom().voidItem();
              }
              final Map<String, Object> inputData =
                  evalJqAsMap(
                      caseInstance.getCaseContext().asJsonNode(), capability.getInputSchema());
              final WorkRequest workRequest = WorkRequest.of(capability.getName(), inputData);
              return reactiveWorkerContextProvider
                  .buildContext(null, caseInstance.getUuid(), workRequest)
                  .flatMap(
                      workerContext -> {
                        final ProvisionContext provisionContext =
                            new ProvisionContext(
                                caseInstance.getUuid(),
                                capability.getName(),
                                workerContext,
                                PropagationContext.createRoot(),
                                null, // triggerChannelId — see engine#231
                                null); // triggerCorrelationId — see engine#231
                        return reactiveWorkerProvisioner
                            .provision(caps, provisionContext)
                            .flatMap(
                                result -> {
                                  lifecycleEvents.fireAsync(
                                      new CaseLifecycleEvent(
                                          caseInstance.getUuid(),
                                          "ProvisionWorker",
                                          "WorkerStarted",
                                          caseInstance.getState().name(),
                                          null,
                                          "System",
                                          traceId));
                                  return Uni.createFrom().voidItem();
                                });
                      });
            })
        .onFailure(ProvisioningException.class)
        .invoke(
            e ->
                LOG.warnf(
                    e,
                    "WorkerProvisioner failed for capability '%s' on case %s — binding remains eligible",
                    capability.getName(),
                    caseInstance.getUuid()));
  }
```

- [ ] **Step 6.3: Run the new test — expect green**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest="CaseContextChangedEventHandlerRoutingTest" -q 2>&1 | tail -10
```

Expected: `Tests run: 4, Failures: 0, Errors: 0, BUILD SUCCESS`.

---

## Task 7: Full test suite verification

- [ ] **Step 7.1: Install dependencies first**

```bash
mvn install -DskipTests -q
```

- [ ] **Step 7.2: Run api module tests**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api 2>&1 | grep -E "Tests run:|BUILD" | tail -10
```

Expected: `BUILD SUCCESS`.

- [ ] **Step 7.3: Run runtime module tests**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime 2>&1 | grep -E "Tests run:|BUILD" | tail -10
```

Expected: `BUILD SUCCESS`.

- [ ] **Step 7.4: Run ledger module tests**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl ledger 2>&1 | grep -E "Tests run:|BUILD" | tail -10
```

Expected: `BUILD SUCCESS` (ledger tests use `CaseLifecycleEvent` in test code — verify no compile errors from the 7-field record staying unchanged).

- [ ] **Step 7.5: Run scheduler-quartz module tests**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl scheduler-quartz 2>&1 | grep -E "Tests run:|BUILD" | tail -10
```

Expected: `BUILD SUCCESS`.

- [ ] **Step 7.6: Run blackboard module tests**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl blackboard 2>&1 | grep -E "Tests run:|BUILD" | tail -10
```

Expected: `BUILD SUCCESS`.

---

## Task 8: File claudony cross-repo issue

- [ ] **Step 8.1: File issue on casehubio/claudony**

```bash
gh issue create --repo casehubio/claudony --title "feat: wire causedByEntryId through provisioner after engine#389" --body "$(cat <<'EOF'
## Context

engine#389 introduces \`ProvisionResult(UUID causedByEntryId)\` as the return type of \`ReactiveWorkerProvisioner.provision()\`. It also fires \`CaseLifecycleEvent(\"WorkerStarted\")\` when provisioning succeeds.

\`causedByEntryId\` carries the ledger entry ID of the Qhorus COMMAND that triggered provisioning — enabling causal linkage in the audit trail.

## Required changes

### 1. Update \`ClaudonyReactiveWorkerProvisioner.provision()\` return type

Change \`Uni<Worker>\` to \`Uni<ProvisionResult>\`.

When \`provisionContext.triggerChannelId()\` and \`provisionContext.triggerCorrelationId()\` are both non-null (available after engine#231):
- Look up \`MessageLedgerEntry\` by \`(subjectId=triggerChannelId, correlationId=triggerCorrelationId)\` via the ledger EntityManager
- Store \`(caseId → resolvedEntryId)\` in a \`ConcurrentHashMap\` field on the provisioner bean
- Return \`new ProvisionResult(resolvedEntryId)\`

When either trigger field is null, store nothing and return \`ProvisionResult.empty()\`.

### 2. Update \`ClaudonyLedgerEventCapture.onCaseLifecycleEvent()\`

When observing a \`CaseLifecycleEvent\` with \`eventType = "WorkerStarted"\`:
- Drain the in-memory map by \`event.caseId()\`
- If a value is present, set \`entry.causedByEntryId = drainedValue\` before persisting

### Notes

- The map key should be \`caseId\` (one provisioning per case at a time is the invariant)
- Timing is safe: the provisioner stores before returning, the event fires after the Uni resolves — observer always runs after the store
- \`causedByEntryId\` will be null in \`ProvisionResult\` until engine#231 ships; the plumbing should be in place from day 1

## Depends on

- engine#389 (ProvisionResult SPI — ships first)
- engine#231 (triggerChannelId + triggerCorrelationId in ProvisionContext — required for non-null values)
EOF
)"
```

---

## Task 9: Commit

- [ ] **Step 9.1: Stage all changed files**

```bash
git -C /Users/mdproctor/claude/casehub/engine add \
  api/src/main/java/io/casehub/api/spi/ProvisionResult.java \
  api/src/main/java/io/casehub/api/spi/WorkerProvisioner.java \
  api/src/main/java/io/casehub/api/spi/ReactiveWorkerProvisioner.java \
  api/src/test/java/io/casehub/api/spi/ProvisionResultTest.java \
  api/src/test/java/io/casehub/api/spi/WorkerProvisionerContractTest.java \
  api/src/test/java/io/casehub/api/spi/ReactiveWorkerProvisionerContractTest.java \
  runtime/src/main/java/io/casehub/engine/internal/worker/NoOpWorkerProvisioner.java \
  runtime/src/main/java/io/casehub/engine/internal/worker/NoOpReactiveWorkerProvisioner.java \
  runtime/src/main/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandler.java \
  runtime/src/test/java/io/casehub/engine/internal/worker/DefaultWorkerSpiImplementationsTest.java \
  runtime/src/test/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandlerRoutingTest.java
```

- [ ] **Step 9.2: Verify staged diff stat**

```bash
git -C /Users/mdproctor/claude/casehub/engine diff --staged --stat
```

Expected: 11 files changed.

- [ ] **Step 9.3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine commit -m "$(cat <<'EOF'
feat(spi): ProvisionResult return type + WorkerStarted lifecycle event

Introduces ProvisionResult(UUID causedByEntryId) as the return type for
both WorkerProvisioner and ReactiveWorkerProvisioner.provision(), replacing
the raw Worker return. Worker remains a pure case-definition artifact.

CaseContextChangedEventHandler.tryProvision() now captures traceId before
the reactive chain (thread-local OTel context is intact at that point),
then fires CaseLifecycleEvent("WorkerStarted", commandType="ProvisionWorker")
when an external provisioner succeeds. CaseLifecycleEvent stays at 7 fields.

No-op defaults return ProvisionResult.empty() (still throw ProvisioningException
on provision() — that behaviour is unchanged). Contract tests and
DefaultWorkerSpiImplementationsTest updated for the new return type.

Closes #389
EOF
)"
```

- [ ] **Step 9.4: Confirm commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine log --oneline -1
```
