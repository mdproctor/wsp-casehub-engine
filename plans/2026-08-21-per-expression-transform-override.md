# Per-Expression Transform Override Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #943 — per-expression override for data transform projections
**Issue group:** #943

**Goal:** Route all data transform projections (`inputProjection`, `outputProjection`,
`inputProjectionOverride`) through `ExpressionEngineRegistry.transform()` instead of
hardcoded JQ, enabling per-expression language override via YAML `{lang: expr}` map syntax.

**Architecture:** Expand `CapabilityTarget` record to carry resolved `ExpressionEvaluator`
objects for input/output projections. Change `Binding.inputProjectionOverride` from `String`
to `ExpressionEvaluator`. Thread evaluators through `WorkerScheduleEvent` and runtime handlers.
Replace all `evalJqAsJsonNode`/`evalJqAsMap` calls with `registry.transform()`.

**Tech Stack:** Java 21, Quarkus 3.32.2, jackson-jq 1.6, JUnit 5 + AssertJ

## Global Constraints

- Foundation tier (`worker-api`) is NOT modified — `Capability.inputSchema`/`outputSchema` stay as Strings
- `ExpressionEvaluator` is from `io.casehub.platform.api.expression` (platform-api)
- `JQ_ONLY` fallback in `CaseDefinitionYamlMapper` must NOT reference `runtime` classes (api→runtime dependency violation)
- Error handling preserved: projection failures produce `LOG.warnf` + graceful fallback, never crash the case
- `resolveExpression()` already exists at `CaseDefinitionYamlMapper:1039` — reuse for all projection fields
- `WorkerExecutor.execute()` takes `String outputProjection` — must change to `ExpressionEvaluator`
- `DefaultWorkerExecutor.applyOutputSchema()` hardcodes JQ type — must use evaluator directly
- Planning module (`SubCaseExecutionHandler`, `SubCaseCompletionService`) has JQ-hardcoded call sites
- EventLog metadata for SubCase mappings must store evaluator type alongside expression
- `JqOnlyExpressionEngineRegistry` test helper needs `transform()` support alongside `JQ_ONLY`

## Revised Batch Structure (post-review R1-01 fix)

The original 3-batch plan creates uncompilable intermediate states (model type changes
in Batch 1, callers updated in Batch 3). Revised to 2 batches where each type change
is atomic with all its callers:

- **Batch 1:** All model type changes + YAML mapper + AgentConverter + ALL runtime callers.
  One atomic pass: type migration top-to-bottom. Tasks follow the original ordering but
  compile fixes cascade into the same commit.
- **Batch 2:** Cleanup — remove dead evalJq methods, verify clean build.

During execution, Tasks 1-7 are done as a single compilable unit. Each task's steps
are guidance, not independent compilation gates.

---

## Batch 1: Model types — CapabilityTarget, Binding, SubCaseMapping

### Task 1: Expand CapabilityTarget with projection evaluators

**Files:**
- Modify: `api/src/main/java/io/casehub/api/model/CapabilityTarget.java`
- Create: `api/src/test/java/io/casehub/api/model/CapabilityTargetTest.java`

**Interfaces:**
- Produces: `CapabilityTarget(Capability, ExpressionEvaluator inputProjection, ExpressionEvaluator outputProjection)` — 3-arg constructor
- Produces: `CapabilityTarget(Capability)` — 1-arg convenience that wraps Strings in `JQExpressionEvaluator`
- Produces: `inputProjection()`, `outputProjection()` — evaluator accessors

- [ ] **Step 1: Write failing tests**

  ```java
  @Test
  void oneArgConstructor_wrapsStringsInJqEvaluator() {
      var cap = Capability.of("cap", ".input", ".output");
      var ct = new CapabilityTarget(cap);
      assertThat(ct.inputProjection()).isInstanceOf(JQExpressionEvaluator.class);
      assertThat(ct.inputProjection().expression()).isEqualTo(".input");
      assertThat(ct.outputProjection()).isInstanceOf(JQExpressionEvaluator.class);
      assertThat(ct.outputProjection().expression()).isEqualTo(".output");
  }

  @Test
  void oneArgConstructor_nullInputSchema_nullEvaluator() {
      var cap = new Capability("cap", ".", null, null);
      var ct = new CapabilityTarget(cap);
      assertThat(ct.inputProjection()).isNotNull();
      assertThat(ct.outputProjection()).isNull();
  }

  @Test
  void threeArgConstructor_preservesEvaluators() {
      var cap = Capability.of("cap", ".input", ".output");
      var mvelInput = new TypedMvelExpressionEvaluator("user.name");
      var ct = new CapabilityTarget(cap, mvelInput, null);
      assertThat(ct.inputProjection()).isSameAs(mvelInput);
      assertThat(ct.outputProjection()).isNull();
  }
  ```

- [ ] **Step 2: Run tests — verify they fail** (CapabilityTarget only has 1-arg constructor)

  Run: `mvn test -pl api -Dtest=CapabilityTargetTest -DfailIfNoTests=false`

- [ ] **Step 3: Implement CapabilityTarget expansion**

  Use `ide_replace_member` to replace the record declaration. The record changes from
  `CapabilityTarget(Capability capability)` to the 3-arg form with a custom 1-arg constructor:

  ```java
  public record CapabilityTarget(
      Capability capability,
      ExpressionEvaluator inputProjection,
      ExpressionEvaluator outputProjection
  ) implements BindingTarget {

      public CapabilityTarget(Capability capability) {
          this(capability,
               capability.inputSchema() != null
                   ? new JQExpressionEvaluator(capability.inputSchema()) : null,
               capability.outputSchema() != null
                   ? new JQExpressionEvaluator(capability.outputSchema()) : null);
      }
  }
  ```

- [ ] **Step 4: Fix compilation — update all `new CapabilityTarget(cap)` call sites**

  Use `ide_find_references` on the CapabilityTarget constructor to find all call sites.
  Most use the 1-arg form (`new CapabilityTarget(capability)`) — these compile unchanged
  because the 1-arg convenience constructor still exists. Only call sites that directly
  construct with the canonical constructor need updating.

- [ ] **Step 5: Run tests — verify green**

  Run: `mvn test -pl api -Dtest=CapabilityTargetTest`

- [ ] **Step 6: Run full api module tests — verify no regressions**

  Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api`

- [ ] **Step 7: Commit**

  `feat(#943): expand CapabilityTarget with projection ExpressionEvaluator fields Refs #943`

---

### Task 2: Change Binding.inputProjectionOverride to ExpressionEvaluator

**Files:**
- Modify: `api/src/main/java/io/casehub/api/model/Binding.java`
- Modify: `api/src/test/java/io/casehub/api/model/BindingTest.java`

**Interfaces:**
- Consumes: `CapabilityTarget.inputProjection()` from Task 1
- Produces: `Binding.getInputProjectionOverride() → ExpressionEvaluator`
- Produces: `Binding.effectiveInputProjection(CapabilityTarget) → ExpressionEvaluator`
- Produces: `Builder.inputProjectionOverride(ExpressionEvaluator)` and `Builder.inputProjectionOverride(String)` (JQ convenience)

- [ ] **Step 1: Write failing tests**

  ```java
  @Test
  void effectiveInputProjection_returnsOverrideWhenPresent() {
      var cap = Capability.of("cap", ".full", ".");
      var ct = new CapabilityTarget(cap);
      var override = new JQExpressionEvaluator(".reduced");
      var b = Binding.builder().name("b").capability(cap)
          .on(new ContextChangeTrigger(null))
          .inputProjectionOverride(override).build();
      assertThat(b.effectiveInputProjection(ct)).isSameAs(override);
  }

  @Test
  void effectiveInputProjection_fallsBackToCapabilityTarget() {
      var cap = Capability.of("cap", ".full", ".");
      var ct = new CapabilityTarget(cap);
      var b = Binding.builder().name("b").capability(cap)
          .on(new ContextChangeTrigger(null)).build();
      assertThat(b.effectiveInputProjection(ct).expression()).isEqualTo(".full");
  }

  @Test
  void builderStringConvenience_wrapsInJqEvaluator() {
      var b = Binding.builder().name("b")
          .capability(Capability.of("cap", ".", "."))
          .on(new ContextChangeTrigger(null))
          .inputProjectionOverride(".narrow").build();
      assertThat(b.getInputProjectionOverride()).isInstanceOf(JQExpressionEvaluator.class);
      assertThat(((JQExpressionEvaluator) b.getInputProjectionOverride()).expression())
          .isEqualTo(".narrow");
  }
  ```

- [ ] **Step 2: Run tests — verify they fail**

  Run: `mvn test -pl api -Dtest=BindingTest -DfailIfNoTests=false`

- [ ] **Step 3: Implement Binding changes**

  Use `ide_edit_member` on `Binding`:
  - Change field `private String inputProjectionOverride` → `private ExpressionEvaluator inputProjectionOverride`
  - Change setter parameter type
  - Change getter return type
  - Change `effectiveInputProjection` to take `CapabilityTarget` and return `ExpressionEvaluator`:
    ```java
    public ExpressionEvaluator effectiveInputProjection(CapabilityTarget capTarget) {
        return inputProjectionOverride != null ? inputProjectionOverride : capTarget.inputProjection();
    }
    ```
  - Add `Builder.inputProjectionOverride(ExpressionEvaluator)` overload
  - Keep `Builder.inputProjectionOverride(String)` wrapping in `JQExpressionEvaluator`

- [ ] **Step 4: Fix compilation across api module**

  Use `ide_diagnostics` to find all compilation errors from the type change. Fix each:
  - `CaseDefinitionYamlMapper`: `builder.inputProjectionOverride(schemaBinding.getInputProjectionOverride())`
    → pass through `resolveExpression()` (done in Task 4)
  - For now, temporarily wrap String calls in `new JQExpressionEvaluator()` to compile

- [ ] **Step 5: Run tests — verify green**

  Run: `mvn test -pl api -Dtest=BindingTest`

- [ ] **Step 6: Commit**

  `feat(#943): change Binding.inputProjectionOverride to ExpressionEvaluator Refs #943`

---

### Task 3: Change SubCaseMapping.Expression to carry ExpressionEvaluator

**Files:**
- Modify: `api/src/main/java/io/casehub/api/model/SubCaseMapping.java`
- Modify: `api/src/test/java/io/casehub/api/model/SubCaseMappingTest.java` (create if absent)

**Interfaces:**
- Produces: `Expression(ExpressionEvaluator evaluator)` — record component
- Produces: `SubCaseMapping.of(String)` wraps in `JQExpressionEvaluator`

- [ ] **Step 1: Write failing tests**

  ```java
  @Test
  void ofString_wrapsInJqEvaluator() {
      SubCaseMapping mapping = SubCaseMapping.of(".child");
      assertThat(mapping).isInstanceOf(SubCaseMapping.Expression.class);
      var expr = (SubCaseMapping.Expression) mapping;
      assertThat(expr.evaluator()).isInstanceOf(JQExpressionEvaluator.class);
  }

  @Test
  void expression_preservesEvaluator() {
      var mvel = new TypedMvelExpressionEvaluator("child");
      var expr = new SubCaseMapping.Expression(mvel);
      assertThat(expr.evaluator()).isSameAs(mvel);
  }
  ```

- [ ] **Step 2: Run tests — verify they fail**

- [ ] **Step 3: Implement SubCaseMapping changes**

  Use `ide_replace_member` to change the `Expression` record:
  ```java
  record Expression(ExpressionEvaluator evaluator) implements SubCaseMapping {
      public Expression {
          Objects.requireNonNull(evaluator);
      }
  }
  ```

  Update `SubCaseMapping.of(String)`:
  ```java
  static SubCaseMapping of(String expression) {
      return new Expression(new JQExpressionEvaluator(expression));
  }
  ```

- [ ] **Step 4: Fix compilation — `expr.expression()` → `expr.evaluator()` accessor renames**

  Use `ide_find_references` on the old `expression()` accessor to find all pattern match sites.
  Each `case SubCaseMapping.Expression expr -> ... expr.expression()` becomes
  `case SubCaseMapping.Expression expr -> ... expr.evaluator()`.

  Key sites:
  - `CaseContextChangedEventHandler.publishSubCaseSchedule()`: pattern match + evalJqAsMap call
  - `SubCaseCompletionService.applyOutputMapping()`: reads expression from EventLog metadata,
    wraps in `SubCaseMapping.of(expr)` (still works — `of(String)` wraps in JQExpressionEvaluator)

- [ ] **Step 5: Run tests — verify green**

- [ ] **Step 6: Commit**

  `feat(#943): change SubCaseMapping.Expression to carry ExpressionEvaluator Refs #943`

---

## Batch 2: Parse-time wiring — YAML mapper and AgentConverter

### Task 4: Wire projection fields through resolveExpression in CaseDefinitionYamlMapper

**Files:**
- Modify: `api/src/main/java/io/casehub/api/model/converter/CaseDefinitionYamlMapper.java`
- Modify: `api/src/test/java/io/casehub/api/model/converter/CaseDefinitionYamlMapperExpressionOverrideTest.java`

**Interfaces:**
- Consumes: `CapabilityTarget(cap, inputEval, outputEval)` from Task 1
- Consumes: `Builder.inputProjectionOverride(ExpressionEvaluator)` from Task 2
- Consumes: `SubCaseMapping.Expression(ExpressionEvaluator)` from Task 3
- Produces: Correctly resolved projection evaluators from YAML `{lang: expr}` map syntax

- [ ] **Step 1: Write failing tests for projection override parsing**

  ```java
  @Test
  void inputProjectionOverride_plainString_createsJqEvaluator() {
      JsonNode node = new TextNode(".reduced");
      ExpressionEvaluator result = CaseDefinitionYamlMapper.resolveExpression(node, registry, "jq");
      assertThat(result).isInstanceOf(JQExpressionEvaluator.class);
      assertThat(result.expression()).isEqualTo(".reduced");
  }

  @Test
  void inputProjectionOverride_mapSyntax_createsOverrideEvaluator() {
      ObjectNode node = MAPPER.createObjectNode();
      node.put("mvel", "transaction.amount");
      ExpressionEvaluator result = CaseDefinitionYamlMapper.resolveExpression(node, registry, "jq");
      assertThat(result.type()).isEqualTo("mvel");
  }
  ```

  (These tests already pass because `resolveExpression` is generic — verify they pass to confirm
  the utility works for projections too.)

- [ ] **Step 2: Write failing YAML round-trip test for capability projection override**

  Create a YAML definition with `inputProjection: { mvel: "user.name" }` on a capability.
  Load via `CaseDefinitionYamlMapper.load()`. Assert the `CapabilityTarget.inputProjection()`
  is a `TypedMvelExpressionEvaluator`.

- [ ] **Step 3: Update capability parsing in convertToApiModel**

  In `convertToApiModel`, where capabilities are built, read the raw YAML node for
  `inputProjection` and `outputProjection` and resolve via `resolveExpression()`:
  ```java
  ExpressionEvaluator inputEval = resolveExpression(
      rawCapNode.get("inputProjection"), effectiveRegistry, expressionLang);
  ExpressionEvaluator outputEval = resolveExpression(
      rawCapNode.get("outputProjection"), effectiveRegistry, expressionLang);
  ```
  Pass these to `CapabilityTarget(cap, inputEval, outputEval)` when building bindings.

- [ ] **Step 4: Update binding parsing in convertBinding**

  Read `rawBindingNode.get("inputProjectionOverride")` as raw JsonNode, resolve via
  `resolveExpression()`, pass to `builder.inputProjectionOverride(ExpressionEvaluator)`.

- [ ] **Step 5: Update SubCase parsing**

  Read `inputMapping` and `outputMapping` as raw JsonNode, resolve via `resolveExpression()`,
  wrap in `SubCaseMapping.Expression(evaluator)`.

- [ ] **Step 6: Update JQ_ONLY fallback with transform() support**

  Add `transform()` to the `JQ_ONLY` registry using `JQEvaluator` from `common` (NOT
  `JQExpressionEngine` from `runtime`):
  ```java
  @Override
  public List<JsonNode> transform(ExpressionEvaluator evaluator, JsonNode input) {
      if (!JQExpressionEvaluator.TYPE.equals(evaluator.type())) {
          throw new IllegalArgumentException("Only 'jq' without CDI. Got: " + evaluator.type());
      }
      JQExpressionEvaluator jqEval = (JQExpressionEvaluator) evaluator;
      ValidationResult vr = new JQEvaluator().eval(jqEval.expression(), input);
      return vr.ok() && vr.output() != null ? vr.output() : List.of(input);
  }
  ```

- [ ] **Step 7: Run tests — verify green**

  Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api`

- [ ] **Step 8: Commit**

  `feat(#943): wire projection fields through resolveExpression in YAML mapper Refs #943`

---

### Task 5: Update AgentConverter to resolve projections through registry

**Files:**
- Modify: `api/src/main/java/io/casehub/api/model/converter/AgentConverter.java`
- Modify: `api/src/test/java/io/casehub/api/model/converter/AgentConverterTest.java` (create if absent)

**Interfaces:**
- Consumes: `resolveExpression()` from CaseDefinitionYamlMapper
- Produces: `AgentConverter.toApiAgent(Agent, JsonNode, ExpressionEngineRegistry, String)` — new signature

- [ ] **Step 1: Write failing test**

  ```java
  @Test
  void toApiAgent_resolvesProjectionViaRegistry() {
      // Build schema Agent with inputProjection
      Agent schemaAgent = new Agent();
      schemaAgent.setSystemPrompt("test");
      schemaAgent.setInputProjection(".narrow");
      schemaAgent.setModel(testModel());
      // raw node with plain string
      ObjectNode rawNode = MAPPER.createObjectNode();
      rawNode.put("inputProjection", ".narrow");
      // Call new overload
      io.casehub.api.model.ai.Agent result = AgentConverter.toApiAgent(
          schemaAgent, rawNode, JQ_ONLY_REGISTRY, "jq");
      assertThat(result).isNotNull();
  }
  ```

- [ ] **Step 2: Run test — verify it fails** (method signature doesn't exist yet)

- [ ] **Step 3: Implement new AgentConverter.toApiAgent overload**

  Add 4-arg overload. Inside, resolve `inputProjection` and `outputProjection` from the raw
  node via `resolveExpression()`. If an evaluator is found, wrap in a `UnaryOperator<JsonNode>`
  lambda and pass to `builder.inputTransformer()`:

  ```java
  ExpressionEvaluator inputEval = CaseDefinitionYamlMapper.resolveExpression(
      rawNode != null ? rawNode.get("inputProjection") : null, registry, expressionLang);
  if (inputEval != null) {
      builder.inputTransformer(input -> {
          List<JsonNode> result = registry.transform(inputEval, input);
          return result.isEmpty() ? input : result.get(0);
      });
  } else if (schemaAgent.getInputProjection() != null) {
      builder.inputProjection(schemaAgent.getInputProjection());
  }
  ```

  Same pattern for `outputProjection`. Keep the 1-arg overload delegating to the 4-arg with
  `JQ_ONLY` registry and `"jq"` default.

- [ ] **Step 4: Update CaseDefinitionYamlMapper call site**

  Where `AgentConverter.toApiAgent(schemaAgent)` is called, pass the registry and raw agent
  node: `AgentConverter.toApiAgent(schemaAgent, rawAgentNode, registry, expressionLang)`.

- [ ] **Step 5: Run tests — verify green**

  Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api`

- [ ] **Step 6: Commit**

  `feat(#943): resolve agent projections through expression registry Refs #943`

---

## Batch 3: Runtime call sites — thread evaluators, replace evalJq

### Task 6: Update WorkerScheduleEvent and handler input projection

**Files:**
- Modify: `common/src/main/java/io/casehub/engine/common/internal/event/WorkerScheduleEvent.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/WorkerScheduleEventHandler.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandler.java`

**Interfaces:**
- Consumes: `Binding.effectiveInputProjection(CapabilityTarget) → ExpressionEvaluator` from Task 2
- Consumes: `CapabilityTarget.inputProjection()` from Task 1
- Produces: `WorkerScheduleEvent.inputProjectionOverride() → ExpressionEvaluator`
- Produces: `WorkerScheduleEvent.effectiveInputProjection() → ExpressionEvaluator`

- [ ] **Step 1: Change WorkerScheduleEvent field type**

  Use `ide_edit_member` to change `inputProjectionOverride` from `String` to `ExpressionEvaluator`.
  Update `effectiveInputProjection()` to return `ExpressionEvaluator`:
  ```java
  public ExpressionEvaluator effectiveInputProjection() {
      return inputProjectionOverride != null ? inputProjectionOverride
          : (capability.inputSchema() != null
              ? new JQExpressionEvaluator(capability.inputSchema()) : null);
  }
  ```
  Update all constructor overloads: parameter type String → ExpressionEvaluator, defaulting to null.

- [ ] **Step 2: Fix CaseContextChangedEventHandler.publishWorkerSchedule()**

  Where `WorkerScheduleEvent` is constructed, pass `binding.effectiveInputProjection(capTarget)`
  (ExpressionEvaluator) instead of `binding.effectiveInputProjection(capability)` (String).
  This requires having `CapabilityTarget` in scope — extract it from the binding's target.

- [ ] **Step 3: Replace evalJqAsJsonNode in WorkerScheduleEventHandler**

  Replace:
  ```java
  JsonNode narrowedInput = evalJqAsJsonNode(workingLayer, event.effectiveInputProjection());
  ```
  With:
  ```java
  JsonNode narrowedInput = transformSingle(event.effectiveInputProjection(), workingLayer);
  ```
  Add the `transformSingle` helper method (uses `registry.transform()`).

- [ ] **Step 4: Replace evalJqAsMap in CaseContextChangedEventHandler.tryProvision()**

  The inline effective-projection computation + `evalJqAsMap()` call. Replace with
  `transformAsMap(binding.effectiveInputProjection(capTarget), ctx)`.

- [ ] **Step 5: Fix HumanTask inputMapping instanceof check**

  Replace:
  ```java
  if (target.inputMapping() instanceof JQExpressionEvaluator jq) {
      evalJqAsMap(ctx, jq.expression());
  }
  ```
  With:
  ```java
  transformAsMap(target.inputMapping(), ctx);
  ```
  No more `instanceof` — the registry dispatches by evaluator type.

- [ ] **Step 6: Replace SubCase evalJqAsMap calls**

  For SubCase input mapping: replace `evalJqAsMap(ctx, expr.expression())` with
  `transformAsMap(expr.evaluator(), ctx)`.

- [ ] **Step 7: Run module tests — verify green**

  Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime`

- [ ] **Step 8: Commit**

  `feat(#943): route input projections through ExpressionEngineRegistry.transform() Refs #943`

---

### Task 7: Thread output projection evaluators through runtime

**Files:**
- Modify: `runtime/src/main/java/io/casehub/engine/internal/executor/PersistentWorkerFunctionHandler.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/worker/scope/DefaultPersistentScope.java`
- Modify: `scheduler-quartz/src/main/java/io/casehub/engine/scheduler/quartz/QuartzWorkerExecutionJob.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/orchestration/DefaultWorkOrchestrator.java`

**Interfaces:**
- Consumes: `CapabilityTarget.outputProjection()` from Task 1
- Consumes: `CapabilityTarget.inputProjection()` from Task 1 (for DefaultWorkOrchestrator)

- [ ] **Step 1: Update PersistentWorkerFunctionHandler**

  Where it reads `ct.capability().outputSchema()` as String, change to
  `ct.outputProjection()` as ExpressionEvaluator. Thread both input and output evaluators
  into `DefaultPersistentScope`.

- [ ] **Step 2: Update DefaultPersistentScope**

  Change `String inputProjection` and `String outputSchema` fields to `ExpressionEvaluator`.
  In `nextEvent()`, replace `jqEvaluator.eval(inputProjection, ...)` with
  `registry.transform(inputProjection, ...)`.
  In `emit()`, replace `jqEvaluator.eval(outputSchema, ...)` with
  `registry.transform(outputProjection, ...)`.

- [ ] **Step 3: Update DefaultWorkOrchestrator**

  In `doSubmitWork()`, look up the binding from `CaseDefinition.getBindings()` by capability
  name to get the `CapabilityTarget`. Replace `evalJqAsMap(ctx, capability.inputSchema())`
  with `transformAsMap(capTarget.inputProjection(), ctx)`. Fallback when no binding matches:
  wrap `capability.inputSchema()` in `JQExpressionEvaluator`.

- [ ] **Step 4: Update QuartzWorkerExecutionJob output projection**

  Where `capability.outputSchema()` is passed as String, resolve from `CapabilityTarget`
  stored in EventLog metadata (store evaluator type alongside expression, same pattern as
  `contextBridgeType`).

- [ ] **Step 5: Run full test suite**

  Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime,scheduler-quartz`

- [ ] **Step 6: Commit**

  `feat(#943): thread output projection evaluators through runtime Refs #943`

---

### Task 8: Remove dead evalJq methods and verify clean build

**Files:**
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/WorkerScheduleEventHandler.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandler.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/orchestration/DefaultWorkOrchestrator.java`

**Interfaces:**
- None — cleanup only

- [ ] **Step 1: Remove evalJqAsJsonNode and evalJqAsMap methods**

  Use `ide_find_references` on each method to confirm zero remaining callers, then
  `ide_refactor_safe_delete`.

- [ ] **Step 2: Remove unused JQEvaluator injections**

  If any handler injected `JQEvaluator` solely for projection evaluation and no longer uses
  it, remove the injection field.

- [ ] **Step 3: Run full build**

  Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test`

- [ ] **Step 4: Commit**

  `refactor(#943): remove dead evalJq projection methods Closes #943`

---

## References

- [2026-08-21-per-expression-transform-override-design.md] — design spec this plan implements
- [ExpressionEngine.java:154] — transform() SPI method
- [ExpressionEngineRegistry.java:110] — transform() registry dispatch
- [CaseDefinitionYamlMapper.java:1039] — resolveExpression() utility
- [CapabilityTarget.java:21] — current 1-field record
- [Binding.java:39] — inputProjectionOverride String field
- [WorkerScheduleEvent.java:38] — inputProjectionOverride String field
- [AgentConverter.java:45] — static toApiAgent method
- [GE-20260609-3c86d1] — ADR-0009 superseded by #925
- GitHub #943 — focal issue
- GitHub #925 — per-expression override for boolean conditions (predecessor)
