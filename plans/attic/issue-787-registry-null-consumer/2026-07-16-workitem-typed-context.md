# WorkItem Typed Context Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> executing-plans to implement this plan task-by-task. Each task follows
> TDD (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #689 — feat: typed context for WorkItem boundary via ContextBridge
**Issue group:** #689

**Goal:** Extend the ContextBridge protocol to the WorkItem boundary so
HumanTaskTarget can declare `payloadType` and `resolutionType` with
fail-fast bridge validation on both input and output paths.

**Architecture:** HumanTaskTarget gains two nullable Class<?> fields.
The engine validates inputMapping output against payloadType via
bridge.initialise() at dispatch time. On WorkItem completion, the
engine-adapter validates the resolution against resolutionType via
bridge.deserialise() before applying the PlanItem status transition.
The work repo stores type name strings as opaque metadata and echoes
them back — it stays bridge-agnostic.

**Tech Stack:** Java 21, Quarkus, Jackson, Vert.x EventBus, Flyway (work
repo only), JPA/Panache

**Cross-repo:** Changes span casehub/engine and casehub/work. Create a
branch `issue-689-workitem-typed-context` in the work repo before
starting Task 3.

## Global Constraints

- Pre-release platform — breaking changes cost nothing.
- No serialisation at internal boundaries (event bus carries data as-is).
- Bridge operations happen on the engine side only — work repo core does
  not depend on engine.
- Backward compatible: null payloadType/resolutionType = today's untyped
  behaviour.
- All bridge validation is fail-fast — never silent.

---

### Task 1: HumanTaskTarget type declarations + YAML schema

**Files:**
- Modify: `api/src/main/java/io/casehub/api/model/HumanTaskTarget.java`
- Modify: `schema/src/main/resources/schema/CaseDefinition.yaml`
- Modify: `api/src/main/java/io/casehub/api/model/converter/CaseDefinitionYamlMapper.java`
- Test: `api/src/test/java/io/casehub/api/model/HumanTaskTargetTest.java`

**Interfaces:**
- Produces: `HumanTaskTarget.payloadType()` → `Class<?>` (nullable),
  `HumanTaskTarget.resolutionType()` → `Class<?>` (nullable),
  `HumanTaskTarget.Builder.payloadType(Class<?>)`,
  `HumanTaskTarget.Builder.resolutionType(Class<?>)`

- [ ] **Step 1: Write failing tests for payloadType/resolutionType on HumanTaskTarget**

Add to `HumanTaskTargetTest.java`:

```java
@Test
void payloadType_storedOnBuilder() {
    HumanTaskTarget target = HumanTaskTarget.inline()
        .title("Review")
        .payloadType(Map.class)
        .build();
    assertThat(target.payloadType()).isEqualTo(Map.class);
}

@Test
void resolutionType_storedOnBuilder() {
    HumanTaskTarget target = HumanTaskTarget.inline()
        .title("Review")
        .resolutionType(String.class)
        .build();
    assertThat(target.resolutionType()).isEqualTo(String.class);
}

@Test
void payloadType_nullByDefault() {
    HumanTaskTarget target = HumanTaskTarget.inline().title("Review").build();
    assertThat(target.payloadType()).isNull();
}

@Test
void resolutionType_nullByDefault() {
    HumanTaskTarget target = HumanTaskTarget.inline().title("Review").build();
    assertThat(target.resolutionType()).isNull();
}

@Test
void templateMode_acceptsPayloadAndResolutionTypes() {
    HumanTaskTarget target = HumanTaskTarget.template("tmpl-1")
        .payloadType(Map.class)
        .resolutionType(String.class)
        .build();
    assertThat(target.payloadType()).isEqualTo(Map.class);
    assertThat(target.resolutionType()).isEqualTo(String.class);
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl api -Dtest=HumanTaskTargetTest -DfailIfNoTests=false -q`
Expected: FAIL — `payloadType()` method does not exist.

- [ ] **Step 3: Add payloadType and resolutionType to HumanTaskTarget**

Add fields to `HumanTaskTarget`:
```java
private final Class<?> payloadType;
private final Class<?> resolutionType;
```

Add to constructor from builder:
```java
this.payloadType = builder.payloadType;
this.resolutionType = builder.resolutionType;
```

Add accessor methods:
```java
public Class<?> payloadType() { return payloadType; }
public Class<?> resolutionType() { return resolutionType; }
```

Add Builder fields and methods:
```java
private Class<?> payloadType;
private Class<?> resolutionType;

public Builder payloadType(Class<?> payloadType) {
    this.payloadType = payloadType;
    return this;
}

public Builder resolutionType(Class<?> resolutionType) {
    this.resolutionType = resolutionType;
    return this;
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn test -pl api -Dtest=HumanTaskTargetTest -DfailIfNoTests=false -q`
Expected: PASS

- [ ] **Step 5: Add payloadType/resolutionType to YAML schema**

In `schema/src/main/resources/schema/CaseDefinition.yaml`, add two
properties to the `HumanTask` definition (after `outcomes`):

```yaml
      payloadType:
        type: string
        description: >-
          Fully-qualified Java class name for typed payload validation via ContextBridge.
          Engine validates inputMapping output against this type at dispatch time.
      resolutionType:
        type: string
        description: >-
          Fully-qualified Java class name for typed resolution validation via ContextBridge.
          Engine validates WorkItem resolution against this type at completion time.
```

Regenerate schema classes:
Run: `mvn generate-sources -pl schema -q`

- [ ] **Step 6: Write failing test for YAML mapper parsing**

Add to `CaseDefinitionYamlMapperTest` (or create if it doesn't exist
alongside the mapper):

```java
@Test
void convertHumanTask_parsesPayloadAndResolutionTypes() {
    // Use a YAML snippet that includes payloadType and resolutionType
    String yaml = """
        name: test-case
        bindings:
          - name: review
            humanTask:
              title: "Review"
              payloadType: java.util.Map
              resolutionType: java.lang.String
            on:
              contextChange: ".ready"
        """;
    CaseDefinition def = CaseDefinitionYamlMapper.fromYaml(yaml);
    HumanTaskTarget ht = (HumanTaskTarget) def.getBindings().get(0).getTarget();
    assertThat(ht.payloadType()).isEqualTo(Map.class);
    assertThat(ht.resolutionType()).isEqualTo(String.class);
}

@Test
void convertHumanTask_nullTypesWhenNotSpecified() {
    String yaml = """
        name: test-case
        bindings:
          - name: review
            humanTask:
              title: "Review"
            on:
              contextChange: ".ready"
        """;
    CaseDefinition def = CaseDefinitionYamlMapper.fromYaml(yaml);
    HumanTaskTarget ht = (HumanTaskTarget) def.getBindings().get(0).getTarget();
    assertThat(ht.payloadType()).isNull();
    assertThat(ht.resolutionType()).isNull();
}

@Test
void convertHumanTask_invalidPayloadTypeThrows() {
    String yaml = """
        name: test-case
        bindings:
          - name: review
            humanTask:
              title: "Review"
              payloadType: com.nonexistent.FooBar
            on:
              contextChange: ".ready"
        """;
    assertThatThrownBy(() -> CaseDefinitionYamlMapper.fromYaml(yaml))
        .isInstanceOf(IllegalArgumentException.class)
        .hasMessageContaining("payloadType class not found");
}
```

- [ ] **Step 7: Run tests to verify they fail**

Run: `mvn test -pl api -Dtest=CaseDefinitionYamlMapperTest#convertHumanTask_parsesPayloadAndResolutionTypes -DfailIfNoTests=false -q`
Expected: FAIL — payloadType not parsed yet.

- [ ] **Step 8: Implement YAML mapper parsing**

In `CaseDefinitionYamlMapper.convertHumanTask()`, after the existing
field parsing (after `outcomes` handling), add:

```java
if (schema.getPayloadType() != null) {
    try {
        builder.payloadType(Class.forName(schema.getPayloadType()));
    } catch (ClassNotFoundException e) {
        throw new IllegalArgumentException(
            "humanTask payloadType class not found: " + schema.getPayloadType(), e);
    }
}
if (schema.getResolutionType() != null) {
    try {
        builder.resolutionType(Class.forName(schema.getResolutionType()));
    } catch (ClassNotFoundException e) {
        throw new IllegalArgumentException(
            "humanTask resolutionType class not found: " + schema.getResolutionType(), e);
    }
}
```

- [ ] **Step 9: Run all tests**

Run: `mvn test -pl api -q`
Expected: PASS — all existing tests + new tests pass.

- [ ] **Step 10: Commit**

```
feat(humanTask): add payloadType and resolutionType to HumanTaskTarget

HumanTaskTarget gains two nullable Class<?> fields for ContextBridge
validation. YAML schema and mapper parse payloadType/resolutionType
via Class.forName() with fail-fast on invalid class names.

Refs #689
```

---

### Task 2: BridgeResolver strict resolution + HumanTaskScheduleEvent + engine input path

**Files:**
- Modify: `common/src/main/java/io/casehub/engine/common/internal/context/BridgeResolver.java`
- Modify: `common/src/main/java/io/casehub/engine/common/internal/event/HumanTaskScheduleEvent.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandler.java`
- Test: `common/src/test/java/io/casehub/engine/common/internal/context/BridgeResolverTest.java`
- Test: `runtime/src/test/java/io/casehub/engine/HumanTaskTargetDispatchTest.java`

**Interfaces:**
- Consumes: `HumanTaskTarget.payloadType()`, `HumanTaskTarget.resolutionType()` (from Task 1)
- Produces: `BridgeResolver.resolveByTypeNameStrict(String)` → `ContextBridge<?>` (throws on unknown class),
  `HumanTaskScheduleEvent.payloadTypeName()` → `String` (nullable),
  `HumanTaskScheduleEvent.resolutionTypeName()` → `String` (nullable)

- [ ] **Step 1: Write failing test for BridgeResolver.resolveByTypeNameStrict**

Add to `BridgeResolverTest.java` in `common/src/test/`:

```java
@Test
void resolveByTypeNameStrict_throwsOnUnknownClass() {
    assertThatThrownBy(() -> resolver.resolveByTypeNameStrict("com.nonexistent.FooBar"))
        .isInstanceOf(IllegalArgumentException.class)
        .hasMessageContaining("com.nonexistent.FooBar");
}

@Test
void resolveByTypeNameStrict_returnsMapBridgeForMapClass() {
    assertThat(resolver.resolveByTypeNameStrict(Map.class.getName()))
        .isInstanceOf(MapBridge.class);
}

@Test
void resolveByTypeNameStrict_throwsOnNull() {
    assertThatThrownBy(() -> resolver.resolveByTypeNameStrict(null))
        .isInstanceOf(IllegalArgumentException.class);
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl common -Dtest=BridgeResolverTest#resolveByTypeNameStrict_throwsOnUnknownClass -DfailIfNoTests=false -q`
Expected: FAIL — method does not exist.

- [ ] **Step 3: Implement resolveByTypeNameStrict**

Add to `BridgeResolver`:

```java
public ContextBridge<?> resolveByTypeNameStrict(String typeName) {
    if (typeName == null) {
        throw new IllegalArgumentException("typeName must not be null for strict resolution");
    }
    try {
        return resolveByType(Class.forName(typeName));
    } catch (ClassNotFoundException e) {
        throw new IllegalArgumentException(
            "Bridge type class not found: " + typeName, e);
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn test -pl common -Dtest=BridgeResolverTest -DfailIfNoTests=false -q`
Expected: PASS

- [ ] **Step 5: Add payloadTypeName and resolutionTypeName to HumanTaskScheduleEvent**

The record currently has 12 fields. Add two new fields after `inputData`:

```java
public record HumanTaskScheduleEvent(
    UUID caseId,
    String tenancyId,
    String bindingName,
    HumanTaskTarget target,
    Map<String, Object> inputData,
    String payloadTypeName,        // new
    String resolutionTypeName,     // new
    Set<String> resolvedCandidateGroups,
    Set<String> resolvedCandidateUsers,
    Instant caseBudgetDeadline,
    Instant expiresAtDeadline,
    String resolvedTitle,
    String resolvedScope,
    java.time.Duration resolvedExpiresIn) {}
```

- [ ] **Step 6: Fix the publish call site in CaseContextChangedEventHandler**

In `publishHumanTaskSchedule()`, the `new HumanTaskScheduleEvent(...)`
constructor call gains two arguments after `inputData`:

```java
String payloadTypeName = target.payloadType() != null
    ? target.payloadType().getName() : null;
String resolutionTypeName = target.resolutionType() != null
    ? target.resolutionType().getName() : null;
```

Pass these as the 6th and 7th arguments in the constructor call.

- [ ] **Step 7: Add bridge validation before publish**

In `publishHumanTaskSchedule()`, after `evaluateInputMapping()` and
before the candidate set resolution, add validation:

```java
if (target.payloadType() != null && target.inputMapping() != null
        && !inputData.isEmpty()) {
    try {
        ContextBridge<?> bridge = bridgeResolver.resolveByType(target.payloadType());
        bridge.initialise(
            caseInstance.getCaseContext(),
            MAPPER.valueToTree(inputData));
    } catch (Exception e) {
        LOG.warnf(e,
            "Bridge validation failed for HumanTask binding '%s' caseId=%s — "
                + "inputMapping output does not match payloadType %s. PlanItem stays PENDING.",
            binding.getName(), caseInstance.getUuid(), target.payloadType().getName());
        return Uni.createFrom().voidItem();
    }
}
```

This requires injecting `BridgeResolver` into `CaseContextChangedEventHandler`.
The handler already injects it (used in `publishWorkerSchedule()`), so just
confirm it's available. Also needs the Jackson `MAPPER` — already present.

- [ ] **Step 8: Fix all compile errors from HumanTaskScheduleEvent change**

The constructor change breaks all existing call sites. Use
`ide_find_references` on `HumanTaskScheduleEvent` to find them:
- `CaseContextChangedEventHandler.publishHumanTaskSchedule()` — already fixed
- `HumanTaskTargetDispatchTest` recording handler — add `null, null` for the
  two new fields in the constructor

- [ ] **Step 9: Write integration test for typed payload dispatch**

Add to `HumanTaskTargetDispatchTest`:

```java
@Test
void typedPayload_validatesInputMappingOutput() {
    // Define a HumanTask with payloadType and verify the event carries the type name
    // Use a real POJO type that the inputMapping produces
}

@Test
void typedPayload_failFastOnMismatch() {
    // Define a HumanTask with a payloadType that doesn't match the inputMapping output
    // Verify the PlanItem stays PENDING (dispatch aborted)
}
```

Exact test code depends on existing test infrastructure patterns in
`HumanTaskTargetDispatchTest` — follow the existing `CaseHub` inner
class pattern.

- [ ] **Step 10: Run all tests**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest=HumanTaskTargetDispatchTest -DfailIfNoTests=false -q`
Expected: PASS

- [ ] **Step 11: Commit**

```
feat(humanTask): bridge validation on input path + strict BridgeResolver

BridgeResolver gains resolveByTypeNameStrict() that throws on unknown
class instead of silently falling back to MapBridge.
HumanTaskScheduleEvent carries payloadTypeName and resolutionTypeName.
Engine validates inputMapping output against payloadType via
bridge.initialise() at dispatch time — fail-fast on shape mismatch.

Refs #689
```

---

### Task 3: Work repo — data carrier changes

**Repo:** casehub/work (create branch `issue-689-workitem-typed-context` first)

**Files:**
- Modify: `api/src/main/java/io/casehub/work/api/WorkItemRef.java`
- Modify: `api/src/main/java/io/casehub/work/api/WorkItemCreateRequest.java`
- Modify: `api/src/main/java/io/casehub/work/api/WorkItemEvent.java`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/model/WorkItem.java`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/service/WorkItemService.java`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/spi/WorkItemSpiAdapter.java`
- Create: `runtime/src/main/resources/db/work/migration/V10__work_item_type_names.sql`
- Test: `api/src/test/java/io/casehub/work/api/WorkItemRefTest.java`
- Test: `api/src/test/java/io/casehub/work/api/WorkItemCreateRequestTest.java`
- Test: `runtime/src/test/java/io/casehub/work/runtime/spi/WorkItemSpiAdapterTest.java`

**Interfaces:**
- Produces: `WorkItemRef.payloadTypeName()` → `String` (nullable),
  `WorkItemRef.resolutionTypeName()` → `String` (nullable),
  `WorkItemCreateRequest.payloadTypeName` → `String` (nullable),
  `WorkItemCreateRequest.Builder.payloadTypeName(String)`,
  `WorkItemCreateRequest.Builder.resolutionTypeName(String)`,
  `WorkItemEvent.resolutionTypeName()` → `String` default method

- [ ] **Step 1: Create branch in work repo**

```bash
git -C /Users/mdproctor/claude/casehub/work checkout -b issue-689-workitem-typed-context
```

- [ ] **Step 2: Write failing test for WorkItemRef with type names**

Add to `WorkItemRefTest`:

```java
@Test
void record_carriesTypeNames() {
    WorkItemRef ref = new WorkItemRef(
        UUID.randomUUID(), WorkItemStatus.COMPLETED, "cr", "a1",
        "{}", "g1", "approved", "t1", "{}", "com.example.Payload", "com.example.Resolution");
    assertThat(ref.payloadTypeName()).isEqualTo("com.example.Payload");
    assertThat(ref.resolutionTypeName()).isEqualTo("com.example.Resolution");
}
```

- [ ] **Step 3: Add fields to WorkItemRef**

```java
public record WorkItemRef(
    UUID id,
    WorkItemStatus status,
    String callerRef,
    String assigneeId,
    String resolution,
    String candidateGroups,
    String outcome,
    String tenancyId,
    String payload,
    String payloadTypeName,
    String resolutionTypeName
) {}
```

- [ ] **Step 4: Fix all WorkItemRef constructor call sites**

Use `ide_find_references` on WorkItemRef constructor to find every call
site. Add `null, null` to each (backward compat — existing WorkItems have
no type names). Key sites:
- `WorkItemSpiAdapter.toRef()` — will be updated in Step 10
- Test constructors throughout the work repo
- Any other adapters or factories

- [ ] **Step 5: Add resolutionTypeName default method to WorkItemEvent**

```java
default String resolutionTypeName() { return ref().resolutionTypeName(); }
```

- [ ] **Step 6: Add fields to WorkItemCreateRequest**

Add fields:
```java
public final String payloadTypeName;
public final String resolutionTypeName;
```

Add to private constructor:
```java
this.payloadTypeName = b.payloadTypeName;
this.resolutionTypeName = b.resolutionTypeName;
```

Add Builder fields and methods:
```java
private String payloadTypeName;
private String resolutionTypeName;

public Builder payloadTypeName(final String v) { this.payloadTypeName = v; return this; }
public Builder resolutionTypeName(final String v) { this.resolutionTypeName = v; return this; }
```

Add to `equals()`, `hashCode()`, copy constructor.

- [ ] **Step 7: Add columns to WorkItem entity**

```java
@Column(name = "payload_type_name")
public String payloadTypeName;

@Column(name = "resolution_type_name")
public String resolutionTypeName;
```

- [ ] **Step 8: Create Flyway migration V10**

Create `runtime/src/main/resources/db/work/migration/V10__work_item_type_names.sql`:

```sql
ALTER TABLE work_item ADD COLUMN payload_type_name VARCHAR(512);
ALTER TABLE work_item ADD COLUMN resolution_type_name VARCHAR(512);
```

- [ ] **Step 9: Map fields in WorkItemService.create()**

After `item.scope = request.scope;` add:

```java
item.payloadTypeName = request.payloadTypeName;
item.resolutionTypeName = request.resolutionTypeName;
```

- [ ] **Step 10: Map fields in WorkItemSpiAdapter.toRef()**

```java
static WorkItemRef toRef(final WorkItem wi) {
    return new WorkItemRef(
            wi.id, wi.status, wi.callerRef, wi.assigneeId,
            wi.resolution, wi.candidateGroups, wi.outcome, wi.tenancyId,
            wi.payload, wi.payloadTypeName, wi.resolutionTypeName);
}
```

- [ ] **Step 11: Run tests**

Run: `mvn test -pl api -q && mvn test -pl runtime -q`
Expected: PASS (after fixing all constructor call sites)

- [ ] **Step 12: Commit**

```
feat(workitem): add payloadTypeName and resolutionTypeName to data carriers

WorkItemRef, WorkItemCreateRequest, and WorkItem entity gain typed
context metadata fields. Flyway V10 adds the columns. Fields are
opaque strings — the work repo stores and echoes them but does not
interpret them.

Refs casehubio/engine#689
```

---

### Task 4: Work repo adapter — HumanTaskScheduleHandler pass-through

**Repo:** casehub/work

**Files:**
- Modify: `engine-adapter/src/main/java/io/casehub/work/engine/HumanTaskScheduleHandler.java`
- Test: `engine-adapter/src/test/java/io/casehub/work/engine/HumanTaskScheduleHandlerTest.java`

**Interfaces:**
- Consumes: `HumanTaskScheduleEvent.payloadTypeName()`,
  `HumanTaskScheduleEvent.resolutionTypeName()` (from Task 2),
  `WorkItemCreateRequest.Builder.payloadTypeName(String)`,
  `WorkItemCreateRequest.Builder.resolutionTypeName(String)` (from Task 3)

- [ ] **Step 1: Write failing test**

Add to `HumanTaskScheduleHandlerTest`:

```java
@Test
void typeNames_passedThroughToWorkItemCreateRequest() {
    // Create a HumanTaskScheduleEvent with payloadTypeName and resolutionTypeName
    // Verify the WorkItemCreateRequest passed to WorkItemCreator has the type names
}
```

Follow existing test patterns — use the recording `WorkItemCreator` mock.

- [ ] **Step 2: Implement pass-through in handleInlineMode**

In `createInline()`, add to the `WorkItemCreateRequest.Builder` chain:

```java
.payloadTypeName(event.payloadTypeName())
.resolutionTypeName(event.resolutionTypeName())
```

Wait — `createInline()` doesn't receive the event directly. It receives
individual fields. Either thread the type names through as parameters, or
refactor to pass the event. The cleaner approach: add `payloadTypeName`
and `resolutionTypeName` parameters to `createInline()`.

In `handleInlineMode()`:
```java
createInline(
    event.target(),
    event.inputData(),
    event.resolvedCandidateGroups(),
    event.resolvedCandidateUsers(),
    callerRef,
    event.expiresAtDeadline(),
    event.caseBudgetDeadline(),
    event.payloadTypeName(),
    event.resolutionTypeName());
```

In `createInline()`, add the two new String params and set them on the
builder.

- [ ] **Step 3: Implement pass-through in handleTemplateMode**

In `handleTemplateMode()`, add to the `WorkItemCreateRequest.Builder`:

```java
.payloadTypeName(event.payloadTypeName())
.resolutionTypeName(event.resolutionTypeName())
```

- [ ] **Step 4: Run tests**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl engine-adapter -Dtest=HumanTaskScheduleHandlerTest -DfailIfNoTests=false -q`
Expected: PASS

- [ ] **Step 5: Commit**

```
feat(adapter): pass payloadTypeName/resolutionTypeName to WorkItem creation

HumanTaskScheduleHandler reads type names from the event and sets them
on WorkItemCreateRequest so they are stored on the WorkItem entity.

Refs casehubio/engine#689
```

---

### Task 5: Work repo adapter — PlanItemCompletionApplier bridge validation

**Repo:** casehub/work

**Files:**
- Modify: `engine-adapter/src/main/java/io/casehub/work/engine/PlanItemCompletionApplier.java`
- Test: `engine-adapter/src/test/java/io/casehub/work/engine/PlanItemCompletionApplierTest.java` (create if absent)

**Interfaces:**
- Consumes: `WorkItemRef.resolutionTypeName()` (from Task 3),
  `BridgeResolver.resolveByTypeNameStrict(String)` (from Task 2)

- [ ] **Step 1: Check if PlanItemCompletionApplier has an existing test**

Search for `PlanItemCompletionApplierTest` in the work repo. If absent,
create it. If present, extend it.

- [ ] **Step 2: Write failing test for resolution validation**

```java
@Test
void apply_validatesResolutionAgainstBridge_failFast() {
    // Set up a WorkItemRef with resolutionTypeName pointing to a POJO class
    // Set resolution JSON that does NOT match the POJO
    // Verify that apply() does NOT transition the PlanItem to COMPLETED
    // Verify that a workItemValidationFailed signal is written to context
}

@Test
void apply_skipsValidation_whenResolutionTypeNameIsNull() {
    // Set up a WorkItemRef with null resolutionTypeName
    // Verify that apply() transitions the PlanItem normally (today's behavior)
}

@Test
void apply_validResolution_proceedsNormally() {
    // Set up a WorkItemRef with resolutionTypeName and matching resolution JSON
    // Verify PlanItem transitions to COMPLETED and outputMapping runs
}
```

- [ ] **Step 3: Inject BridgeResolver into PlanItemCompletionApplier**

```java
@Inject BridgeResolver bridgeResolver;
```

- [ ] **Step 4: Add validation before status transition in apply()**

In `apply()`, after PlanItem and CaseInstance lookup, before
`applyStatus()`:

```java
if (ref != null && ref.resolutionTypeName() != null && ref.resolution() != null) {
    try {
        ContextBridge<?> bridge = bridgeResolver.resolveByTypeNameStrict(
            ref.resolutionTypeName());
        bridge.deserialise(MAPPER.readTree(ref.resolution()));
    } catch (Exception e) {
        LOG.warnf(e,
            "Resolution validation failed for PlanItem %s caseId=%s — "
                + "resolution does not match resolutionType %s",
            planItemId, caseId, ref.resolutionTypeName());
        writeValidationFailedSignal(instance, item, ref, e);
        return;
    }
}
```

- [ ] **Step 5: Implement writeValidationFailedSignal**

```java
private void writeValidationFailedSignal(
        CaseInstance instance, PlanItem item, WorkItemRef ref, Exception cause) {
    instance.getCaseContext().set("workItemValidationFailed",
        Map.of(
            "workItemId", ref.id().toString(),
            "bindingName", item.getBindingName(),
            "resolutionTypeName", ref.resolutionTypeName(),
            "error", cause.getMessage() != null ? cause.getMessage() : cause.getClass().getName()));
    eventBus.publish(
        EventBusAddresses.CONTEXT_CHANGED,
        new CaseContextChangedEvent(
            instance, instance.getCaseContext().snapshot(), ContextLayer.WORKING));
}
```

Add required imports for `ContextBridge`, `BridgeResolver`.

- [ ] **Step 6: Run tests**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl engine-adapter -q`
Expected: PASS

- [ ] **Step 7: Commit**

```
feat(adapter): bridge validation on WorkItem resolution before PlanItem completion

PlanItemCompletionApplier validates resolution against resolutionTypeName
via BridgeResolver.resolveByTypeNameStrict() + bridge.deserialise()
before transitioning the PlanItem. On validation failure, writes a
workItemValidationFailed signal to case context and returns without
completing the PlanItem.

Refs casehubio/engine#689
```

---

### Task 6: Integration test — typed round-trip

**Repo:** casehub/engine (back to engine repo)

**Files:**
- Create: `runtime/src/test/java/io/casehub/engine/HumanTaskTypedContextTest.java`

**Interfaces:**
- Consumes: All previous tasks

- [ ] **Step 1: Write the typed round-trip integration test**

Create a `@QuarkusTest` that:
1. Defines a `CaseHub` with a HumanTask binding declaring `payloadType`
   and `resolutionType`
2. Starts a case with context data matching the payload type
3. Verifies the `HumanTaskScheduleEvent` carries `payloadTypeName` and
   `resolutionTypeName`
4. Simulates WorkItem completion with valid resolution
5. Verifies the PlanItem completes and outputMapping runs

Also test the fail-fast path:
1. Define a HumanTask with `payloadType`
2. Set inputMapping to produce data that doesn't match the type
3. Verify the HumanTask is NOT scheduled (PlanItem stays PENDING)

Use the existing `HumanTaskTargetDispatchTest` as the pattern for the
`CaseHub` inner class, recording handler, and test structure.

- [ ] **Step 2: Run the integration test**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest=HumanTaskTypedContextTest -DfailIfNoTests=false -q`
Expected: PASS

- [ ] **Step 3: Run full test suite**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -q`
Expected: PASS — no regressions.

- [ ] **Step 4: Commit**

```
test: typed round-trip integration test for WorkItem boundary

Verifies payloadType/resolutionType validation at both dispatch and
completion, including fail-fast on shape mismatch and successful
outputMapping with valid typed resolution.

Refs #689
```

---

## Execution Notes

**Cross-repo coordination:**
- Tasks 1-2 and 6 are in casehub/engine
- Tasks 3-5 are in casehub/work
- Install engine SNAPSHOTs before running work repo tests:
  `mvn install -DskipTests -q` in the engine repo
- The work repo's engine-adapter depends on engine SNAPSHOTs

**Branch cleanup:**
- Engine: already on `issue-689-workitem-typed-context`
- Work: create branch in Task 3, Step 1
- Both branches must be closed together at work-end
