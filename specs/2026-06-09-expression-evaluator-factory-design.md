# Expression Evaluator Factory — Design Spec
**Issue:** engine#289 (previously anticipated as engine#280 in the JQ evaluator consolidation spec)
**Date:** 2026-06-09

---

## Problem

`CaseDefinitionYamlMapper` hardcodes `new JQExpressionEvaluator(expression)` at five call sites
when parsing YAML case definitions. The runtime evaluation path is already fully pluggable —
`ExpressionEngine` SPI, `Instance<ExpressionEngine>` CDI discovery, `ExpressionEngineRegistry`
dispatch — but the parse-time instantiation path is not. A consumer who deploys a custom
expression language has a runtime engine to evaluate expressions but no way to produce the right
`ExpressionEvaluator` value object from a YAML string.

Compounding this: `CaseDefinitionYamlMapper` holds two static mutable fields (`yamlMapper`,
and the factory added by the first spec) injected via `StartupEvent` observers. Both observers
fire at CDI default priority 2500. `DefaultCaseDefinitionRegistry.onStart()` is annotated
`@Priority(10)` — lower number = higher priority = fires first. This means both static injections
arrive **after** all case definitions have been loaded. The workarounds are not just inelegant;
they are **broken** for their stated purpose. This PR fixes both simultaneously.

---

## What Is Not Changing

The runtime evaluation path — `ExpressionEngine`, `ExpressionEngineRegistry`,
`JQExpressionEngine`, `LambdaExpressionEngine`, `DefaultExpressionEngineRegistry`,
`StageLifecycleEvaluator` — is correct and stays as-is. This spec is about parse-time
instantiation and the mapper's dependency injection, nothing else.

---

## Design Decisions

### Decision 1 — Extend ExpressionEngine, not a new factory interface

The critique's recommended direction: add `create(String expression)` to `ExpressionEngine`.

The invariant load-bearing for the entire pluggability story is:

> **`expressionLang` in YAML == `ExpressionEvaluator.type()` == `ExpressionEngine.type()`**

With a separate `ExpressionEvaluatorFactory` interface, this invariant is split across three
independent objects that must agree. Violation is silent: a factory that produces an evaluator
of the wrong type routes to a different engine at evaluation time with no error at definition
registration.

Extending `ExpressionEngine` collapses the invariant to one object. The same CDI bean declares
the language (`type()`), creates value objects from strings (`create()`), evaluates them
(`evaluate()`), and validates them (`validate()`). Mismatch is structurally impossible.

This is also consistent with the existing SPI pattern: `ExpressionEngine.type()` already IS the
language identifier, and `DefaultExpressionEngineRegistry` already discovers all engines via
`Instance<ExpressionEngine>`. Adding `create()` gives the registry a natural dispatch path with
no new CDI patterns, no new `@DefaultBean`/`@Alternative` mechanics, and no new interfaces.

**`LambdaExpressionEvaluator` is Java-DSL-only** — it cannot be created from a string. The
`LambdaExpressionEngine.create()` uses the default method (throws
`UnsupportedOperationException`). This is correct: `expressionLang: lambda` in YAML is
meaningless and should fail at definition registration. `StageLifecycleEvaluator` is unaffected;
it only calls `evaluate()`.

### Decision 2 — Pass registry as a parameter; delete both static workarounds

`CaseHub` already uses `@Inject CaseHubRuntime runtime` — CDI injection in the api module is
established. `YamlCaseHub` can add `@Inject ExpressionEngineRegistry expressionEngineRegistry`
and `@Inject @YamlMapper ObjectMapper objectMapper` directly. CDI injects these fields when the
subclass bean is instantiated, before any `StartupEvent` fires. No static state, no priority
race, no workarounds.

`CaseDefinitionYamlMapper.load(InputStream, ObjectMapper, ExpressionEngineRegistry)` takes both
dependencies as parameters. A no-arg overload `load(InputStream)` exists for tests and
non-CDI contexts (uses a plain `new ObjectMapper(new YAMLFactory())` and a JQ-only inline
implementation). This overload is explicitly for non-CDI use; it cannot support custom
expression languages.

`ObjectMapperInjector` is deleted. `setObjectMapper()` is deleted. Thread safety is no longer
a concern: CDI-injected fields are written before application code runs; the `volatile` field
`definition` in `YamlCaseHub` handles lazy init correctly.

### Decision 3 — Per-definition expressionLang is correct for v1

SW 1.0 declares `expressionLang` at the workflow level. For CaseHub:

The `ExpressionEvaluator.type()` field already encodes the language on every value object — the
runtime dispatch is per-evaluator. Per-expression granularity at the model level already exists.
What #289 adds is per-definition configuration at the **YAML level**: the mapper uses one
language to create all evaluators in a definition.

Per-expression YAML syntax (e.g., `{lang: drools, expr: "..."}` per binding condition) is a
valid future enhancement but requires a separate schema design. v1 per-definition granularity is:
- Consistent with SW 1.0
- Simpler authoring (one language declared once at the top)
- Coherent validation: the registry can check at registration time that an engine exists for
  the declared language, before evaluating any individual expression

Java DSL consumers can already mix evaluators freely (`new JQExpressionEvaluator()` alongside
`new LambdaExpressionEvaluator()` in the same definition). Per-expression mixing is a YAML
syntax concern only.

---

## Changes

### 1. Schema — add `expressionLang`

Add to `schema/src/main/resources/schema/CaseDefinition.yaml` at the top level:

```yaml
expressionLang:
  type: string
  title: ExpressionLang
  description: >
    Expression language used for all condition expressions in this definition.
    Must match the expressionLang() of a registered ExpressionEngine CDI bean.
    Defaults to "jq". Must satisfy: expressionLang == ExpressionEvaluator.type()
    == ExpressionEngine.type() for the engine that handles it.
  default: "jq"
```

Regenerate schema model: `io.casehub.model.CaseDefinition` gains `getExpressionLang()`.
Propagate to `io.casehub.api.model.CaseDefinition`: add `String expressionLang` field (default
`"jq"`) with getter/setter; set in `convertToApiModel` from schema model
(`nullSafe default → "jq"`).

### 2. `ExpressionEngine` — add `create()` default method

```java
/**
 * Creates an ExpressionEvaluator for this engine's expression language from a raw
 * expression string. Called by ExpressionEngineRegistry.create() during YAML definition
 * loading; never called for lambda-type evaluators.
 *
 * <p>The returned evaluator's type() MUST equal this engine's type().
 *
 * @throws UnsupportedOperationException if this engine does not support string-based creation
 *     (e.g. LambdaExpressionEngine — lambdas are Java-DSL-only and cannot be created from YAML)
 */
default ExpressionEvaluator create(String expression) {
    throw new UnsupportedOperationException(
        "ExpressionEngine '" + type() + "' does not support creation from string expressions. "
        + "Use the Java DSL to construct evaluators of this type.");
}
```

### 3. `ExpressionEngineRegistry` — add `create()` dispatch

```java
/**
 * Creates an ExpressionEvaluator for the given expression language.
 *
 * <p>Dispatches to the ExpressionEngine whose type() equals expressionLang. The returned
 * evaluator's type() is guaranteed to equal expressionLang (enforced by ExpressionEngine
 * contract).
 *
 * @throws IllegalArgumentException if no engine is registered for expressionLang
 * @throws UnsupportedOperationException if the matching engine does not support string creation
 */
ExpressionEvaluator create(String expression, String expressionLang);
```

### 4. `DefaultExpressionEngineRegistry` — implement `create()`

```java
@Override
public ExpressionEvaluator create(final String expression, final String expressionLang) {
    for (ExpressionEngine engine : engines) {
        if (engine.type().equals(expressionLang)) {
            return engine.create(expression);
        }
    }
    throw new IllegalArgumentException(
        "No ExpressionEngine registered for expressionLang '" + expressionLang + "'");
}
```

### 5. `JQExpressionEngine` — implement `create()`

```java
@Override
public ExpressionEvaluator create(final String expression) {
    return new JQExpressionEvaluator(expression);
}
```

### 6. `CaseDefinitionYamlMapper` — remove static state; add registry parameter

Remove:
- `private static ObjectMapper yamlMapper` field
- `public static void setObjectMapper(ObjectMapper)` method
- (factory static field and setter — never added, but explicitly not added here)

Add:
```java
public static CaseDefinition load(InputStream stream,
                                   ObjectMapper objectMapper,
                                   ExpressionEngineRegistry registry) throws IOException { ... }

/** Convenience overload for non-CDI contexts (tests, tooling). JQ only; no custom languages. */
public static CaseDefinition load(InputStream stream) throws IOException {
    return load(stream,
        new ObjectMapper(new YAMLFactory()),
        (expr, lang) -> switch (lang) {
            case "jq" -> new JQExpressionEvaluator(expr);
            default -> throw new IllegalArgumentException(
                "No CDI registry available. Only 'jq' is supported without injection. Got: " + lang);
        });
}
```

The `convertToApiModel` method receives `expressionLang` (read from schema model, null-safe
default `"jq"`) and passes it to the registry at all five call sites:

| Location | Expression source |
|----------|------------------|
| `convertBinding` | `schemaBinding.getWhen()` |
| `convertTrigger` | `schemaTrigger.getContextChange().getFilter()` |
| Milestones loop | `sm.getCondition()`, `sm.getEntryCriteria()` |
| Goals loop | `sg.getCondition()` |

Each replaces `new JQExpressionEvaluator(expr)` with `registry.create(expr, expressionLang)`.

### 7. `YamlCaseHub` — inject dependencies directly

```java
public class YamlCaseHub extends CaseHub {

    @Inject ExpressionEngineRegistry expressionEngineRegistry;
    @Inject @YamlMapper ObjectMapper objectMapper;

    private final String path;
    private volatile CaseDefinition definition;

    public YamlCaseHub(String path) {
        this.path = path;
    }

    @Override
    public CaseDefinition getDefinition() {
        if (definition == null) {
            synchronized (this) {
                if (definition == null) {
                    try (InputStream is = loadStream()) {
                        definition = (expressionEngineRegistry != null && objectMapper != null)
                            ? CaseDefinitionYamlMapper.load(is, objectMapper, expressionEngineRegistry)
                            : CaseDefinitionYamlMapper.load(is);
                    } catch (IOException e) {
                        throw new RuntimeException("Failed to load CaseHub definition from " + path, e);
                    }
                }
            }
        }
        return definition;
    }
}
```

CDI fields are null when `YamlCaseHub` is instantiated outside a CDI container (pure Java tests).
The null check falls back to `load(InputStream)` which supports JQ only.
The `volatile definition` field provides correct lazy initialisation without further barriers.

### 8. `ObjectMapperInjector` — delete

No longer needed. `YamlCaseHub` injects the `@YamlMapper ObjectMapper` directly.
`CaseDefinitionYamlMapper.setObjectMapper()` is deleted. The priority race is eliminated at root.

### 9. `DefaultCaseDefinitionRegistry` — validate expressionLang at registration

Add a validation step in `validateExpressions()` that confirms an engine exists for the declared
`expressionLang` **before** attempting to validate individual expressions:

```java
private void validateExpressionLang(CaseDefinition definition) {
    String lang = definition.getExpressionLang(); // defaults "jq"
    try {
        // probe: create a trivial expression to verify the engine exists and supports create()
        expressionEngineRegistry.create("true", lang);
    } catch (IllegalArgumentException e) {
        throw new IllegalArgumentException(
            "Case definition '" + definition.getName() + "' declares expressionLang '" + lang
            + "' but no ExpressionEngine is registered for this language.", e);
    } catch (UnsupportedOperationException e) {
        throw new IllegalArgumentException(
            "Case definition '" + definition.getName() + "' declares expressionLang '" + lang
            + "' but the matching ExpressionEngine does not support YAML expression creation.", e);
    }
}
```

This enforces the three-way invariant at registration time: if `expressionLang` has no matching
engine that supports `create()`, the definition is rejected with a clear error.

---

## Invariant

> **`expressionLang` in YAML == `ExpressionEvaluator.type()` returned by the engine ==
> `ExpressionEngine.type()` of the engine that evaluates it**

Enforced structurally: the registry dispatches `create()` to the engine whose `type()` equals
`expressionLang`. The engine's `create()` returns an evaluator; by contract, the returned
evaluator's `type()` must equal the engine's `type()`. The same engine handles evaluation.
One object, one type, one language. Violation requires deliberate misimplementation of the
engine's `create()` contract — which the javadoc makes explicit.

---

## ADR

An ADR is needed for the `expressionLang` YAML field:
- **Decision**: adopt `expressionLang` at definition level, following CNCF SW 1.0
- **Rationale**: CNCF alignment for interoperability; per-definition is the natural boundary
  for a deployment unit; per-expression YAML syntax deferred to a future schema design
- **Consequences**: definitions with `expressionLang` other than `"jq"` require a matching
  `ExpressionEngine` CDI bean at deployment time; fail-fast at registration

File as `docs/adr/` entry alongside this implementation.

---

## Testing

No static state means no ordering dependencies and no `@AfterEach` resets.

**`CaseDefinitionYamlMapperTest`** — existing tests call `load(is)` (JQ default). All pass
unchanged. Add:
- `expressionLang` field round-trips from YAML → `CaseDefinition.expressionLang`
- Custom language: pass a stub `ExpressionEngineRegistry` (inline lambda) to `load(is, mapper, registry)`; verify all five sites call the stub
- Missing engine: stub registry throws `IllegalArgumentException`; verify load propagates it

**`DefaultExpressionEngineRegistryTest`** — add:
- `create("expr", "jq")` → returns `JQExpressionEvaluator`
- `create("expr", "unknown")` → throws `IllegalArgumentException`
- `create("expr", "lambda")` → throws `UnsupportedOperationException` (LambdaExpressionEngine default)

**`JQExpressionEngineTest`** — add:
- `create("expr")` returns `JQExpressionEvaluator("expr")`; verify `type() == "jq"`

**`CaseDefinitionRegistryValidationTest`** — add:
- Definition with unknown `expressionLang` rejected at registration with clear error message
