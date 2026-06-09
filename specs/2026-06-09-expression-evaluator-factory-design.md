# Expression Evaluator Factory — Design Spec
**Issue:** engine#289  
**Date:** 2026-06-09

## Problem

`CaseDefinitionYamlMapper` hardcodes `new JQExpressionEvaluator(expression)` at five call sites when parsing YAML case definitions. The evaluation side is already fully pluggable (`ExpressionEngine` SPI, `ExpressionEngineRegistry`, CDI discovery) — but the instantiation side at parse time is not. There is no way to produce a different `ExpressionEvaluator` subtype from a YAML definition without modifying the mapper.

Secondary: the YAML schema has no `expressionLang` field, which CNCF Serverless Workflow 1.0 declares at the workflow level.

## What Is Not Changing

The runtime evaluation path (`ExpressionEngine`, `ExpressionEngineRegistry`, `JQExpressionEngine`, `LambdaExpressionEngine`) is already pluggable and is not touched by this change. This spec is about parse-time instantiation only.

## Design

### 1. Schema — add `expressionLang`

Add to `schema/src/main/resources/schema/CaseDefinition.yaml` at the top level, alongside `dsl`:

```yaml
expressionLang:
  type: string
  title: ExpressionLang
  description: >
    Expression language used for all condition expressions in this definition.
    Matches a registered ExpressionEvaluatorFactory. Default: jq.
  default: "jq"
```

Regenerate schema model. `io.casehub.model.CaseDefinition` gets `getExpressionLang()`.

Propagate to `io.casehub.api.model.CaseDefinition`: add `private String expressionLang` field with getter/setter and default `"jq"` in `convertToApiModel`.

### 2. New SPI — `ExpressionEvaluatorFactory`

Location: `api/src/main/java/io/casehub/api/spi/ExpressionEvaluatorFactory.java`

```java
public interface ExpressionEvaluatorFactory {
    ExpressionEvaluator create(String expression, String expressionLang);
}
```

Single method. Takes the raw expression string and the language identifier declared on the case definition. Returns the appropriate `ExpressionEvaluator` subtype. Throws `IllegalArgumentException` for unknown languages.

### 3. `CaseDefinitionYamlMapper` — use the factory

Static field with sensible default (JQ inline, no CDI deps):

```java
private static ExpressionEvaluatorFactory expressionEvaluatorFactory =
    (expression, lang) -> switch (lang) {
        case "jq" -> new JQExpressionEvaluator(expression);
        default -> throw new IllegalArgumentException("Unknown expressionLang: " + lang);
    };

public static void setExpressionEvaluatorFactory(ExpressionEvaluatorFactory factory) {
    expressionEvaluatorFactory = factory;
}
```

`convertToApiModel` reads `expressionLang` from the schema model (null-safe, defaults `"jq"`), passes it through to all five factory call sites that currently hardcode `new JQExpressionEvaluator(...)`.

Follows the same injection pattern as the existing `setObjectMapper()` workaround (already present; fixing that is out of scope here).

### 4. Runtime CDI wiring

**`DefaultExpressionEvaluatorFactory`** — `@ApplicationScoped @DefaultBean` in `runtime`:

```java
@ApplicationScoped
@DefaultBean
public class DefaultExpressionEvaluatorFactory implements ExpressionEvaluatorFactory {
    @Override
    public ExpressionEvaluator create(String expression, String expressionLang) {
        return switch (expressionLang) {
            case "jq" -> new JQExpressionEvaluator(expression);
            default -> throw new IllegalArgumentException(
                "No ExpressionEvaluatorFactory registered for expressionLang '" + expressionLang + "'");
        };
    }
}
```

**`ExpressionEvaluatorFactoryInjector`** — `@ApplicationScoped` in `runtime`, observes `StartupEvent`:

```java
@ApplicationScoped
public class ExpressionEvaluatorFactoryInjector {
    @Inject ExpressionEvaluatorFactory factory;

    void onStartup(@Observes StartupEvent event) {
        CaseDefinitionYamlMapper.setExpressionEvaluatorFactory(factory);
    }
}
```

Consistent with the existing `ObjectMapperInjector` pattern.

Consumer override: provide an `@ApplicationScoped @Alternative @Priority(1)` implementation of `ExpressionEvaluatorFactory`.

## Five Mapper Call Sites

| Method | Expression source |
|--------|------------------|
| `convertBinding` | `schemaBinding.getWhen()` |
| `convertTrigger` | `schemaTrigger.getContextChange().getFilter()` |
| `convertToApiModel` (milestones) | `sm.getCondition()`, `sm.getEntryCriteria()` |
| `convertToApiModel` (goals) | `sg.getCondition()` |

All five replace `new JQExpressionEvaluator(expr)` with `expressionEvaluatorFactory.create(expr, expressionLang)`.

## Testing

- `CaseDefinitionYamlMapperTest`: existing tests still pass (default factory = JQ)
- New: test with custom factory that produces a stub `ExpressionEvaluator` — verify all five sites use it
- New: test `expressionLang` field round-trips from YAML into `CaseDefinition.expressionLang`
- New: `DefaultExpressionEvaluatorFactory` unit test — JQ produces correct type; unknown lang throws

## What This Enables

A consumer deploys a custom `ExpressionEvaluatorFactory @Alternative @Priority(1)` alongside a matching `ExpressionEngine @ApplicationScoped` CDI bean. Case definitions declare `expressionLang: <their-lang>`. Both instantiation (factory) and evaluation (engine) are now pluggable end-to-end.
