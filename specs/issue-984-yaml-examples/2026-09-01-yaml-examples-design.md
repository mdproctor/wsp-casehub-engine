# Standalone YAML Examples — Three-Pathway Rosetta Stone

Refs: engine#984 (parent: engine#978)

## Context

CaseHub has three authoring pathways — YAML, Java DSL (`CaseHub` subclasses), and Java Annotations (`@Case` interfaces) — but no place where the same case definition exists in all three forms. The existing 9 annotated Java examples cover choreography and GOAP but miss sequential, LLM decomposition, HumanTask, SubCase, A2A, and MCP. There are zero standalone YAML examples in `examples/`.

This spec creates 8 "Rosetta Stone" example sets — each exists in all three forms so users can see exactly how the same semantics map across pathways, and where annotations need `@Customize` escape hatches.

## Scope

Core 8 execution models for this branch. The remaining 15 from the #978 table are follow-on work.

| # | Execution Model | Domain | Key YAML Features |
|---|---|---|---|
| 1 | Choreography | Banking — customer onboarding | Multiple `contextChange` bindings, goals, milestones, completion |
| 2 | Sequential | HR — employee onboarding | `planningStrategy: sequential`, `sequence:` on workers |
| 3 | GOAP | Legal — contract review | `decompositionStrategy: goap`, `goapActions:` with preconditions/effects/cost |
| 4 | LLM decomposition | Research — analysis pipeline | `decompositionStrategy: llm`, goals with descriptions, agent worker |
| 5 | HumanTask | Finance — loan approval | `humanTask:` target with candidates, outcomes, SLA, `payloadType` |
| 6 | SubCase | Insurance — claims processing | `subCase:` with input/output mapping, `maxRecursionDepth`, grouped sub-cases |
| 7 | A2A | Intelligence — market research | `a2a:` block with endpoint, skill, streaming, auth |
| 8 | MCP | DevOps — code analysis | `mcp:` block with command (stdio) and url (HTTP) transports |

## Directory Structure

```
examples/
  yaml/                                ← NEW: all YAML examples
    choreography-onboarding.yaml
    sequential-onboarding.yaml
    goap-contract-review.yaml
    llm-research-analysis.yaml
    humantask-loan-approval.yaml
    subcase-insurance-claims.yaml
    a2a-market-research.yaml
    mcp-code-analysis.yaml

  choreography-annotated/              ← RENAMED from simple-case-annotated
  choreography-dsl/                    ← NEW
  sequential-annotated/                ← NEW
  sequential-dsl/                      ← NEW
  goap-annotated/                      ← RENAMED from goap-case-annotated
  goap-dsl/                            ← NEW
  llm-decomposition-annotated/         ← NEW
  llm-decomposition-dsl/              ← NEW
  humantask-annotated/                 ← NEW
  humantask-dsl/                       ← NEW
  subcase-annotated/                   ← NEW
  subcase-dsl/                         ← NEW
  a2a-annotated/                       ← NEW
  a2a-dsl/                             ← NEW
  mcp-annotated/                       ← NEW
  mcp-dsl/                             ← NEW

  # Existing examples — untouched
  warehouse-annotated/
  aircraft-maintenance-annotated/
  incident-response-annotated/
  search-rescue-annotated/
  wildfire-response-annotated/
  composable-routing/
  typed-context/
```

## Alignment with Existing Examples

| Execution Model | Existing Match | Action |
|---|---|---|
| Choreography | `simple-case-annotated` (banking onboarding) | Rename dir → `choreography-annotated`. Write YAML + DSL pairs. |
| GOAP | `goap-case-annotated` (contract review) | Rename dir → `goap-annotated`. Extend existing schema YAML → `examples/yaml/`. Write DSL pair. |
| Sequential | *none* | Write all three from scratch |
| LLM decomposition | *none* | Write all three |
| HumanTask | *none* (partial in aircraft-maintenance) | Write all three |
| SubCase | *none* | Write all three |
| A2A | *none* | Write all three |
| MCP | *none* | Write all three |

## Three-Pathway Mechanics

### YAML
Self-contained `.yaml` files. Workers use external dispatch (`do:` blocks with HTTP calls, `a2a:` endpoints, `mcp:` servers) or no-function workers (HumanTask, SubCase). No Java code.

### Java DSL
`CaseHub` subclasses using fluent builders. Each is a standalone Maven module with `src/main/java/` and a test that loads the definition.

```java
public class ChoreographyOnboardingCase extends CaseHub {
  @Override
  protected CaseDefinition define() {
    return CaseDefinition.builder()
        .namespace("banking").name("customer-onboarding").version("1.0.0")
        .capability(Capability.of("verifyIdentity", ...))
        .worker(Worker.builder().name("id-verifier").capabilityName("verifyIdentity")...)
        .binding(Binding.builder().capability("verifyIdentity").on(contextChange(".application != null"))...)
        .goal(Goal.builder().name("accountOpened").when(".account != null")...)
        .build();
  }
}
```

### Java Annotations
`@Case` interfaces. Where annotations can't express a feature natively, `@Customize` drops into the DSL — the escape hatch is visible and documented.

```java
@Case(namespace = "research", name = "analysis", version = "1.0.0")
public interface LlmResearchCase {

  @Worker(capability = "analyze")
  @Bind(contextChange = ".topic != null")
  default AnalysisResult analyze(String topic) { ... }

  @Customize
  static void customize(CaseDefinition.Builder builder) {
    // Annotations can't express decompositionStrategy — DSL fills the gap
    builder.decompositionStrategy("llm");
  }
}
```

The `@Customize` blocks are the documentation — they show exactly where annotations reach their limit.

## Per-Example YAML Structure

Each YAML file follows a consistent structure:

```yaml
##
## Example: <Execution Model> — <Domain>
##
## Scenario: <1-2 sentence description>
##
## Pathway: YAML (pure — no Java required)
## See also: examples/<model>-annotated/ (annotations)
##           examples/<model>-dsl/ (DSL)
##
## Key features demonstrated:
##   - <feature 1>
##   - <feature 2>
##

dsl: "1.0.0"
namespace: casehub-examples
name: <name>
version: "1.0.0"
types:
  - <type-path>
labels:
  - example/reference

spec:
  ## ─── Capabilities ───
  ## ─── Workers ────────
  ## ─── Bindings ───────
  ## ─── Milestones ─────
  ## ─── Goals & Completion ───
```

Comments annotate each section explaining the YAML surface being demonstrated.

## Relocated Files

- `schema/examples/document-processing.yaml` → `examples/yaml/choreography-onboarding.yaml` (refactored to match banking onboarding domain from `simple-case-annotated`)
- `schema/examples/goap-yaml-example.yaml` → `examples/yaml/goap-contract-review.yaml` (extended with full spec-level `goapActions:`)
- Remaining schema examples stay as schema test fixtures

## Validation

- `SchemaValidationTest` updated to scan `examples/yaml/` in addition to `schema/examples/`
- Each Java example module has a test that loads the definition and asserts key properties (capabilities count, worker count, binding count)
- Each YAML file must parse without error through `CaseDefinitionYamlMapper.load()`

## Annotation Escape Hatch Inventory

Features that require `@Customize` in the annotation pathway:

| Feature | Annotation Gap | `@Customize` Fills |
|---|---|---|
| `decompositionStrategy` | No `@Case` attribute | `builder.decompositionStrategy("llm")` |
| `planningStrategy` | No `@Case` attribute | `builder.planningStrategy("sequential")` |
| `humanTask:` target | `@Bind` targets capabilities only | `builder.binding(Binding.builder()...humanTask(...)...)` |
| `subCase:` target | Not expressible | Full binding with `SubCaseTarget` |
| `a2a:` worker block | Not expressible | Worker with `A2AWorkerFunction` |
| `mcp:` worker block | Not expressible | Worker with MCP function |
| `goapActions:` at spec level | Inferred from `@Worker` annotations | `builder.goapAction(GoapAction.of(...))` |

## Implementation Phases

### Phase 1 — YAML examples (8 files)
Write all 8 YAML files in `examples/yaml/`. Validate against schema. This is the forcing function — if a YAML feature doesn't parse, fix the mapper.

### Phase 2 — DSL examples (8 modules)
Write `CaseHub` subclass versions. Each is a Maven module under `examples/<model>-dsl/`.

### Phase 3 — Annotation alignment (8 modules)
Rename `simple-case-annotated` → `choreography-annotated`, `goap-case-annotated` → `goap-annotated`. Write 6 new annotation modules. Document `@Customize` escape hatches.

### Phase 4 — Validation + Schema test update
Update `SchemaValidationTest` to scan `examples/yaml/`. Add per-module tests for Java examples.

## References

- engine#978 — parent epic (Part 2 table: 23 examples)
- engine#986 — Three Pathways Guide (will reference these examples)
- `schema/src/main/resources/examples/` — existing schema YAML examples (6 files)
- `examples/` — existing annotated Java examples (9 directories)
- `examples/simple-case-annotated/src/main/java/.../SimpleAnnotatedCase.java` — choreography annotation pattern
- `examples/goap-case-annotated/src/main/java/.../GoalOrientedContractReview.java` — GOAP annotation pattern
- `annotations/runtime/src/main/java/io/casehub/engine/annotations/` — annotation definitions
- `api/src/main/java/io/casehub/api/model/converter/CaseDefinitionYamlMapper.java` — YAML loading pipeline
