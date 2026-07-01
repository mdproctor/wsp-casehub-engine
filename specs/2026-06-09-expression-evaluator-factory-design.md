# Expression Evaluator Factory — Design Spec
**Issue:** engine#289 (previously anticipated as engine#280 in the JQ evaluator consolidation spec)
**Date:** 2026-06-09

---

## Problem

`CaseDefinitionYamlMapper` hardcodes `new JQExpressionEvaluator(expression)` at five call
sites when parsing YAML case definitions. The runtime evaluation path is already fully pluggable
— `ExpressionEngine` SPI, `Instance<ExpressionEngine>` CDI discovery, `ExpressionEngineRegistry`
dispatch — but the parse-time instantiation path is not. A consumer who deploys a custom
expression language has a runtime engine to evaluate expressions but no way to produce the right
`ExpressionEvaluator` value object from a YAML string.

Compounding this: `CaseDefinitionYamlMapper` uses a static `ObjectMapper` field set by
`ObjectMapperInjector @Observes StartupEvent`. `DefaultCaseDefinitionRegistry.onStart()` is
annotated `@Priority(10)` — lower CDI priority number means earlier execution. `ObjectMapperInjector`
has no `@Priority`, which defaults to 2500. The definitions are loaded at priority 10; the
ObjectMapper is injected at priority 2500. **The workaround fires after the work it was meant
to front-run.** This PR fixes both problems simultaneously.

---

## What Is Not Changing

The runtime evaluation path — `ExpressionEngine`, `JQExpressionEngine`, `LambdaExpressionEngine`,
`DefaultExpressionEngineRegistry`, `StageLifecycleEvaluator` — is correct and is not touched.
This spec is about parse-time instantiation and dependency injection into the mapper.

---

## Module Restructuring (prerequisite)

### Move `ExpressionEngineRegistry` from `common/spi/` → `api/engine/`

Currently: `io.casehub.engine.common.spi.ExpressionEngineRegistry`
New location: `io.casehub.api.engine.ExpressionEngineRegistry`

This move is valid: all dependencies of `ExpressionEngineRegistry`
(`ExpressionEvaluator`, `CaseContext`, `JsonNode`) are already in `api`.
`ExpressionEngineRegistry` belongs alongside `ExpressionEngine` in `api/engine/` — they are
complementary parts of the same SPI: one declares what handles a language, the other dispatches
across all registered handlers.

**Blast radius:** 15 references, all in `runtime` or `scheduler-quartz`. All already transitively
depend on `api` via `common`. Change is import-path only for `runtime`; `scheduler-quartz`
requires one `pom.xml` change (see below). Affected files:
- `DefaultExpressionEngineRegistry` (implements clause + import)
- `DefaultCaseDefinitionRegistry`, `MilestoneLifecycleManager`, `CaseContextChangedEventHandler`
  (inject + import)
- `ConditionalScheduledTriggerJob` in `scheduler-quartz` (inject + import)
- `ExpressionEngineRegistryTest`, `CaseContextChangedEventHandlerRoutingTest` (test import)

**`scheduler-quartz/pom.xml`** — add explicit dependency:

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-engine-api</artifactId>
    <version>${project.version}</version>
</dependency>
```

`scheduler-quartz` currently declares only `casehub-engine-common` as a direct dependency.
`casehub-engine-api` is available transitively (common → api), so the build compiles today —
but after moving `ExpressionEngineRegistry` to `api/engine/`, `ConditionalScheduledTriggerJob`
would directly import from a module it does not declare. Maven permits this but it is dependency
hygiene debt. Making the dependency explicit is the correct fix.

### Move `@YamlMapper` qualifier from `runtime/internal/marshaller/` → `api/marshaller/` (new package)

Currently: `io.casehub.engine.internal.marshaller.YamlMapper`
New location: `io.casehub.api.marshaller.YamlMapper`

`@YamlMapper` is a marshaller qualifier — it identifies an `ObjectMapper` configured for YAML
with placeholder resolution. Placing it in `api/engine/` alongside `ExpressionEngine` and
`CaseHubRuntime` would mix marshalling infrastructure into the engine SPI package. A new
`api/marshaller/` package is the architecturally correct home: it describes purpose, not
convenience. `api/engine/` has CDI qualifier precedent in `common/qualifier/`, but those are
engine-operation qualifiers (`@CrossTenant`, `@EngineSystem`) — `@YamlMapper` is not.

`@YamlMapper` is a pure annotation with no deps beyond `jakarta.inject.Qualifier` (already in
`api/pom.xml`). Moving it creates no new module dependency. The producer
`CaseHubObjectMapperProducer` stays in `runtime`.

**Blast radius:** 7 references, all import updates:
- `CaseHubObjectMapperProducer` (qualifier on `@Produces` method, import update)
- `ObjectMapperInjector` (deleted — see below)
- 4 test classes in `runtime` (import update)

---

## Design

### Invariant

> **`expressionLang` declared in YAML == `ExpressionEvaluator.type()` produced by the engine ==
> `ExpressionEngine.type()` of the engine that evaluates it**

This three-way identity is load-bearing. With the approach below it is structurally enforced:
the same CDI bean whose `type()` equals `expressionLang` also supplies `create()`. The returned
evaluator's type is asserted at the registry dispatch site. Violation by a buggy implementation
is caught immediately on first use.

---

### 1. `ExpressionEngine` — add two default methods

```java
/**
 * Creates an ExpressionEvaluator from a raw expression string.
 * Only engines that override this method can be used in YAML definitions
 * (expressionLang: <type>). Lambda-type evaluators are Java-DSL-only
 * and intentionally do not override this method.
 */
default ExpressionEvaluator create(String expression) {
    throw new UnsupportedOperationException(
        "ExpressionEngine '" + type() + "' does not support creation from string expressions. "
        + "Use the Java DSL to construct evaluators of this type.");
}

/**
 * Returns true if this engine supports create(String).
 * Used by assertLanguageSupported() to distinguish "no engine" from
 * "engine exists but is Java-DSL-only".
 */
default boolean supportsStringCreation() {
    return false;
}
```

`LambdaExpressionEngine` uses both defaults — correct, lambda predicates cannot be created
from strings. `JQExpressionEngine` overrides both to return `new JQExpressionEvaluator(expression)`
and `true` respectively.

---

### 2. `ExpressionEngineRegistry` — add `create()` and `assertLanguageSupported()`

```java
/**
 * Creates an ExpressionEvaluator for the given expression language by dispatching to
 * the ExpressionEngine whose type() equals expressionLang. The returned evaluator's
 * type() is asserted to equal expressionLang at the call site.
 *
 * @throws IllegalArgumentException if no engine is registered for expressionLang
 * @throws UnsupportedOperationException if the matching engine does not override create()
 */
ExpressionEvaluator create(String expression, String expressionLang);

/**
 * Asserts that a registered ExpressionEngine exists for expressionLang and that it
 * supports creation from string expressions. Does NOT call create() — no domain
 * objects are constructed as a side effect.
 *
 * @throws IllegalArgumentException if no engine is registered for expressionLang
 * @throws UnsupportedOperationException if the engine exists but is Java-DSL-only
 */
void assertLanguageSupported(String expressionLang);
```

---

### 3. `DefaultExpressionEngineRegistry` — implement both methods

```java
@Override
public ExpressionEvaluator create(final String expression, final String expressionLang) {
    for (ExpressionEngine engine : engines) {
        if (engine.type().equals(expressionLang)) {
            ExpressionEvaluator evaluator = engine.create(expression);
            // Enforce the invariant: evaluator.type() must equal expressionLang
            if (!evaluator.type().equals(expressionLang)) {
                throw new IllegalStateException(
                    "ExpressionEngine '" + engine.type() + "'.create() returned evaluator of type '"
                    + evaluator.type() + "' — must equal '" + expressionLang + "'");
            }
            return evaluator;
        }
    }
    throw new IllegalArgumentException(
        "No ExpressionEngine registered for expressionLang '" + expressionLang + "'");
}

@Override
public void assertLanguageSupported(final String expressionLang) {
    for (ExpressionEngine engine : engines) {
        if (engine.type().equals(expressionLang)) {
            if (!engine.supportsStringCreation()) {
                throw new UnsupportedOperationException(
                    "expressionLang '" + expressionLang + "' is a Java-DSL-only type and cannot "
                    + "be used in YAML definitions. Use expressionLang: jq or register a custom "
                    + "ExpressionEngine that overrides supportsStringCreation().");
            }
            return;
        }
    }
    throw new IllegalArgumentException(
        "No ExpressionEngine registered for expressionLang '" + expressionLang + "'");
}
```

---

### 4. `JQExpressionEngine` — override both methods

```java
@Override
public ExpressionEvaluator create(final String expression) {
    return new JQExpressionEvaluator(expression);
}

@Override
public boolean supportsStringCreation() {
    return true;
}
```

---

### 5. Schema — add `expressionLang`

Add to `schema/src/main/resources/schema/CaseDefinition.yaml` at the top level:

```yaml
expressionLang:
  type: string
  title: ExpressionLang
  description: >
    Expression language for all condition expressions in this definition.
    Must match the type() of a registered ExpressionEngine that overrides create().
    Defaults to "jq".
  default: "jq"
```

Regenerate schema model: `io.casehub.model.CaseDefinition` gains `getExpressionLang()`.

**`expressionLang` is NOT added to `io.casehub.api.model.CaseDefinition`.**

After parsing, every `ExpressionEvaluator` in the definition carries its language through
`type()`. There is no runtime use for `expressionLang` on the API model. For programmatic
(Java DSL) definitions, a field that defaults to `"jq"` would silently misrepresent definitions
that mix evaluator types. `expressionLang` is a parsing directive; it belongs only in the schema
model and as a local variable in `convertToApiModel`.

---

### 6. `CaseDefinitionYamlMapper` — fix the root cause

Remove:
- `private static ObjectMapper yamlMapper` field
- `public static void setObjectMapper(ObjectMapper)` method

Add the full-featured overload (all dependencies as parameters — no static state, no injection
race):

```java
public static CaseDefinition load(InputStream stream,
                                   ObjectMapper objectMapper,
                                   ExpressionEngineRegistry registry) throws IOException { ... }
```

Inside `convertToApiModel`, read `expressionLang` as a local variable (null-safe):

```java
String expressionLang = schema.getExpressionLang() != null ? schema.getExpressionLang() : "jq";
registry.assertLanguageSupported(expressionLang);  // fail-fast, no side effects
```

Then replace all five `new JQExpressionEvaluator(expr)` calls with
`registry.create(expr, expressionLang)`.

Add a JQ-only convenience overload for non-CDI contexts (tests, tooling). `ExpressionEngineRegistry`
has four abstract methods and cannot be expressed as a lambda; use a named static field with an
anonymous class:

```java
private static final ExpressionEngineRegistry JQ_ONLY = new ExpressionEngineRegistry() {
    @Override
    public ExpressionEvaluator create(String expression, String expressionLang) {
        if (!JQExpressionEvaluator.TYPE.equals(expressionLang)) {
            throw new IllegalArgumentException(
                "No CDI registry available; only 'jq' is supported without injection. Got: "
                + expressionLang);
        }
        return new JQExpressionEvaluator(expression);
    }
    @Override public void assertLanguageSupported(String lang) {
        if (!JQExpressionEvaluator.TYPE.equals(lang))
            throw new IllegalArgumentException("Only 'jq' supported without CDI. Got: " + lang);
    }
    @Override public boolean evaluate(ExpressionEvaluator e, CaseContext c) {
        throw new UnsupportedOperationException("Evaluation requires CDI"); }
    @Override public boolean evaluate(ExpressionEvaluator e, JsonNode n) {
        throw new UnsupportedOperationException("Evaluation requires CDI"); }
    @Override public void validate(ExpressionEvaluator e) {
        /* no-op: loading-only registry; validation occurs through the CDI path
           during case definition registration in DefaultCaseDefinitionRegistry */ }
};

/** Non-CDI convenience overload. JQ only; does not support custom expression languages. */
public static CaseDefinition load(InputStream stream) throws IOException {
    return load(stream, new ObjectMapper(new YAMLFactory()), JQ_ONLY);
}
```

---

### 7. `YamlCaseHub` — inject dependencies directly

After the module moves above, `ExpressionEngineRegistry` (now in `api/engine/`) and
`@YamlMapper` (now in `api/marshaller/`) are both reachable from `api`. `jakarta.inject-api`
is already a declared dependency of `api/pom.xml`.

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
                    try (InputStream is =
                            Thread.currentThread().getContextClassLoader()
                                .getResourceAsStream(path)) {
                        if (is == null) {
                            throw new IllegalStateException(
                                "Resource " + path + " not found on classpath");
                        }
                        definition = CaseDefinitionYamlMapper.load(is, objectMapper,
                            expressionEngineRegistry);
                    } catch (IOException e) {
                        throw new RuntimeException(
                            "Failed to load CaseHub definition from " + path, e);
                    }
                }
            }
        }
        return definition;
    }
}
```

No null guard. In a CDI container, `@Inject` fields that cannot be resolved cause deployment
failure — they are never null. The null check would mask missing dependencies as a silent
fallback to JQ-only parsing. Non-CDI tests of the mapper use `CaseDefinitionYamlMapper.load(is)`
directly; they do not go through `YamlCaseHub`.

The `volatile` field and double-checked locking are unchanged — correct lazy initialisation.
Thread safety: CDI-injected fields are written before any application code executes; the
`volatile definition` provides the only required memory barrier.

---

### 8. Delete `ObjectMapperInjector`

No longer needed. The static `yamlMapper` field is gone; the CDI `ObjectMapper` reaches
`CaseDefinitionYamlMapper` as a constructor-style parameter via `YamlCaseHub`. Deleting this
class also deletes the `setObjectMapper()` workaround.

---

## Five Mapper Call Sites

`expressionLang` and `registry` are local variables in `convertToApiModel`. Two of the five
call sites are in private static helpers that must receive them as additional parameters.

### Private method signature changes

```java
// Before:
private static Binding convertBinding(
    io.casehub.model.Binding schemaBinding, Map<String, Capability> capabilityMap)

private static io.casehub.api.model.Trigger convertTrigger(
    io.casehub.model.Trigger schemaTrigger)

// After:
private static Binding convertBinding(
    io.casehub.model.Binding schemaBinding,
    Map<String, Capability> capabilityMap,
    ExpressionEngineRegistry registry,
    String expressionLang)

private static io.casehub.api.model.Trigger convertTrigger(
    io.casehub.model.Trigger schemaTrigger,
    ExpressionEngineRegistry registry,
    String expressionLang)
```

### Call site updates in `convertToApiModel`

```java
// Before:
Binding binding = convertBinding(sr, capabilityMap);
io.casehub.api.model.Trigger trigger = convertTrigger(schemaBinding.getOn());

// After:
Binding binding = convertBinding(sr, capabilityMap, registry, expressionLang);
io.casehub.api.model.Trigger trigger = convertTrigger(schemaBinding.getOn(), registry, expressionLang);
```

### All five sites

| Location | Method | Expression source |
|----------|--------|-----------------|
| `convertBinding` | private helper | `schemaBinding.getWhen()` |
| `convertTrigger` | private helper | `schemaTrigger.getContextChange().getFilter()` |
| `convertToApiModel` | milestones loop | `sm.getCondition()` |
| `convertToApiModel` | milestones loop | `sm.getEntryCriteria()` |
| `convertToApiModel` | goals loop | `sg.getCondition()` |

Each replaces `new JQExpressionEvaluator(expr)` with `registry.create(expr, expressionLang)`.

---

## ADR

File `docs/adr/` entry covering:
- **Decision:** adopt `expressionLang` at YAML definition level, following CNCF SW 1.0
- **Rationale:** CNCF alignment for interoperability; per-definition is the natural unit for a
  deployment; per-expression YAML syntax (mixing languages per-binding) is deferred — it is
  already supported at the model level via `ExpressionEvaluator.type()` and in the Java DSL;
  the YAML syntax concern is separate
- **Consequences:** YAML definitions declaring non-`"jq"` `expressionLang` require a matching
  `ExpressionEngine` CDI bean that overrides `supportsStringCreation()` to `true` and overrides
  `create()`; fail-fast at parse time via `assertLanguageSupported()`

---

## Testing

No static state → no ordering dependencies, no `@AfterEach` resets.

**`CaseDefinitionYamlMapperTest`** — existing tests use `load(is)` (JQ default); all pass
unchanged. Add:
- `expressionLang` field round-trips from YAML schema model through `convertToApiModel`
  (verify `assertLanguageSupported` is called; verify registry receives correct lang)
- Custom language: pass a stub `ExpressionEngineRegistry` (anonymous class) to
  `load(is, mapper, registry)`; verify all five sites call `registry.create()`
- Unknown `expressionLang`: stub `assertLanguageSupported` throws; verify load propagates it
- `load(is)` with non-JQ lang: verify `IllegalArgumentException` from `JQ_ONLY`

**`DefaultExpressionEngineRegistryTest`** — add (to existing `@QuarkusTest` class):
- `create("expr", "jq")` → returns `JQExpressionEvaluator("expr")` with `type() == "jq"`
- `create("expr", "unknown")` → `IllegalArgumentException`
- `create("expr", "lambda")` → `UnsupportedOperationException` (engine exists, no override)
- Type assertion: stub engine whose `create()` returns wrong type → `IllegalStateException`
- `assertLanguageSupported("jq")` → no throw
- `assertLanguageSupported("unknown")` → `IllegalArgumentException`
- `assertLanguageSupported("lambda")` → `UnsupportedOperationException`

**`JQExpressionEngineTest`** — add:
- `create("expr")` returns `JQExpressionEvaluator("expr")`; `type() == "jq"` (invariant check)
- `supportsStringCreation()` returns `true`