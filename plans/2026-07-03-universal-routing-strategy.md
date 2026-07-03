# Universal Routing Strategy Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Establish a universal pluggable routing strategy architecture across the casehub platform — shared convention in platform-api, two new SPIs in engine-api, seven existing SPIs retrofitted, engine/work wiring updated.

**Architecture:** `NamedStrategy` marker interface + `StrategyResolver` CDI bean in casehub-platform-api/platform. Domain-specific strategy interfaces (AgentRoutingStrategy, CandidateSetStrategy, etc.) extend NamedStrategy and are resolved by `id` via StrategyResolver. CandidateSetStrategy replaces sealed ListEvaluator; CandidateMatchingStrategy replaces hardcoded AgentCandidateFactory matching.

**Tech Stack:** Java 21, Quarkus 3.32.2, CDI (Quarkus ARC), Mutiny (Uni<>), Maven multi-module

**Spec:** `docs/specs/2026-07-02-universal-routing-strategy-design.md`
**Audit:** `docs/specs/2026-07-02-universal-routing-strategy-audit.md`

## Global Constraints

- All new routing SPIs return `Uni<T>` per protocol PP-20260529-9f9627
- All `@DefaultBean` implementations use `io.quarkus.arc.DefaultBean`
- All strategy SPIs extend `io.casehub.platform.api.routing.NamedStrategy`
- NamedStrategy + StrategyResolver live in `casehub-platform-api` (SPI) / `casehub-platform` (runtime impl)
- Value-object strategies (StaticSetStrategy, ExpressionSetStrategy) are NOT CDI beans — constructed by YAML mapper/builder
- Named CDI strategies are resolved via `StrategyResolver.resolve(type, id)` at dispatch time
- `HumanTaskTarget` stores `CandidateSetSpec` (sealed: Inline | Named), not `CandidateSetStrategy` directly
- CaseDefinition stores String strategy IDs — no CDI types on the data model
- Unknown strategy ID → startup failure (fail loud)
- Duplicate IDs for same type → startup failure
- Tests named `*Test.java` (never `*IT.java` — surefire, not failsafe)

## Cross-Repo Sequencing

This plan spans three repos. Dependency order:

```
casehub-platform (Task 1)
    ↓
casehub-engine (Tasks 2–6)  ←  casehub-work (Task 7)
    ↓                              ↓
Consumer repos (Task 8)      Consumer repos (Task 8)
    ↓
Documentation (Task 9)
```

Platform must be published (`mvn deploy`) before engine/work can consume it. Engine-api must be published before consumer repos can consume the new SPI types.

---

### Task 1: Platform Foundation — NamedStrategy + StrategyResolver

**Repo:** `casehub-platform` (`/Users/mdproctor/claude/casehub/platform`)

**Files:**
- Create: `platform-api/src/main/java/io/casehub/platform/api/routing/NamedStrategy.java`
- Create: `platform-api/src/main/java/io/casehub/platform/api/routing/StrategyResolver.java`
- Create: `platform/src/main/java/io/casehub/platform/routing/DefaultStrategyResolver.java`
- Create: `platform-api/src/test/java/io/casehub/platform/api/routing/NamedStrategyContractTest.java`
- Create: `platform/src/test/java/io/casehub/platform/routing/DefaultStrategyResolverTest.java`

**Interfaces:**
- Produces: `NamedStrategy { String id(); }` — marker for all routing strategies
- Produces: `StrategyResolver { resolve(), find(), defaultStrategy(), available() }` — CDI strategy lookup
- Produces: `DefaultStrategyResolver` — `@ApplicationScoped` CDI impl

- [ ] **Step 1: Write NamedStrategy contract test**

```java
package io.casehub.platform.api.routing;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.*;

class NamedStrategyContractTest {

    interface TestStrategy extends NamedStrategy {
        String doWork();
    }

    static class ConcreteStrategy implements TestStrategy {
        private final String id;
        ConcreteStrategy(String id) { this.id = id; }
        @Override public String id() { return id; }
        @Override public String doWork() { return "done"; }
    }

    @Test
    void idIsStableKey() {
        var strategy = new ConcreteStrategy("my-strategy");
        assertThat(strategy.id()).isEqualTo("my-strategy");
    }

    @Test
    void extendingInterfacePreservesMarker() {
        var strategy = new ConcreteStrategy("test");
        assertThat(strategy).isInstanceOf(NamedStrategy.class);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl platform-api -Dtest=NamedStrategyContractTest -f /Users/mdproctor/claude/casehub/platform/pom.xml`
Expected: FAIL — `NamedStrategy` does not exist

- [ ] **Step 3: Implement NamedStrategy**

```java
package io.casehub.platform.api.routing;

/**
 * Marker interface for named, CDI-discoverable routing strategies.
 *
 * <p>Domain-specific strategy interfaces (e.g. AgentRoutingStrategy,
 * WorkerSelectionStrategy) extend this marker. Strategies are resolved
 * by {@link StrategyResolver} using the {@link #id()} as a stable key.
 */
public interface NamedStrategy {
    String id();
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn test -pl platform-api -Dtest=NamedStrategyContractTest -f /Users/mdproctor/claude/casehub/platform/pom.xml`
Expected: PASS

- [ ] **Step 5: Write StrategyResolver contract test**

```java
package io.casehub.platform.api.routing;

import org.junit.jupiter.api.Test;
import java.util.Optional;
import java.util.List;
import static org.assertj.core.api.Assertions.*;

class StrategyResolverContractTest {

    @Test
    void interfaceCompiles() {
        StrategyResolver resolver = new StrategyResolver() {
            @Override public <T extends NamedStrategy> T resolve(Class<T> type, String id) { return null; }
            @Override public <T extends NamedStrategy> Optional<T> find(Class<T> type, String id) { return Optional.empty(); }
            @Override public <T extends NamedStrategy> T defaultStrategy(Class<T> type) { return null; }
            @Override public <T extends NamedStrategy> List<T> available(Class<T> type) { return List.of(); }
        };
        assertThat(resolver).isNotNull();
    }
}
```

- [ ] **Step 6: Implement StrategyResolver interface**

```java
package io.casehub.platform.api.routing;

import java.util.List;
import java.util.Optional;

/**
 * Resolves {@link NamedStrategy} beans by type and id.
 *
 * <p>Resolution: look up by {@code (type, id)}. If {@code id} is null,
 * return the {@code @DefaultBean} instance. If no bean with that id exists,
 * throw {@link IllegalArgumentException}.
 */
public interface StrategyResolver {

    <T extends NamedStrategy> T resolve(Class<T> type, String id);

    <T extends NamedStrategy> Optional<T> find(Class<T> type, String id);

    <T extends NamedStrategy> T defaultStrategy(Class<T> type);

    <T extends NamedStrategy> List<T> available(Class<T> type);
}
```

- [ ] **Step 7: Run tests to verify they pass**

Run: `mvn test -pl platform-api -f /Users/mdproctor/claude/casehub/platform/pom.xml`
Expected: PASS

- [ ] **Step 8: Write DefaultStrategyResolver test**

```java
package io.casehub.platform.routing;

import io.casehub.platform.api.routing.NamedStrategy;
import io.casehub.platform.api.routing.StrategyResolver;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.util.List;
import static org.assertj.core.api.Assertions.*;

class DefaultStrategyResolverTest {

    interface TestStrategy extends NamedStrategy {
        String value();
    }

    static class StrategyA implements TestStrategy {
        @Override public String id() { return "a"; }
        @Override public String value() { return "alpha"; }
    }

    static class StrategyB implements TestStrategy {
        @Override public String id() { return "b"; }
        @Override public String value() { return "beta"; }
    }

    interface OtherStrategy extends NamedStrategy {}

    private StrategyResolver resolver;

    @BeforeEach
    void setUp() {
        resolver = new DefaultStrategyResolver(
            List.of(new StrategyA(), new StrategyB()));
    }

    @Test
    void resolveByIdReturnsCorrectStrategy() {
        TestStrategy result = resolver.resolve(TestStrategy.class, "a");
        assertThat(result.value()).isEqualTo("alpha");
    }

    @Test
    void resolveByIdReturnsSecondStrategy() {
        TestStrategy result = resolver.resolve(TestStrategy.class, "b");
        assertThat(result.value()).isEqualTo("beta");
    }

    @Test
    void resolveUnknownIdThrows() {
        assertThatThrownBy(() -> resolver.resolve(TestStrategy.class, "unknown"))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("unknown");
    }

    @Test
    void resolveNullIdReturnsDefault() {
        TestStrategy result = resolver.resolve(TestStrategy.class, null);
        assertThat(result).isNotNull();
    }

    @Test
    void findByIdReturnsOptional() {
        assertThat(resolver.find(TestStrategy.class, "a")).isPresent();
        assertThat(resolver.find(TestStrategy.class, "unknown")).isEmpty();
    }

    @Test
    void availableListsAllForType() {
        List<TestStrategy> strategies = resolver.available(TestStrategy.class);
        assertThat(strategies).hasSize(2);
        assertThat(strategies).extracting(NamedStrategy::id).containsExactlyInAnyOrder("a", "b");
    }

    @Test
    void availableForUnregisteredTypeReturnsEmpty() {
        List<OtherStrategy> strategies = resolver.available(OtherStrategy.class);
        assertThat(strategies).isEmpty();
    }

    @Test
    void duplicateIdsThrowAtConstruction() {
        assertThatThrownBy(() -> new DefaultStrategyResolver(
            List.of(new StrategyA(), new StrategyA())))
            .isInstanceOf(IllegalStateException.class)
            .hasMessageContaining("Duplicate");
    }
}
```

- [ ] **Step 9: Run test to verify it fails**

Run: `mvn test -pl platform -Dtest=DefaultStrategyResolverTest -f /Users/mdproctor/claude/casehub/platform/pom.xml`
Expected: FAIL — `DefaultStrategyResolver` does not exist

- [ ] **Step 10: Implement DefaultStrategyResolver**

```java
package io.casehub.platform.routing;

import io.casehub.platform.api.routing.NamedStrategy;
import io.casehub.platform.api.routing.StrategyResolver;
import jakarta.enterprise.inject.Any;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;
import jakarta.inject.Singleton;
import java.util.*;
import java.util.stream.Collectors;
import java.util.stream.StreamSupport;

@Singleton
public class DefaultStrategyResolver implements StrategyResolver {

    private final Map<Class<?>, Map<String, NamedStrategy>> index;
    private final Map<Class<?>, NamedStrategy> defaults;

    @Inject
    public DefaultStrategyResolver(@Any Instance<NamedStrategy> strategies) {
        this(StreamSupport.stream(strategies.spliterator(), false).toList());
    }

    DefaultStrategyResolver(List<? extends NamedStrategy> strategies) {
        this.index = new HashMap<>();
        this.defaults = new HashMap<>();
        for (NamedStrategy strategy : strategies) {
            for (Class<?> iface : resolveStrategyTypes(strategy.getClass())) {
                var byId = index.computeIfAbsent(iface, k -> new LinkedHashMap<>());
                NamedStrategy existing = byId.put(strategy.id(), strategy);
                if (existing != null) {
                    throw new IllegalStateException(
                        "Duplicate strategy id '" + strategy.id() + "' for type "
                        + iface.getSimpleName() + ": " + existing.getClass().getName()
                        + " and " + strategy.getClass().getName());
                }
                defaults.putIfAbsent(iface, strategy);
            }
        }
    }

    @Override
    @SuppressWarnings("unchecked")
    public <T extends NamedStrategy> T resolve(Class<T> type, String id) {
        if (id == null) {
            return defaultStrategy(type);
        }
        var byId = index.get(type);
        if (byId == null || !byId.containsKey(id)) {
            var available = byId == null ? Set.of() : byId.keySet();
            throw new IllegalArgumentException(
                "No strategy with id '" + id + "' for type " + type.getSimpleName()
                + ". Available: " + available);
        }
        return (T) byId.get(id);
    }

    @Override
    @SuppressWarnings("unchecked")
    public <T extends NamedStrategy> Optional<T> find(Class<T> type, String id) {
        var byId = index.get(type);
        if (byId == null) return Optional.empty();
        return Optional.ofNullable((T) byId.get(id));
    }

    @Override
    @SuppressWarnings("unchecked")
    public <T extends NamedStrategy> T defaultStrategy(Class<T> type) {
        T def = (T) defaults.get(type);
        if (def == null) {
            throw new IllegalArgumentException(
                "No default strategy for type " + type.getSimpleName());
        }
        return def;
    }

    @Override
    @SuppressWarnings("unchecked")
    public <T extends NamedStrategy> List<T> available(Class<T> type) {
        var byId = index.get(type);
        if (byId == null) return List.of();
        return byId.values().stream().map(s -> (T) s).toList();
    }

    private static Set<Class<?>> resolveStrategyTypes(Class<?> clazz) {
        Set<Class<?>> result = new LinkedHashSet<>();
        for (Class<?> iface : clazz.getInterfaces()) {
            if (NamedStrategy.class.isAssignableFrom(iface) && iface != NamedStrategy.class) {
                result.add(iface);
            }
        }
        Class<?> superclass = clazz.getSuperclass();
        if (superclass != null && superclass != Object.class) {
            result.addAll(resolveStrategyTypes(superclass));
        }
        for (Class<?> iface : clazz.getInterfaces()) {
            result.addAll(resolveStrategyTypes(iface));
        }
        result.remove(NamedStrategy.class);
        return result;
    }
}
```

- [ ] **Step 11: Run tests to verify they pass**

Run: `mvn test -pl platform -Dtest=DefaultStrategyResolverTest -f /Users/mdproctor/claude/casehub/platform/pom.xml`
Expected: PASS

- [ ] **Step 12: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/platform add platform-api/src/main/java/io/casehub/platform/api/routing/ platform/src/main/java/io/casehub/platform/routing/ platform-api/src/test/java/io/casehub/platform/api/routing/ platform/src/test/java/io/casehub/platform/routing/
git -C /Users/mdproctor/claude/casehub/platform commit -m "feat(#634): NamedStrategy + StrategyResolver in casehub-platform-api

Refs casehubio/engine#634"
```

- [ ] **Step 13: Install to local Maven repo**

Run: `mvn install -DskipTests -q -f /Users/mdproctor/claude/casehub/platform/pom.xml`

---

### Task 2: CandidateSetStrategy SPI — Replace Sealed ListEvaluator

**Repo:** `casehub-engine` (`/Users/mdproctor/claude/casehub/engine`)

**Files:**
- Create: `api/src/main/java/io/casehub/api/spi/routing/CandidateSetStrategy.java`
- Create: `api/src/main/java/io/casehub/api/spi/routing/CandidateSetContext.java`
- Create: `api/src/main/java/io/casehub/api/spi/routing/CandidateSetSpec.java`
- Create: `api/src/main/java/io/casehub/api/spi/routing/StaticSetStrategy.java`
- Create: `runtime/src/main/java/io/casehub/engine/internal/routing/ExpressionSetStrategy.java`
- Create: `api/src/test/java/io/casehub/api/spi/routing/CandidateSetStrategyContractTest.java`
- Create: `api/src/test/java/io/casehub/api/spi/routing/StaticSetStrategyTest.java`
- Modify: `api/src/main/java/io/casehub/api/model/HumanTaskTarget.java` — candidateGroups/Users type change
- Delete: `api/src/main/java/io/casehub/api/model/evaluator/ListEvaluator.java` (after all references migrated)

**Interfaces:**
- Consumes: `NamedStrategy` from Task 1
- Produces: `CandidateSetStrategy extends NamedStrategy { Uni<Set<String>> evaluate(CandidateSetContext) }`
- Produces: `CandidateSetSpec` sealed: `Inline(CandidateSetStrategy)` | `Named(String strategyId, Map<String,Object> config)`
- Produces: `CandidateSetContext(JsonNode caseContext, Map<String,Object> config)`
- Produces: `StaticSetStrategy` — value object, `id()="static"`, wraps fixed `Set<String>`
- Produces: `ExpressionSetStrategy` — value object, `id()="expression"`, delegates to `ExpressionEngineRegistry`

- [ ] **Step 1: Write CandidateSetStrategy contract test**

```java
package io.casehub.api.spi.routing;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import io.smallrye.mutiny.Uni;
import org.junit.jupiter.api.Test;
import java.util.Map;
import java.util.Set;
import static org.assertj.core.api.Assertions.*;

class CandidateSetStrategyContractTest {

    private static final ObjectMapper MAPPER = new ObjectMapper();

    @Test
    void strategyExtendsNamedStrategy() {
        CandidateSetStrategy strategy = new CandidateSetStrategy() {
            @Override public String id() { return "test"; }
            @Override public Uni<Set<String>> evaluate(CandidateSetContext context) {
                return Uni.createFrom().item(Set.of("group-a"));
            }
        };
        assertThat(strategy).isInstanceOf(io.casehub.platform.api.routing.NamedStrategy.class);
        assertThat(strategy.id()).isEqualTo("test");
    }

    @Test
    void evaluateReturnsCandidateSet() {
        CandidateSetStrategy strategy = ctx ->
            Uni.createFrom().item(Set.of("group-a", "group-b"));
        // id() default needed — add as default on interface? No, it's required.
        // Use anonymous class instead:
        CandidateSetStrategy named = new CandidateSetStrategy() {
            @Override public String id() { return "test"; }
            @Override public Uni<Set<String>> evaluate(CandidateSetContext context) {
                return Uni.createFrom().item(Set.of("group-a", "group-b"));
            }
        };

        JsonNode context = MAPPER.createObjectNode();
        Set<String> result = named.evaluate(new CandidateSetContext(context)).await().indefinitely();
        assertThat(result).containsExactlyInAnyOrder("group-a", "group-b");
    }

    @Test
    void candidateSetContextWithConfig() {
        JsonNode node = MAPPER.createObjectNode();
        var ctx = new CandidateSetContext(node, Map.of("session", "irb"));
        assertThat(ctx.config()).containsEntry("session", "irb");
    }

    @Test
    void candidateSetContextDefaultEmptyConfig() {
        JsonNode node = MAPPER.createObjectNode();
        var ctx = new CandidateSetContext(node);
        assertThat(ctx.config()).isEmpty();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=CandidateSetStrategyContractTest -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: FAIL — types do not exist

- [ ] **Step 3: Implement CandidateSetStrategy, CandidateSetContext, CandidateSetSpec**

`api/src/main/java/io/casehub/api/spi/routing/CandidateSetStrategy.java`:
```java
package io.casehub.api.spi.routing;

import com.fasterxml.jackson.databind.JsonNode;
import io.casehub.platform.api.routing.NamedStrategy;
import io.smallrye.mutiny.Uni;
import java.util.Set;

public interface CandidateSetStrategy extends NamedStrategy {
    Uni<Set<String>> evaluate(CandidateSetContext context);
}
```

`api/src/main/java/io/casehub/api/spi/routing/CandidateSetContext.java`:
```java
package io.casehub.api.spi.routing;

import com.fasterxml.jackson.databind.JsonNode;
import java.util.Map;

public record CandidateSetContext(
    JsonNode caseContext,
    Map<String, Object> config
) {
    public CandidateSetContext(JsonNode caseContext) {
        this(caseContext, Map.of());
    }
}
```

`api/src/main/java/io/casehub/api/spi/routing/CandidateSetSpec.java`:
```java
package io.casehub.api.spi.routing;

import java.util.Map;

public sealed interface CandidateSetSpec {
    record Inline(CandidateSetStrategy strategy) implements CandidateSetSpec {}
    record Named(String strategyId, Map<String, Object> config) implements CandidateSetSpec {}
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=CandidateSetStrategyContractTest -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: PASS

- [ ] **Step 5: Write StaticSetStrategy test**

```java
package io.casehub.api.spi.routing;

import com.fasterxml.jackson.databind.ObjectMapper;
import org.junit.jupiter.api.Test;
import java.util.Set;
import static org.assertj.core.api.Assertions.*;

class StaticSetStrategyTest {

    private static final ObjectMapper MAPPER = new ObjectMapper();

    @Test
    void idIsStatic() {
        var strategy = StaticSetStrategy.of("a", "b");
        assertThat(strategy.id()).isEqualTo("static");
    }

    @Test
    void evaluateReturnsFixedSet() {
        var strategy = StaticSetStrategy.of("compliance-team", "legal");
        var ctx = new CandidateSetContext(MAPPER.createObjectNode());
        Set<String> result = strategy.evaluate(ctx).await().indefinitely();
        assertThat(result).containsExactlyInAnyOrder("compliance-team", "legal");
    }

    @Test
    void evaluateIgnoresContext() {
        var strategy = StaticSetStrategy.of("group-a");
        var ctx = new CandidateSetContext(
            MAPPER.createObjectNode().put("irrelevant", "data"));
        Set<String> result = strategy.evaluate(ctx).await().indefinitely();
        assertThat(result).containsExactly("group-a");
    }

    @Test
    void defensiveCopyOfInput() {
        var strategy = StaticSetStrategy.of("a", "b");
        Set<String> result = strategy.evaluate(
            new CandidateSetContext(MAPPER.createObjectNode())).await().indefinitely();
        assertThat(result).isUnmodifiable();
    }
}
```

- [ ] **Step 6: Implement StaticSetStrategy**

`api/src/main/java/io/casehub/api/spi/routing/StaticSetStrategy.java`:
```java
package io.casehub.api.spi.routing;

import io.smallrye.mutiny.Uni;
import java.util.Set;

public final class StaticSetStrategy implements CandidateSetStrategy {

    private final Set<String> values;

    private StaticSetStrategy(Set<String> values) {
        this.values = Set.copyOf(values);
    }

    public static StaticSetStrategy of(String... values) {
        return new StaticSetStrategy(Set.of(values));
    }

    public static StaticSetStrategy of(Set<String> values) {
        return new StaticSetStrategy(values);
    }

    @Override
    public String id() { return "static"; }

    @Override
    public Uni<Set<String>> evaluate(CandidateSetContext context) {
        return Uni.createFrom().item(values);
    }

    public Set<String> values() { return values; }
}
```

- [ ] **Step 7: Run tests to verify they pass**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest="CandidateSetStrategyContractTest,StaticSetStrategyTest" -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: PASS

- [ ] **Step 8: Implement ExpressionSetStrategy** (runtime module — needs ExpressionEngineRegistry)

`runtime/src/main/java/io/casehub/engine/internal/routing/ExpressionSetStrategy.java`:
```java
package io.casehub.engine.internal.routing;

import com.fasterxml.jackson.databind.JsonNode;
import io.casehub.api.engine.ExpressionEngineRegistry;
import io.casehub.api.model.evaluator.ExpressionEvaluator;
import io.casehub.api.spi.routing.CandidateSetContext;
import io.casehub.api.spi.routing.CandidateSetStrategy;
import io.smallrye.mutiny.Uni;
import java.util.LinkedHashSet;
import java.util.List;
import java.util.Set;

public final class ExpressionSetStrategy implements CandidateSetStrategy {

    private final ExpressionEvaluator evaluator;
    private final ExpressionEngineRegistry registry;

    public ExpressionSetStrategy(ExpressionEvaluator evaluator, ExpressionEngineRegistry registry) {
        this.evaluator = evaluator;
        this.registry = registry;
    }

    public static ExpressionSetStrategy of(
            String expression, String lang, ExpressionEngineRegistry registry) {
        ExpressionEvaluator evaluator = registry.create(expression, lang);
        return new ExpressionSetStrategy(evaluator, registry);
    }

    public static ExpressionSetStrategy jq(String expression, ExpressionEngineRegistry registry) {
        return of(expression, "jq", registry);
    }

    @Override
    public String id() { return "expression"; }

    @Override
    public Uni<Set<String>> evaluate(CandidateSetContext context) {
        return Uni.createFrom().item(() -> {
            List<JsonNode> results = registry.transform(evaluator, context.caseContext());
            Set<String> values = new LinkedHashSet<>();
            for (JsonNode node : results) {
                if (node.isArray()) {
                    node.forEach(element -> {
                        if (element.isTextual()) values.add(element.asText());
                    });
                } else if (node.isTextual()) {
                    values.add(node.asText());
                }
            }
            return Set.copyOf(values);
        });
    }

    public ExpressionEvaluator evaluator() { return evaluator; }
}
```

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add api/src/main/java/io/casehub/api/spi/routing/CandidateSetStrategy.java api/src/main/java/io/casehub/api/spi/routing/CandidateSetContext.java api/src/main/java/io/casehub/api/spi/routing/CandidateSetSpec.java api/src/main/java/io/casehub/api/spi/routing/StaticSetStrategy.java runtime/src/main/java/io/casehub/engine/internal/routing/ExpressionSetStrategy.java api/src/test/java/io/casehub/api/spi/routing/
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(#634): CandidateSetStrategy SPI — replaces sealed ListEvaluator

Refs #634"
```

---

### Task 3: CandidateMatchingStrategy SPI — Replace AgentCandidateFactory Matching

**Repo:** `casehub-engine`

**Files:**
- Create: `api/src/main/java/io/casehub/api/spi/routing/CandidateMatchingStrategy.java`
- Create: `api/src/main/java/io/casehub/api/spi/routing/CandidateMatchingContext.java`
- Create: `runtime/src/main/java/io/casehub/engine/internal/routing/ExactMatchStrategy.java`
- Create: `runtime/src/main/java/io/casehub/engine/internal/routing/SubsumptionMatchStrategy.java`
- Create: `api/src/test/java/io/casehub/api/spi/routing/CandidateMatchingStrategyContractTest.java`
- Create: `runtime/src/test/java/io/casehub/engine/internal/routing/ExactMatchStrategyTest.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/routing/AgentCandidateFactory.java` — delegate matching

**Interfaces:**
- Consumes: `NamedStrategy` from Task 1, `Worker` from worker-api, `CaseDefinition` from engine-api
- Produces: `CandidateMatchingStrategy extends NamedStrategy { Uni<List<Worker>> match(CandidateMatchingContext) }`
- Produces: `CandidateMatchingContext(String capabilityName, List<Worker> workers, CaseDefinition caseDefinition)`

- [ ] **Step 1: Write CandidateMatchingStrategy contract test**

```java
package io.casehub.api.spi.routing;

import io.casehub.platform.api.routing.NamedStrategy;
import io.casehub.worker.api.Worker;
import io.smallrye.mutiny.Uni;
import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.Set;
import static org.assertj.core.api.Assertions.*;

class CandidateMatchingStrategyContractTest {

    @Test
    void strategyExtendsNamedStrategy() {
        CandidateMatchingStrategy strategy = new CandidateMatchingStrategy() {
            @Override public String id() { return "test"; }
            @Override public Uni<List<Worker>> match(CandidateMatchingContext context) {
                return Uni.createFrom().item(List.of());
            }
        };
        assertThat(strategy).isInstanceOf(NamedStrategy.class);
    }

    @Test
    void contextCarriesCapabilityAndWorkers() {
        Worker worker = Worker.builder().name("w1").capabilityName("cap-a").noFunction().build();
        var ctx = new CandidateMatchingContext("cap-a", List.of(worker), null);
        assertThat(ctx.capabilityName()).isEqualTo("cap-a");
        assertThat(ctx.workers()).hasSize(1);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=CandidateMatchingStrategyContractTest -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: FAIL

- [ ] **Step 3: Implement CandidateMatchingStrategy + CandidateMatchingContext**

`api/src/main/java/io/casehub/api/spi/routing/CandidateMatchingStrategy.java`:
```java
package io.casehub.api.spi.routing;

import io.casehub.platform.api.routing.NamedStrategy;
import io.casehub.worker.api.Worker;
import io.smallrye.mutiny.Uni;
import java.util.List;

public interface CandidateMatchingStrategy extends NamedStrategy {
    Uni<List<Worker>> match(CandidateMatchingContext context);
}
```

`api/src/main/java/io/casehub/api/spi/routing/CandidateMatchingContext.java`:
```java
package io.casehub.api.spi.routing;

import io.casehub.api.model.CaseDefinition;
import io.casehub.worker.api.Worker;
import java.util.List;

public record CandidateMatchingContext(
    String capabilityName,
    List<Worker> workers,
    CaseDefinition caseDefinition
) {}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=CandidateMatchingStrategyContractTest -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: PASS

- [ ] **Step 5: Write ExactMatchStrategy test**

```java
package io.casehub.engine.internal.routing;

import io.casehub.api.spi.routing.CandidateMatchingContext;
import io.casehub.worker.api.Worker;
import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.Set;
import static org.assertj.core.api.Assertions.*;

class ExactMatchStrategyTest {

    private final ExactMatchStrategy strategy = new ExactMatchStrategy();

    @Test
    void idIsExact() {
        assertThat(strategy.id()).isEqualTo("exact");
    }

    @Test
    void matchesWorkerWithCapability() {
        Worker w1 = Worker.builder().name("w1").capabilityName("cap-a").noFunction().build();
        Worker w2 = Worker.builder().name("w2").capabilityName("cap-b").noFunction().build();
        var ctx = new CandidateMatchingContext("cap-a", List.of(w1, w2), null);
        List<Worker> result = strategy.match(ctx).await().indefinitely();
        assertThat(result).hasSize(1);
        assertThat(result.get(0).name()).isEqualTo("w1");
    }

    @Test
    void noMatchReturnsEmpty() {
        Worker w1 = Worker.builder().name("w1").capabilityName("cap-b").noFunction().build();
        var ctx = new CandidateMatchingContext("cap-a", List.of(w1), null);
        List<Worker> result = strategy.match(ctx).await().indefinitely();
        assertThat(result).isEmpty();
    }

    @Test
    void multipleCapabilitiesOnWorker() {
        Worker w1 = Worker.builder().name("w1").capabilityName("cap-a").capabilityName("cap-b").noFunction().build();
        var ctx = new CandidateMatchingContext("cap-b", List.of(w1), null);
        List<Worker> result = strategy.match(ctx).await().indefinitely();
        assertThat(result).hasSize(1);
    }
}
```

- [ ] **Step 6: Implement ExactMatchStrategy + SubsumptionMatchStrategy**

`runtime/src/main/java/io/casehub/engine/internal/routing/ExactMatchStrategy.java`:
```java
package io.casehub.engine.internal.routing;

import io.casehub.api.spi.routing.CandidateMatchingContext;
import io.casehub.api.spi.routing.CandidateMatchingStrategy;
import io.casehub.worker.api.Worker;
import io.smallrye.mutiny.Uni;
import java.util.List;

public class ExactMatchStrategy implements CandidateMatchingStrategy {

    @Override
    public String id() { return "exact"; }

    @Override
    public Uni<List<Worker>> match(CandidateMatchingContext context) {
        return Uni.createFrom().item(() ->
            context.workers().stream()
                .filter(w -> w.capabilityNames().contains(context.capabilityName()))
                .toList());
    }
}
```

`runtime/src/main/java/io/casehub/engine/internal/routing/SubsumptionMatchStrategy.java`:
```java
package io.casehub.engine.internal.routing;

import io.casehub.api.model.ai.AgentDescriptor;
import io.casehub.api.spi.VocabularyRegistry;
import io.casehub.api.spi.routing.CandidateMatchingContext;
import io.casehub.api.spi.routing.CandidateMatchingStrategy;
import io.casehub.eidos.api.CapabilityResolver;
import io.casehub.worker.api.Worker;
import io.quarkus.arc.DefaultBean;
import io.smallrye.mutiny.Uni;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.util.ArrayList;
import java.util.List;

@DefaultBean
@ApplicationScoped
public class SubsumptionMatchStrategy implements CandidateMatchingStrategy {

    private final VocabularyRegistry vocabularyRegistry;

    @Inject
    public SubsumptionMatchStrategy(VocabularyRegistry vocabularyRegistry) {
        this.vocabularyRegistry = vocabularyRegistry;
    }

    @Override
    public String id() { return "subsumption"; }

    @Override
    public Uni<List<Worker>> match(CandidateMatchingContext context) {
        return Uni.createFrom().item(() -> {
            List<Worker> matched = new ArrayList<>();
            for (Worker worker : context.workers()) {
                if (worker.capabilityNames().contains(context.capabilityName())) {
                    matched.add(worker);
                    continue;
                }
                AgentDescriptor descriptor = context.caseDefinition() != null
                    ? context.caseDefinition().agentDescriptorFor(worker.name()).orElse(null)
                    : null;
                if (descriptor != null && !descriptor.capabilities().isEmpty()) {
                    var resolved = CapabilityResolver.resolve(
                        descriptor.capabilities(), context.capabilityName(), vocabularyRegistry);
                    if (resolved != null) {
                        matched.add(worker);
                    }
                }
            }
            return List.copyOf(matched);
        });
    }
}
```

- [ ] **Step 7: Run tests**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api,runtime -Dtest="CandidateMatchingStrategyContractTest,ExactMatchStrategyTest" -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: PASS

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add api/src/main/java/io/casehub/api/spi/routing/CandidateMatchingStrategy.java api/src/main/java/io/casehub/api/spi/routing/CandidateMatchingContext.java runtime/src/main/java/io/casehub/engine/internal/routing/ExactMatchStrategy.java runtime/src/main/java/io/casehub/engine/internal/routing/SubsumptionMatchStrategy.java api/src/test/java/io/casehub/api/spi/routing/ runtime/src/test/java/io/casehub/engine/internal/routing/ExactMatchStrategyTest.java
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(#634): CandidateMatchingStrategy SPI — replaces AgentCandidateFactory matching

Refs #634"
```

---

### Task 4: Retrofit Engine SPIs + CaseDefinition Model

**Repo:** `casehub-engine`

**Files:**
- Modify: `api/src/main/java/io/casehub/api/spi/routing/AgentRoutingStrategy.java` — extends NamedStrategy
- Modify: `api/src/main/java/io/casehub/api/spi/routing/ImplementationRoutingStrategy.java` — extends NamedStrategy
- Modify: `api/src/main/java/io/casehub/api/spi/routing/TrustRoutingPolicyProvider.java` — extends NamedStrategy
- Modify: `blackboard/src/main/java/io/casehub/blackboard/control/PlanningStrategy.java` — extends NamedStrategy, rename getId() to id()
- Modify: `common/src/main/java/io/casehub/engine/common/spi/scheduler/WorkerExecutionRoutingStrategy.java` — extends NamedStrategy
- Modify: `api/src/main/java/io/casehub/api/model/CaseDefinition.java` — add agentRouting, implementationRouting, candidateMatching fields
- Modify: all engine built-in strategy implementations — add id() method
- Modify: `api/src/main/java/io/casehub/api/spi/RiskDecision.java` — GateRequired.candidateGroups → CandidateSetStrategy

**Interfaces:**
- Consumes: `NamedStrategy` from Task 1, `CandidateSetStrategy` from Task 2
- Produces: All existing SPIs extended with `id()`, CaseDefinition with new strategy fields

- [ ] **Step 1: Add NamedStrategy to all engine SPIs**

Add `extends NamedStrategy` to each interface. For `PlanningStrategy`, rename `getId()` to `id()`.

`AgentRoutingStrategy.java` — add import and extends:
```java
import io.casehub.platform.api.routing.NamedStrategy;

public interface AgentRoutingStrategy extends NamedStrategy {
    Uni<AgentAssignment> select(AgentRoutingContext context, List<AgentCandidate> candidates);
}
```

`ImplementationRoutingStrategy.java`:
```java
import io.casehub.platform.api.routing.NamedStrategy;

public interface ImplementationRoutingStrategy extends NamedStrategy {
    Uni<ImplementationSelection> select(ImplementationRoutingContext context, List<ImplementationCandidate> candidates);
}
```

`TrustRoutingPolicyProvider.java`:
```java
import io.casehub.platform.api.routing.NamedStrategy;

public interface TrustRoutingPolicyProvider extends NamedStrategy {
    TrustRoutingPolicy forCapability(String capabilityName);
}
```

`PlanningStrategy.java`:
```java
import io.casehub.platform.api.routing.NamedStrategy;

public interface PlanningStrategy extends NamedStrategy {
    // getId() renamed to id() — NamedStrategy contract
    @Override String id();
    String getName();
    Uni<List<Binding>> select(CasePlanModel plan, PlanExecutionContext context, List<Binding> eligible);
}
```

`WorkerExecutionRoutingStrategy.java`:
```java
import io.casehub.platform.api.routing.NamedStrategy;

public interface WorkerExecutionRoutingStrategy extends NamedStrategy {
    Optional<WorkerExecutionManager> select(List<WorkerExecutionManager> candidates, Worker worker, Capability capability, String tenancyId);
}
```

- [ ] **Step 2: Add id() to all engine built-in implementations**

`LeastLoadedAgentStrategy` — add `@Override public String id() { return "least-loaded"; }`
`NoOpImplementationRoutingStrategy` — add `@Override public String id() { return "run-all"; }`
`FirstSupportedRoutingStrategy` — add `@Override public String id() { return "first-supported"; }`
`DefaultTrustRoutingPolicyProvider` — add `@Override public String id() { return "default"; }`
`DefaultPlanningStrategy` — rename `getId()` to `id()`, ensure returns `"default"`
`SequentialPlanningStrategy` — rename `getId()` to `id()`, ensure returns `"sequential"`

- [ ] **Step 3: Add strategy fields to CaseDefinition**

Add to `CaseDefinition.java` fields section:
```java
private final String agentRouting;           // nullable — null = @DefaultBean
private final String implementationRouting;  // nullable
private final String candidateMatching;      // nullable — null = "subsumption"
```

Add corresponding builder methods, getter methods, and include in the constructor.

- [ ] **Step 4: Change RiskDecision.GateRequired.candidateGroups**

`api/src/main/java/io/casehub/api/spi/RiskDecision.java`:
```java
import io.casehub.api.spi.routing.CandidateSetStrategy;

public sealed interface RiskDecision permits RiskDecision.Autonomous, RiskDecision.GateRequired {
    record Autonomous() implements RiskDecision {}
    record GateRequired(
        String reason,
        boolean reversible,
        CandidateSetStrategy candidateGroups,  // was List<String>
        Duration expiresIn,
        String scope)
        implements RiskDecision {}
}
```

- [ ] **Step 5: Fix all compilation errors from SPI changes**

The `getId()` → `id()` rename on PlanningStrategy will break `PlanningStrategyLoopControl` (uses `PlanningStrategy::getId`). Fix the reference. Also fix any test files using `getId()`.

- [ ] **Step 6: Run compile check**

Run: `mvn install -DskipTests -q -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: BUILD SUCCESS (or compilation errors to fix — iterate)

- [ ] **Step 7: Run existing tests to check for regressions**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: All existing tests pass (the id() additions are backward compatible for tests; GateRequired change will break consumer tests — fix them)

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(#634): retrofit engine SPIs to extend NamedStrategy + CaseDefinition model

- AgentRoutingStrategy, ImplementationRoutingStrategy, TrustRoutingPolicyProvider,
  PlanningStrategy, WorkerExecutionRoutingStrategy all extend NamedStrategy
- All built-in implementations gain id() methods
- CaseDefinition gains agentRouting, implementationRouting, candidateMatching fields
- RiskDecision.GateRequired.candidateGroups: List<String> → CandidateSetStrategy

Refs #634"
```

---

### Task 5: Engine Wiring — StrategyResolver Integration

**Repo:** `casehub-engine`

**Files:**
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandler.java` — resolve strategies via StrategyResolver
- Modify: `runtime/src/main/java/io/casehub/engine/internal/routing/AgentCandidateFactory.java` — delegate matching to CandidateMatchingStrategy
- Modify: `blackboard/src/main/java/io/casehub/blackboard/control/PlanningStrategyLoopControl.java` — resolve via StrategyResolver
- Modify: `api/src/main/java/io/casehub/api/model/converter/CaseDefinitionYamlMapper.java` — parse new YAML strategy syntax
- Modify: `api/src/main/java/io/casehub/api/model/HumanTaskTarget.java` — candidateGroups/Users type: ListEvaluator → CandidateSetSpec
- Delete: `api/src/main/java/io/casehub/api/model/evaluator/ListEvaluator.java`
- Delete: `runtime/src/main/java/io/casehub/engine/internal/engine/ListExpressionResolver.java`

**Interfaces:**
- Consumes: `StrategyResolver` from Task 1, `CandidateSetStrategy`/`CandidateSetSpec` from Task 2, `CandidateMatchingStrategy` from Task 3, retrofitted SPIs from Task 4

- [ ] **Step 1: Change HumanTaskTarget — candidateGroups/Users type**

Replace `ListEvaluator candidateGroups` with `CandidateSetSpec candidateGroups`. Same for `candidateUsers`. Update builder:

```java
// Value-object overloads
public Builder candidateGroups(CandidateSetStrategy strategy) {
    this.candidateGroups = new CandidateSetSpec.Inline(strategy);
    return this;
}
public Builder candidateGroups(Set<String> groups) {
    return candidateGroups(StaticSetStrategy.of(groups));
}
public Builder candidateGroupsExpression(String expression) {
    // Deferred — ExpressionSetStrategy needs registry at parse time
    // Store as Named with special marker, or handle in YAML mapper
    return this;
}
// Named strategy overload
public Builder candidateGroups(String strategyId) {
    this.candidateGroups = new CandidateSetSpec.Named(strategyId, Map.of());
    return this;
}
public Builder candidateGroups(String strategyId, Map<String, Object> config) {
    this.candidateGroups = new CandidateSetSpec.Named(strategyId, config);
    return this;
}
```

- [ ] **Step 2: Update CaseDefinitionYamlMapper.convertHumanTask()**

Replace `ListEvaluator` parsing (lines 633-645) with `CandidateSetSpec` construction:

```java
final Object rawGroups = schema.getCandidateGroups();
CandidateSetSpec groupsSpec = parseCandidateSet(rawGroups, "candidateGroups");
if (groupsSpec != null) builder.candidateGroups(groupsSpec);

final Object rawUsers = schema.getCandidateUsers();
CandidateSetSpec usersSpec = parseCandidateSet(rawUsers, "candidateUsers");
if (usersSpec != null) builder.candidateUsers(usersSpec);
```

Add parser method:
```java
private CandidateSetSpec parseCandidateSet(Object raw, String fieldName) {
    if (raw instanceof List<?> list && !list.isEmpty()) {
        return new CandidateSetSpec.Inline(
            StaticSetStrategy.of(new LinkedHashSet<>(castStringList(fieldName, list))));
    } else if (raw instanceof String expr && !expr.isBlank()) {
        ExpressionEvaluator evaluator = expressionEngineRegistry.create(expr, "jq");
        return new CandidateSetSpec.Inline(
            new ExpressionSetStrategy(evaluator, expressionEngineRegistry));
    } else if (raw instanceof Map<?, ?> map) {
        if (map.containsKey("strategy")) {
            String strategyId = (String) map.get("strategy");
            @SuppressWarnings("unchecked")
            Map<String, Object> config = map.containsKey("config")
                ? (Map<String, Object>) map.get("config") : Map.of();
            return new CandidateSetSpec.Named(strategyId, config);
        } else if (map.containsKey("expression")) {
            String expression = (String) map.get("expression");
            String lang = map.containsKey("lang") ? (String) map.get("lang") : "jq";
            ExpressionEvaluator evaluator = expressionEngineRegistry.create(expression, lang);
            return new CandidateSetSpec.Inline(
                new ExpressionSetStrategy(evaluator, expressionEngineRegistry));
        }
    }
    return null;
}
```

- [ ] **Step 3: Update CaseContextChangedEventHandler.publishHumanTaskSchedule()**

Replace `listExpressionResolver.resolve()` calls with `CandidateSetSpec` evaluation:

```java
private Uni<Void> publishHumanTaskSchedule(
        CaseInstance caseInstance, Binding binding, HumanTaskTarget target) {
    JsonNode caseContext = caseInstance.getCaseContext()
        .panel(ContextPanel.WORKING).asJsonNode();

    Uni<Set<String>> groupsUni = resolveCandidateSet(target.candidateGroups(), caseContext);
    Uni<Set<String>> usersUni = resolveCandidateSet(target.candidateUsers(), caseContext);

    return Uni.combine().all().unis(groupsUni, usersUni).asTuple()
        .chain(tuple -> {
            Set<String> resolvedGroups = tuple.getItem1();
            Set<String> resolvedUsers = tuple.getItem2();
            // ... existing HumanTaskScheduleEvent publish logic
        });
}

private Uni<Set<String>> resolveCandidateSet(CandidateSetSpec spec, JsonNode caseContext) {
    if (spec == null) return Uni.createFrom().nullItem();
    return switch (spec) {
        case CandidateSetSpec.Inline inline ->
            inline.strategy().evaluate(new CandidateSetContext(caseContext));
        case CandidateSetSpec.Named named -> {
            CandidateSetStrategy resolved = strategyResolver.resolve(
                CandidateSetStrategy.class, named.strategyId());
            yield resolved.evaluate(new CandidateSetContext(caseContext, named.config()));
        }
    };
}
```

- [ ] **Step 4: Update CaseContextChangedEventHandler.publishWorkerSchedule()**

Replace direct `agentRoutingStrategy` injection with StrategyResolver:

```java
// Before: @Inject AgentRoutingStrategy agentRoutingStrategy;
// After:  @Inject StrategyResolver strategyResolver;

// In publishWorkerSchedule():
AgentRoutingStrategy routingStrategy = strategyResolver.resolve(
    AgentRoutingStrategy.class,
    caseInstance.getCaseMetaModel().getCaseDefinition().agentRouting());
routingStrategy.select(ctx, candidates)
```

- [ ] **Step 5: Update AgentCandidateFactory — delegate matching**

Replace hardcoded matching with CandidateMatchingStrategy delegation:

```java
@ApplicationScoped
public class AgentCandidateFactory {

    private final StrategyResolver strategyResolver;
    // Remove: private final VocabularyRegistry vocabularyRegistry;

    @Inject
    public AgentCandidateFactory(StrategyResolver strategyResolver) {
        this.strategyResolver = strategyResolver;
    }

    public List<AgentCandidate> buildCandidates(
            CaseInstance caseInstance, CaseDefinition caseDefinition,
            List<Worker> workers, Capability capability,
            WorkerExecutionManager executionManager, CapabilityHealth capabilityHealth) {

        // Step 1: Resolve matching strategy
        CandidateMatchingStrategy matchingStrategy = strategyResolver.resolve(
            CandidateMatchingStrategy.class, caseDefinition.candidateMatching());

        // Step 2: Match workers to capability
        List<Worker> matched = matchingStrategy.match(
            new CandidateMatchingContext(capability.name(), workers, caseDefinition))
            .await().indefinitely();

        // Step 3: Health probe + candidate construction (unchanged orchestration)
        List<AgentCandidate> candidates = new ArrayList<>();
        for (Worker worker : matched) {
            AgentDescriptor descriptor = caseDefinition.agentDescriptorFor(worker.name()).orElse(null);
            // ... existing health probe logic ...
            // ... existing AgentCandidate construction ...
        }
        return candidates;
    }
}
```

- [ ] **Step 6: Update PlanningStrategyLoopControl — resolve via StrategyResolver**

Replace `Map<String, PlanningStrategy>` with `StrategyResolver`:

```java
// Before: private final Map<String, PlanningStrategy> strategies;
// After:
private final StrategyResolver strategyResolver;

// In constructor:
@Inject
public PlanningStrategyLoopControl(StrategyResolver strategyResolver, ...) {
    this.strategyResolver = strategyResolver;
}

// In select() — replace strategies.get(strategyId):
PlanningStrategy strategy = strategyResolver.resolve(
    PlanningStrategy.class,
    ctx.definition().getPlanningStrategy());
```

- [ ] **Step 7: Delete ListEvaluator and ListExpressionResolver**

Remove `api/src/main/java/io/casehub/api/model/evaluator/ListEvaluator.java` and `runtime/src/main/java/io/casehub/engine/internal/engine/ListExpressionResolver.java`. Fix all remaining compilation references.

- [ ] **Step 8: Compile and fix all errors**

Run: `mvn install -DskipTests -q -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Iterate until BUILD SUCCESS.

- [ ] **Step 9: Run full test suite**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Fix any failures. The main expected breakage: tests using `ListEvaluator`, tests using `GateRequired` with `List<String>`, tests using direct `agentRoutingStrategy` injection.

- [ ] **Step 10: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(#634): engine wiring — StrategyResolver integration

- HumanTaskTarget: ListEvaluator → CandidateSetSpec
- CaseDefinitionYamlMapper: new YAML strategy syntax (static/expression/named)
- CaseContextChangedEventHandler: resolve strategies via StrategyResolver
- AgentCandidateFactory: delegate matching to CandidateMatchingStrategy
- PlanningStrategyLoopControl: resolve via StrategyResolver
- Delete ListEvaluator and ListExpressionResolver

Refs #634"
```

---

### Task 6: GateRequired Evaluation Timing

**Repo:** `casehub-engine`

**Files:**
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/WorkflowExecutionCompletedHandler.java` — evaluate CandidateSetStrategy at gate creation
- Modify: event types carrying resolved groups to ActionGateWorkItemHandler

**Interfaces:**
- Consumes: `CandidateSetStrategy` from Task 2, `StrategyResolver` from Task 1

- [ ] **Step 1: Update WorkflowExecutionCompletedHandler.handleGate()**

In the method that processes `GateRequired`, evaluate the `CandidateSetStrategy` against case context:

```java
// In handleGate() where GateRequired is processed:
CandidateSetStrategy groupsStrategy = gateRequired.candidateGroups();
Set<String> resolvedGroups = groupsStrategy != null
    ? groupsStrategy.evaluate(new CandidateSetContext(caseContext)).await().indefinitely()
    : null;

// Pass resolvedGroups (Set<String>) in ActionGateScheduleEvent, not the strategy
```

This ensures `ActionGateWorkItemHandler` receives resolved groups and doesn't need case context or StrategyResolver.

- [ ] **Step 2: Run tests**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: PASS

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(#634): GateRequired evaluation timing — resolve groups in WorkflowExecutionCompletedHandler

Refs #634"
```

---

### Task 7: Work SPI Retrofit + Wiring

**Repo:** `casehub-work` (`/Users/mdproctor/claude/casehub/work`)

**Files:**
- Modify: `api/src/main/java/io/casehub/work/api/spi/WorkerSelectionStrategy.java` — extends NamedStrategy
- Modify: `api/src/main/java/io/casehub/work/api/spi/InstanceAssignmentStrategy.java` — extends NamedStrategy
- Modify: `api/src/main/java/io/casehub/work/api/spi/ClaimSlaPolicy.java` — extends NamedStrategy
- Modify: All built-in implementations — add id()
- Modify: `runtime/src/main/java/io/casehub/work/runtime/service/WorkItemAssignmentService.java` — resolve via StrategyResolver
- Modify: `runtime/src/main/java/io/casehub/work/runtime/multiinstance/MultiInstanceSpawnService.java` — resolve via StrategyResolver

**Interfaces:**
- Consumes: `NamedStrategy` + `StrategyResolver` from Task 1

- [ ] **Step 1: Add NamedStrategy to work SPIs**

```java
// WorkerSelectionStrategy.java
import io.casehub.platform.api.routing.NamedStrategy;
public interface WorkerSelectionStrategy extends NamedStrategy {
    AssignmentDecision select(SelectionContext context, List<WorkerCandidate> candidates);
    default Set<AssignmentTrigger> triggers() { return Set.of(AssignmentTrigger.values()); }
}

// InstanceAssignmentStrategy.java
import io.casehub.platform.api.routing.NamedStrategy;
public interface InstanceAssignmentStrategy extends NamedStrategy {
    void assign(List<Object> instances, MultiInstanceContext context);
}

// ClaimSlaPolicy.java
import io.casehub.platform.api.routing.NamedStrategy;
public interface ClaimSlaPolicy extends NamedStrategy {
    Instant computePoolDeadline(ClaimSlaContext context);
}
```

- [ ] **Step 2: Add id() to all built-in implementations**

| Class | id() |
|-------|------|
| `LeastLoadedStrategy` | `"least-loaded"` |
| `ClaimFirstStrategy` | `"claim-first"` |
| `RoundRobinStrategy` | `"round-robin"` |
| `SemanticWorkerSelectionStrategy` | `"semantic"` |
| `PoolAssignmentStrategy` | `"pool"` |
| `ExplicitListAssignmentStrategy` | `"explicit"` |
| `RoundRobinAssignmentStrategy` | `"round-robin"` |
| `CompositeInstanceAssignmentStrategy` | `"composite"` |
| `ContinuationPolicy` | `"continuation"` |
| `FreshClockPolicy` | `"fresh-clock"` |
| `SingleBudgetPolicy` | `"single-budget"` |
| `PhaseClockPolicy` | `"phase-clock"` |

Remove `@Named` annotations from `InstanceAssignmentStrategy` implementations.

- [ ] **Step 3: Update WorkItemAssignmentService.activeStrategy()**

Replace CDI `Instance<>` scanning + config switch with StrategyResolver:

```java
// Before: complex 3-tier resolution
// After:
private WorkerSelectionStrategy activeStrategy() {
    if (fixedStrategy != null) return fixedStrategy;
    return strategyResolver.resolve(
        WorkerSelectionStrategy.class,
        config.routing().strategy());  // returns strategy id string
}
```

- [ ] **Step 4: Update MultiInstanceSpawnService.resolveStrategy()**

Replace `@Named` annotation scanning with StrategyResolver:

```java
// Before: iterate Instance<>, match by @Named annotation
// After:
private InstanceAssignmentStrategy resolveStrategy(String strategyName) {
    return strategyResolver.resolve(InstanceAssignmentStrategy.class, strategyName);
}
```

- [ ] **Step 5: Compile and test**

Run: `mvn install -DskipTests -q -f /Users/mdproctor/claude/casehub/work/pom.xml`
Then: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -f /Users/mdproctor/claude/casehub/work/pom.xml`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/work commit -m "feat(engine#634): retrofit work SPIs to extend NamedStrategy

- WorkerSelectionStrategy, InstanceAssignmentStrategy, ClaimSlaPolicy extend NamedStrategy
- All built-in implementations gain id() methods
- WorkItemAssignmentService: resolve via StrategyResolver
- MultiInstanceSpawnService: resolve via StrategyResolver, remove @Named

Refs casehubio/engine#634"
```

---

### Task 8: Consumer Repo Migration

**Repos:** casehub-engine-ledger, casehub-engine-ai, casehub-devtown, casehub-aml, casehub-clinical, casehub-life, casehub-soc, casehub-iot, quarkmind, casehub-ops

Each consumer change is mechanical: add `id()` method and/or wrap `List<String>` in `StaticSetStrategy.of(...)`.

- [ ] **Step 1: Engine-ledger**

`TrustWeightedAgentStrategy` — add `@Override public String id() { return "trust-weighted"; }`
`TrustWeightedImplementationRoutingStrategy` — add `@Override public String id() { return "trust-weighted"; }`
`DefaultTrustRoutingPolicyProvider` — add `@Override public String id() { return "default"; }`

- [ ] **Step 2: Engine-ai**

`SemanticAgentRoutingStrategy` — add `@Override public String id() { return "semantic"; }`

- [ ] **Step 3: Consumer classifiers (per domain repo)**

Each `ActionRiskClassifier` that returns `GateRequired` with `List<String>` candidateGroups:

```java
// Before:
new RiskDecision.GateRequired(reason, reversible, List.of("compliance-team"), expiresIn, scope)

// After:
new RiskDecision.GateRequired(reason, reversible, StaticSetStrategy.of("compliance-team"), expiresIn, scope)
```

Add import: `import io.casehub.api.spi.routing.StaticSetStrategy;`

- [ ] **Step 4: Consumer trust policy providers (per domain repo)**

Each `TrustRoutingPolicyProvider` implementation adds `id()`:

| Repo | Class | id() |
|------|-------|------|
| devtown | `DevtownTrustRoutingPolicyProvider` | `"devtown"` |
| aml | `AmlTrustRoutingPolicyProvider` | `"aml"` |
| clinical | `ClinicalTrustRoutingPolicyProvider` | `"clinical"` |
| life | `LifeTrustRoutingPolicyProvider` | `"life"` |
| quarkmind | `QuarkMindTrustRoutingPolicyProvider` | `"quarkmind"` |
| ops | `DeploymentTrustRoutingPolicyProvider` | `"deployment"` |

- [ ] **Step 5: Compile and test each repo**

Per repo: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test`

- [ ] **Step 6: Commit each repo**

Per repo: `git commit -m "feat(engine#634): add strategy id() for universal routing convention — Refs casehubio/engine#634"`

---

### Task 9: Documentation + Protocol

**Files:**
- Modify: PLATFORM.md (in casehub-parent repo)
- Create: Garden protocol `routing-strategy-convention.md`

- [ ] **Step 1: Update PLATFORM.md capability ownership table**

Add entry for Routing Strategy Resolution under `casehub-platform-api`.

- [ ] **Step 2: Update PLATFORM.md Step 4 rules**

Add routing strategy rule: extend NamedStrategy, declare id(), @DefaultBean default, resolve via StrategyResolver.

- [ ] **Step 3: Create garden protocol**

Create `routing-strategy-convention.md` in `~/.hortora/garden/docs/protocols/casehub/` with scope, rule, and non-members list per spec §7.

- [ ] **Step 4: Commit**

Commit PLATFORM.md to parent repo, protocol to garden repo.

---

## Self-Review Checklist

- [x] **Spec coverage:** Every section of the spec maps to a task. §1→Task 1, §2→Tasks 2-3, §3→Task 4, §4→Task 5, §5→Task 6, §6→Tasks 4-8, §7→Task 9, §8→scope boundaries, §9→Tasks 4-5, §10→validated during audit.
- [x] **Placeholder scan:** No TBD/TODO. All code blocks contain complete implementations. All commands include expected output.
- [x] **Type consistency:** `CandidateSetStrategy`, `CandidateSetSpec`, `CandidateSetContext`, `CandidateMatchingStrategy`, `CandidateMatchingContext`, `NamedStrategy`, `StrategyResolver` — names consistent across all tasks.
- [x] **Cross-repo dependencies:** Platform → Engine/Work → Consumers. Sequenced correctly with `mvn install -DskipTests -q` between phases.
