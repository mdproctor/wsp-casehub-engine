# Case Auto-Discovery — Full-Pipeline YAML Registration

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #1331 — Case engine auto-discovery: scan classpath YAML and register CaseHub beans
**Issue group:** #1331

**Goal:** Enhance `CaseYamlRegistrar` to use the full YAML loading pipeline so `casehub/*.yaml` on the classpath produces fully-wired case definitions (expression engines, worker function providers, agent providers, overlay composition) without requiring any Java code.

**Architecture:** Inject full pipeline dependencies into the existing `CaseYamlRegistrar`. In `register()`, read YAML as `JsonNode`, check for convention overlay, merge if found, and call `CaseDefinitionYamlMapper.load(JsonNode, ...)` instead of `load(InputStream)`. Precedence of explicit CaseHub beans is handled by startup priority ordering (CDI beans at priority 10, registrar at priority 200) plus registry dedup.

**Tech Stack:** Quarkus CDI, Jackson (YAML), CaseHub platform bootstrap SPI

## Global Constraints

- Quarkus 3.32.2 — no APIs from later versions
- All changes in the `runtime` module — no new module creation
- `@ApplicationScoped` only (not `@DefaultBean`) per protocol PP-20260921-b7c277
- Error handling: log and skip, never fail-fast on individual YAML files

---

## Batch 1: Full-Pipeline Auto-Discovery

### Task 1: Make YamlCaseHub.deriveConventionPath() public

**Files:**
- Modify: `api/src/main/java/io/casehub/api/engine/YamlCaseHub.java:143`
- Modify: `api/src/test/java/io/casehub/api/engine/YamlCaseHubTest.java` (verify existing tests still pass)

**Interfaces:**
- Produces: `public static String deriveConventionPath(String basePath)` — called by Task 2

- [ ] **Step 1: Verify existing test coverage for deriveConventionPath**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=YamlCaseHubTest -q`
Expected: PASS — existing tests cover the convention path logic

- [ ] **Step 2: Change visibility from package-private to public**

In `api/src/main/java/io/casehub/api/engine/YamlCaseHub.java`, change line 143:

```java
// Before:
static String deriveConventionPath(String basePath) {

// After:
public static String deriveConventionPath(String basePath) {
```

Use `ide_replace_member` to change the method signature.

- [ ] **Step 3: Run api module tests**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=YamlCaseHubTest -q`
Expected: PASS — visibility change is source-compatible

- [ ] **Step 4: Commit**

```bash
git add api/src/main/java/io/casehub/api/engine/YamlCaseHub.java
git commit -m "refactor(#1331): make YamlCaseHub.deriveConventionPath() public

Needed by CaseYamlRegistrar to share the overlay convention path logic.

Refs #1331

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

### Task 2: Enhance CaseYamlRegistrar with full pipeline loading and overlay support

**Files:**
- Modify: `runtime/src/main/java/io/casehub/engine/runtime/bootstrap/CaseYamlRegistrar.java`
- Modify: `runtime/src/test/java/io/casehub/engine/runtime/bootstrap/CaseYamlRegistrarTest.java`

**Interfaces:**
- Consumes: `YamlCaseHub.deriveConventionPath(String)` from Task 1
- Consumes: `CaseDefinitionYamlMapper.load(JsonNode, ObjectMapper, ExpressionEngineRegistry, WorkerFunctionProviderRegistry, AgentProvider)` — existing static method
- Consumes: `YamlMerger.merge(JsonNode, JsonNode)` — existing static method from platform-api

- [ ] **Step 1: Write failing test — full pipeline load produces wired definition**

Replace the existing test class with an enhanced version. The key change: use a real YAML ObjectMapper (from `YamlMappers.create()`) and verify the registrar calls the full pipeline. Set `expressionEngineRegistry` and `workerFunctionProviderRegistry` to null — `CaseDefinitionYamlMapper.load()` falls back to JQ_ONLY/EMPTY_PROVIDERS internally.

In `runtime/src/test/java/io/casehub/engine/runtime/bootstrap/CaseYamlRegistrarTest.java`:

```java
package io.casehub.engine.runtime.bootstrap;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.never;
import static org.mockito.Mockito.verify;

import io.casehub.api.model.CaseDefinition;
import io.casehub.engine.common.spi.CaseDefinitionRegistry;
import io.casehub.yaml.jackson.YamlMappers;
import java.io.ByteArrayInputStream;
import java.nio.charset.StandardCharsets;
import java.util.Set;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.ArgumentCaptor;

class CaseYamlRegistrarTest {

  CaseYamlRegistrar registrar;
  CaseDefinitionRegistry caseDefinitionRegistry;

  @BeforeEach
  void setUp() {
    registrar = new CaseYamlRegistrar();
    caseDefinitionRegistry = mock(CaseDefinitionRegistry.class);
    registrar.caseDefinitionRegistry = caseDefinitionRegistry;
    registrar.objectMapper = YamlMappers.create();
  }

  @Test
  void sectionKey() {
    assertThat(registrar.sectionKey()).isEqualTo("cases");
  }

  @Test
  void classpathPaths() {
    assertThat(registrar.classpathPaths()).isEqualTo(Set.of("casehub", "cases"));
  }

  @Test
  void priority() {
    assertThat(registrar.priority()).isEqualTo(200);
  }

  @Test
  void register_validCaseYaml_registersDefinition() {
    String yaml =
        """
        dsl: "0.1.0"
        version: "1.0.0"
        name: test-case
        namespace: test
        spec:
          capabilities:
            - name: process
              inputProjection: "."
              outputProjection: "."
          bindings:
            - name: trigger
              capability: process
              on:
                contextChange:
                  filter: '.ready == true'
        """;

    registrar.register("casehub/test-case.yaml", stream(yaml));

    var captor = ArgumentCaptor.forClass(CaseDefinition.class);
    verify(caseDefinitionRegistry).registerCaseDefinition(captor.capture());

    CaseDefinition def = captor.getValue();
    assertThat(def.getName()).isEqualTo("test-case");
    assertThat(def.getNamespace()).isEqualTo("test");
    assertThat(def.getVersion()).isEqualTo("1.0.0");
  }

  @Test
  void register_invalidYaml_doesNotCrash() {
    String yaml = "not: a: valid: case: definition:\n  - broken";

    registrar.register("casehub/broken.yaml", stream(yaml));

    verify(caseDefinitionRegistry, never())
        .registerCaseDefinition(org.mockito.ArgumentMatchers.any());
  }

  private static ByteArrayInputStream stream(String content) {
    return new ByteArrayInputStream(content.getBytes(StandardCharsets.UTF_8));
  }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest=CaseYamlRegistrarTest -q`
Expected: FAIL — `registrar.objectMapper` field does not exist yet on `CaseYamlRegistrar`

- [ ] **Step 3: Implement CaseYamlRegistrar enhancement**

Replace the full class body of `runtime/src/main/java/io/casehub/engine/runtime/bootstrap/CaseYamlRegistrar.java`:

```java
package io.casehub.engine.runtime.bootstrap;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.api.engine.ExpressionEngineRegistry;
import io.casehub.api.engine.YamlCaseHub;
import io.casehub.api.marshaller.YamlMapper;
import io.casehub.api.model.CaseDefinition;
import io.casehub.api.model.converter.CaseDefinitionYamlMapper;
import io.casehub.api.spi.WorkerFunctionProviderRegistry;
import io.casehub.engine.common.spi.CaseDefinitionRegistry;
import io.casehub.platform.api.yaml.YamlMerger;
import io.casehub.platform.bootstrap.spi.ClasspathYamlRegistrar;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;
import java.io.IOException;
import java.io.InputStream;
import java.util.Set;
import org.jboss.logging.Logger;

@ApplicationScoped
public class CaseYamlRegistrar implements ClasspathYamlRegistrar {

  private static final Logger LOG = Logger.getLogger(CaseYamlRegistrar.class);

  @Inject CaseDefinitionRegistry caseDefinitionRegistry;
  @Inject ExpressionEngineRegistry expressionEngineRegistry;
  @Inject @YamlMapper ObjectMapper objectMapper;
  @Inject WorkerFunctionProviderRegistry workerFunctionProviderRegistry;
  @Inject Instance<io.casehub.platform.agent.AgentProvider> agentProviderInstance;

  @Override
  public String sectionKey() {
    return "cases";
  }

  @Override
  public Set<String> classpathPaths() {
    return Set.of("casehub", "cases");
  }

  @Override
  public int priority() {
    return 200;
  }

  @Override
  public void register(String resourcePath, InputStream content) {
    try {
      JsonNode base = objectMapper.readTree(content);
      JsonNode overlay = resolveOverlay(resourcePath);
      JsonNode merged = (overlay != null) ? YamlMerger.merge(base, overlay) : base;

      CaseDefinition definition =
          CaseDefinitionYamlMapper.load(
              merged,
              objectMapper,
              expressionEngineRegistry,
              workerFunctionProviderRegistry,
              agentProviderInstance != null && agentProviderInstance.isResolvable()
                  ? agentProviderInstance.get()
                  : null);

      caseDefinitionRegistry.registerCaseDefinition(definition);
      LOG.debugf("Registered case definition from %s: %s", resourcePath, definition.getName());
    } catch (IOException e) {
      LOG.errorf(e, "Failed to load case definition from %s", resourcePath);
    } catch (RuntimeException e) {
      LOG.errorf(e, "Failed to register case definition from %s", resourcePath);
    }
  }

  private JsonNode resolveOverlay(String resourcePath) {
    String conventionPath = YamlCaseHub.deriveConventionPath(resourcePath);
    InputStream is =
        Thread.currentThread().getContextClassLoader().getResourceAsStream(conventionPath);
    if (is != null) {
      try {
        return objectMapper.readTree(is);
      } catch (IOException e) {
        LOG.warnf(e, "Failed to load overlay from %s", conventionPath);
        return null;
      } finally {
        try {
          is.close();
        } catch (IOException ignored) {
        }
      }
    }
    return null;
  }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest=CaseYamlRegistrarTest -q`
Expected: PASS — all 4 tests pass

- [ ] **Step 5: Run full runtime module tests**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -q`
Expected: PASS — no regressions

- [ ] **Step 6: Commit**

```bash
git add runtime/src/main/java/io/casehub/engine/runtime/bootstrap/CaseYamlRegistrar.java
git add runtime/src/test/java/io/casehub/engine/runtime/bootstrap/CaseYamlRegistrarTest.java
git commit -m "feat(#1331): enhance CaseYamlRegistrar with full pipeline loading

Inject ExpressionEngineRegistry, WorkerFunctionProviderRegistry,
@YamlMapper ObjectMapper, and AgentProvider. Use
CaseDefinitionYamlMapper.load(JsonNode, ...) for full worker/expression
wiring. Add overlay composition via convention path (-overrides suffix).

Closes #1331

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

## References

- [2026-11-19-case-auto-discovery-design.md] — design spec this plan implements
- [CaseYamlRegistrar.java:30] — existing registrar to enhance
- [YamlCaseHub.java:68] — full pipeline load pattern to follow
- [YamlCaseHub.java:143] — deriveConventionPath() to make public
- [CaseDefinitionYamlMapper.java:213] — load(JsonNode, ...) target method
- [DefaultCaseDefinitionRegistry.java:103] — priority 10 startup, dedup at line 157
- [CaseYamlRegistrarTest.java] — existing test to update
- [GitHub #1331] — focal issue
