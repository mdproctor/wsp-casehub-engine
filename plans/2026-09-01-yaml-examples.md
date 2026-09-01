# YAML Examples — Three-Pathway Rosetta Stone Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> executing-plans to implement this plan task-by-task. Each task
> follows TDD (test-driven-development) and uses ide-tooling for
> structural editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #984 — standalone YAML examples for all execution models
**Issue group:** #984

**Goal:** Create 8 "Rosetta Stone" example sets — each exists as YAML + Java DSL + Java Annotations, proving three-pathway parity.

**Architecture:** Each example is a triple: one YAML file in `examples/yaml/`, one `CaseHub` subclass module in `examples/<model>-dsl/`, and one `@Case` interface module in `examples/<model>-annotated/`. Two existing annotated examples (`simple-case-annotated`, `goap-case-annotated`) are renamed for consistency and get YAML + DSL companions. Six new examples are written from scratch in all three forms.

**Tech Stack:** YAML (CaseDefinition schema), Java 21, Quarkus (Gizmo codegen for annotations), Maven multi-module.

## Global Constraints

- YAML files validate against `schema/src/main/resources/schema/CaseDefinition.yaml`
- YAML files parse without error through `CaseDefinitionYamlMapper.load()`
- Java DSL modules depend on `casehub-engine-api` + `casehub-engine` (runtime)
- Java annotation modules depend on `casehub-engine-annotations` + `casehub-engine-annotations-deployment`
- Every example module has `<maven.deploy.skip>true</maven.deploy.skip>`
- File naming: YAML `<model>-<domain>.yaml`, dirs `<model>-dsl/`, `<model>-annotated/`
- Each YAML file has a header comment documenting: scenario, pathway note, key features

---

## Batch 1: Choreography + Sequential

### Task 1: Choreography triple (rename existing + write YAML + DSL)

**Files:**
- Rename: `examples/simple-case-annotated/` → `examples/choreography-annotated/` (use `ide_move_file`)
- Create: `examples/yaml/choreography-onboarding.yaml`
- Create: `examples/choreography-dsl/pom.xml`
- Create: `examples/choreography-dsl/src/main/java/io/casehub/examples/ChoreographyOnboardingCase.java`
- Test: `examples/choreography-dsl/src/test/java/io/casehub/examples/ChoreographyOnboardingCaseTest.java`
- Modify: `pom.xml` (root) — update module path, add new modules

**Interfaces:**
- Produces: `examples/yaml/choreography-onboarding.yaml` — the reference YAML for this execution model
- Produces: `ChoreographyOnboardingCase extends CaseHub` — DSL equivalent

- [ ] **Step 1: Rename simple-case-annotated → choreography-annotated**

Use `ide_move_file` to rename the directory. Update the root `pom.xml` module reference and the module's own `pom.xml` artifact ID/name.

- [ ] **Step 2: Write the YAML example**

Create `examples/yaml/choreography-onboarding.yaml` — banking customer onboarding scenario. Must demonstrate: multiple `contextChange` bindings, `cloudEvent` trigger, `cron` schedule, goals, milestones, completion block. Align the domain (banking onboarding) and workers with the existing `SimpleAnnotatedCase.java` so the three forms express the same case.

Key features to demonstrate:
- `contextChange` bindings with filter expressions
- `when:` guard conditions on bindings
- `schedule: { cron: "0 0 * * * ?" }` trigger
- `goals:` with `condition:` and `kind:`
- `milestones:` with `condition:` and `entryCriteria:`
- `completion:` block with `success:` goal kind

Workers use `do:` blocks (SWF HTTP calls) since this is pure YAML — no Java worker functions.

- [ ] **Step 3: Validate YAML against schema**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl schema -Dtest=SchemaValidationTest -f /Users/mdproctor/claude/casehub/engine/pom.xml`

The test scans `src/main/resources/examples/` — the new file is in `examples/yaml/` so copy it to the schema examples dir temporarily for validation, or update the test to also scan `../../examples/yaml/`.

- [ ] **Step 4: Write the DSL module**

Create `examples/choreography-dsl/` Maven module following the pattern from `simple-case-annotated/pom.xml` but depending on `casehub-engine-api` + `casehub-engine` (runtime) instead of the annotations module. The `CaseHub` subclass uses `CaseDefinition.builder()` to express the same case definition as the YAML.

- [ ] **Step 5: Write DSL test**

Test that loads the definition and asserts key properties: namespace, name, capability count, worker count, binding count, goal count.

- [ ] **Step 6: Add modules to root pom.xml**

Add `examples/choreography-annotated`, `examples/choreography-dsl` to the reactor. Remove the old `examples/simple-case-annotated` module entry.

- [ ] **Step 7: Build and verify**

Run: `mvn install -DskipTests -q -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Then: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl examples/choreography-dsl,examples/choreography-annotated -f /Users/mdproctor/claude/casehub/engine/pom.xml`

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add examples/ pom.xml
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(#984): choreography example — YAML + DSL + annotations (renamed)

Three-pathway Rosetta Stone: banking customer onboarding.
Renamed simple-case-annotated → choreography-annotated.

Refs #984"
```

### Task 2: Sequential triple

**Files:**
- Create: `examples/yaml/sequential-onboarding.yaml`
- Create: `examples/sequential-dsl/pom.xml`
- Create: `examples/sequential-dsl/src/main/java/io/casehub/examples/SequentialOnboardingCase.java`
- Create: `examples/sequential-dsl/src/test/java/io/casehub/examples/SequentialOnboardingCaseTest.java`
- Create: `examples/sequential-annotated/pom.xml`
- Create: `examples/sequential-annotated/src/main/java/io/casehub/examples/SequentialOnboardingAnnotated.java`
- Create: `examples/sequential-annotated/src/test/java/io/casehub/examples/SequentialOnboardingAnnotatedTest.java`
- Modify: `pom.xml` (root) — add new modules

**Interfaces:**
- Produces: YAML, DSL, and annotated versions of a sequential HR onboarding scenario

- [ ] **Step 1: Write the YAML example**

Create `examples/yaml/sequential-onboarding.yaml` — HR employee onboarding. Must demonstrate:
- `planningStrategy: sequential`
- `sequence:` on workers (ordered chain: collect-docs → background-check → provision-access → schedule-orientation)
- Workers with `do:` blocks for HTTP dispatch
- Goals and milestones tracking onboarding progress

- [ ] **Step 2: Write the DSL module**

`SequentialOnboardingCase extends CaseHub`. Uses `builder.planningStrategy("sequential")` and `Worker.builder().sequence(List.of("step1", "step2"))`.

- [ ] **Step 3: Write the annotated module**

`@Case` interface. Sequential planning requires `@Customize`:
```java
@Customize
static void customize(CaseDefinition.Builder builder) {
    builder.planningStrategy("sequential");
}
```
Workers use `sequence:` which is not expressible in annotations — requires `@Customize` to wire.

- [ ] **Step 4: Add modules, build, test, commit**

Same pattern as Task 1 Steps 6-8.

## Batch 2: GOAP + LLM Decomposition

### Task 3: GOAP triple (rename existing + extend YAML + write DSL)

**Files:**
- Rename: `examples/goap-case-annotated/` → `examples/goap-annotated/`
- Create: `examples/yaml/goap-contract-review.yaml` (extend from `schema/examples/goap-yaml-example.yaml`)
- Create: `examples/goap-dsl/pom.xml`
- Create: `examples/goap-dsl/src/main/java/io/casehub/examples/GoapContractReviewCase.java`
- Create: `examples/goap-dsl/src/test/java/io/casehub/examples/GoapContractReviewCaseTest.java`
- Modify: `pom.xml` (root)

Key YAML features: `decompositionStrategy: goap`, spec-level `goapActions:` with `preconditions`, `effects`, `cost`, `benefit`, `softPreconditions`. Align with the legal contract review domain from `goap-case-annotated`.

- [ ] **Steps 1-6: Same pattern as Task 1** — rename, write YAML, validate, write DSL, test, commit.

### Task 4: LLM decomposition triple

**Files:**
- Create: `examples/yaml/llm-research-analysis.yaml`
- Create: `examples/llm-decomposition-dsl/` (full module)
- Create: `examples/llm-decomposition-annotated/` (full module)
- Modify: `pom.xml` (root)

Key YAML features: `decompositionStrategy: llm`, goals with rich `description:` fields (LLM reads these), `agent:` block on workers with model config. Annotations need `@Customize` for `decompositionStrategy`.

- [ ] **Steps 1-6: Same pattern as Task 2.**

## Batch 3: HumanTask + SubCase

### Task 5: HumanTask triple

**Files:**
- Create: `examples/yaml/humantask-loan-approval.yaml`
- Create: `examples/humantask-dsl/` (full module)
- Create: `examples/humantask-annotated/` (full module)
- Modify: `pom.xml` (root)

Key YAML features: `humanTask:` binding target with `title:`, `candidateGroups:`, `candidateUsers:`, `outcomes:`, `expiresIn:`, `payloadType:`, `resolutionType:`, SLA milestones. Finance loan approval domain. Annotations need `@Customize` — `@Bind` can't target humanTask natively.

- [ ] **Steps 1-6: Same pattern as Task 2.**

### Task 6: SubCase triple

**Files:**
- Create: `examples/yaml/subcase-insurance-claims.yaml`
- Create: `examples/subcase-dsl/` (full module)
- Create: `examples/subcase-annotated/` (full module)
- Modify: `pom.xml` (root)

Key YAML features: `subCase:` binding target with `namespace:`, `name:`, `version:`, `inputMapping:`, `outputMapping:`, `maxRecursionDepth:`, grouped sub-cases (`groupId:`, `totalInGroup:`, `requiredCount:`). Insurance claims domain. Annotations need `@Customize` for full SubCase configuration.

- [ ] **Steps 1-6: Same pattern as Task 2.**

## Batch 4: A2A + MCP + Validation

### Task 7: A2A triple

**Files:**
- Create: `examples/yaml/a2a-market-research.yaml`
- Create: `examples/a2a-dsl/` (full module)
- Create: `examples/a2a-annotated/` (full module)
- Modify: `pom.xml` (root)

Key YAML features: `a2a:` block on workers with `endpoint:`, `skill:`, `streaming: true`, `auth: { type: bearer, tokenConfigKey: ... }`. Market research domain with multiple remote A2A agents. Annotations need `@Customize` — A2A is YAML/DSL only.

- [ ] **Steps 1-6: Same pattern as Task 2.**

### Task 8: MCP triple

**Files:**
- Create: `examples/yaml/mcp-code-analysis.yaml`
- Create: `examples/mcp-dsl/` (full module)
- Create: `examples/mcp-annotated/` (full module)
- Modify: `pom.xml` (root)

Key YAML features: `mcp:` block on workers with stdio transport (`command: ["/path/to/server"]`) and HTTP transport (`url: https://...`, `auth:`). Code analysis domain with file-tools (stdio) and remote-analyzer (HTTP). Annotations need `@Customize` — MCP is YAML/DSL only.

- [ ] **Steps 1-6: Same pattern as Task 2.**

### Task 9: Schema validation update + README

**Files:**
- Modify: `schema/src/test/java/io/casehub/model/SchemaValidationTest.java`
- Create: `examples/yaml/README.md`
- Create: `examples/README.md` (update or create, listing all examples with pathway indicators)

- [ ] **Step 1: Update SchemaValidationTest**

Add a second `@MethodSource` that scans `../../examples/yaml/` for YAML files, validating each against the schema. Keep the existing `exampleFiles()` method for `schema/examples/`.

```java
static Stream<Path> yamlExampleFiles() throws IOException {
    Path yamlDir = Path.of("../../examples/yaml");
    if (!Files.exists(yamlDir)) return Stream.empty();
    return Files.list(yamlDir)
        .filter(p -> p.toString().endsWith(".yaml"));
}

@ParameterizedTest
@MethodSource("yamlExampleFiles")
void yamlExample_validatesAgainstSchema(Path yamlFile) throws IOException {
    // same validation logic as exampleYaml_validatesAgainstSchema
}
```

- [ ] **Step 2: Write examples/yaml/README.md**

Document the Rosetta Stone concept: what each example demonstrates, which pathways exist, how to find the Java equivalents.

- [ ] **Step 3: Run full validation**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl schema -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: all YAML examples validate against schema.

Run: `mvn install -DskipTests -q -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: all example modules compile.

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add schema/ examples/
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(#984): schema validation for examples/yaml/ + README

Closes #984"
```

## References

- `specs/issue-984-yaml-examples/2026-09-01-yaml-examples-design.md` — design spec
- `specs/issue-984-yaml-examples/decisions.md` — D1-D4 decisions
- `examples/simple-case-annotated/src/main/java/.../SimpleAnnotatedCase.java` — annotation pattern reference
- `examples/simple-case-annotated/pom.xml` — Maven module structure reference
- `schema/src/test/java/io/casehub/model/SchemaValidationTest.java` — validation test to extend
- `schema/src/main/resources/schema/CaseDefinition.yaml` — YAML schema
- `schema/src/main/resources/examples/document-processing.yaml` — choreography YAML reference
- `schema/src/main/resources/examples/goap-yaml-example.yaml` — GOAP YAML reference
- engine#978 — parent epic (Part 2 example table)
- engine#986 — Three Pathways Guide (will reference these examples)
