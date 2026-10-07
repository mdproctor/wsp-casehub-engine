# Fix CI: Spring Bean Conflicts, @PlatformStream Restoration, Flaky Test — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #1222 — fix(ci): Spring bean conflicts and generator gaps blocking CI
**Issue group:** #1222

**Goal:** Make CI green by removing duplicate Spring beans, restoring @PlatformStream annotations, fixing stale exclude paths, and eliminating a flaky test.

**Architecture:** Four independent fixes targeting different modules: runtime-spring (bean removal), api (annotation restoration), spring-integration-test (config correction), runtime (test isolation). No cross-task dependencies.

**Tech Stack:** Java 21, Spring Boot 3, Quarkus 3.32.2, Maven

## Global Constraints

- No timeout increases for flaky tests — fix root cause
- Use `ide_edit_member` / `ide_replace_member` for Java structural edits
- All `mvn test` commands require `TESTCONTAINERS_RYUK_DISABLED=true` prefix
- Always `mvn install -DskipTests -q` before module-specific test runs

---

## Batch 1: Spring and API fixes

### Task 1: Remove crossTenant passthrough beans from RuntimeManualConfig

**Files:**
- Modify: `runtime-spring/src/main/java/io/casehub/engine/runtime/spring/RuntimeManualConfig.java:403-414`

**Interfaces:**
- Consumes: nothing
- Produces: RuntimeManualConfig no longer defines `crossTenantEventLogRepository` or `crossTenantCaseInstanceRepository` beans — PersistenceAutoConfiguration remains the sole provider

- [ ] **Step 1: Delete the two passthrough @Bean methods**

Use `ide_edit_member` or Edit tool to remove lines 403-414 from RuntimeManualConfig.java. The two methods to remove:

```java
  @Bean
  public io.casehub.engine.common.spi.CrossTenantEventLogRepository crossTenantEventLogRepository(
      io.casehub.engine.common.spi.CrossTenantEventLogRepository repo) {
    return repo;
  }

  @Bean
  public io.casehub.engine.common.spi.CrossTenantCaseInstanceRepository
      crossTenantCaseInstanceRepository(
          io.casehub.engine.common.spi.CrossTenantCaseInstanceRepository repo) {
    return repo;
  }
```

After removal, the file should end with the closing brace of `yamlObjectMapper()` method followed by the class closing brace.

- [ ] **Step 2: Fix stale exclude paths in spring-integration-test**

Modify `spring-integration-test/src/test/resources/application.properties` lines 7-8.

Change:
```properties
  io.casehub.engine.internal.spring.RuntimeManualConfig,\
  io.casehub.engine.internal.spring.RuntimeAutoConfiguration,\
```

To:
```properties
  io.casehub.engine.runtime.spring.RuntimeManualConfig,\
```

Remove the `RuntimeAutoConfiguration` line entirely (class doesn't exist).

- [ ] **Step 3: Build and run spring-integration-test**

```bash
mvn install -DskipTests -q
```

Then:

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl spring-integration-test
```

Expected: All tests pass, including `contextLoads()` — no more `BeanDefinitionOverrideException`.

- [ ] **Step 4: Commit**

```bash
git add runtime-spring/src/main/java/io/casehub/engine/runtime/spring/RuntimeManualConfig.java spring-integration-test/src/test/resources/application.properties
git commit -m "$(cat <<'EOF'
fix(#1222): remove crossTenant passthrough beans and fix stale exclude paths

Remove identity-passthrough crossTenantEventLogRepository and
crossTenantCaseInstanceRepository from RuntimeManualConfig — they
served a Quarkus @CrossTenant qualifier that Spring doesn't have,
and conflicted with the real JPA beans in PersistenceAutoConfiguration.

Fix spring-integration-test exclude paths: io.casehub.engine.internal.spring
→ io.casehub.engine.runtime.spring. Remove non-existent
RuntimeAutoConfiguration exclude.

Refs #1222

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

### Task 2: Restore @PlatformStream annotations

**Files:**
- Modify: `api/src/main/java/io/casehub/api/engine/rest/EngineCaseApi.java:72-73`
- Modify: `api/src/main/java/io/casehub/api/engine/rest/EnginePlanApi.java:46-47`

**Interfaces:**
- Consumes: nothing
- Produces: SSE streaming methods annotated with @PlatformStream — generators will produce REST resource endpoints for them

- [ ] **Step 1: Restore @PlatformStream on EngineCaseApi.caseStream()**

In `api/src/main/java/io/casehub/api/engine/rest/EngineCaseApi.java`:

Add import (after existing `PlatformQuery` import, line 31):
```java
import io.casehub.platform.api.mcp.PlatformStream;
```

Replace the comment and method (lines 72-73):
```java
  // SSE streaming — excluded from annotation processing (Flow.Publisher not yet supported)
  Flow.Publisher<CaseStreamEventView> caseStream(@PathParam UUID caseId);
```

With:
```java
  @PlatformStream("Live case event stream")
  Flow.Publisher<CaseStreamEventView> caseStream(@PathParam UUID caseId);
```

- [ ] **Step 2: Restore @PlatformStream on EnginePlanApi.executionStateStream()**

In `api/src/main/java/io/casehub/api/engine/rest/EnginePlanApi.java`:

Add import (after existing `PlatformQuery` import, line 21):
```java
import io.casehub.platform.api.mcp.PlatformStream;
```

Replace the comment and method (lines 46-47):
```java
  // SSE streaming — excluded from annotation processing (Flow.Publisher not yet supported)
  Flow.Publisher<JsonNode> executionStateStream(@PathParam UUID caseId);
```

With:
```java
  @PlatformStream("Live execution state updates")
  Flow.Publisher<JsonNode> executionStateStream(@PathParam UUID caseId);
```

- [ ] **Step 3: Compile api module to verify annotation processing**

```bash
mvn compile -pl api
```

Expected: Compiles without errors.

- [ ] **Step 4: Commit**

```bash
git add api/src/main/java/io/casehub/api/engine/rest/EngineCaseApi.java api/src/main/java/io/casehub/api/engine/rest/EnginePlanApi.java
git commit -m "$(cat <<'EOF'
fix(#1222): restore @PlatformStream annotations on SSE endpoints

Platform generators (#1214) now handle Flow.Publisher return types.
Restore @PlatformStream removed by 3a77d000f so SSE endpoints are
auto-generated into REST resources.

Refs #1222

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

## Batch 2: Flaky test fix and full verification

### Task 3: Fix HybridOrchestrationIntegrationTest flakiness

**Files:**
- Modify: `runtime/src/test/java/io/casehub/engine/HybridOrchestrationIntegrationTest.java:47-53`

**Interfaces:**
- Consumes: nothing
- Produces: deterministic test isolation via @BeforeEach cache cleanup

- [ ] **Step 1: Add @BeforeEach cache cleanup**

In `runtime/src/test/java/io/casehub/engine/HybridOrchestrationIntegrationTest.java`, add after the field declarations (after line 52):

```java
  @org.junit.jupiter.api.BeforeEach
  void setUp() {
    cache.clear();
  }
```

The import for `org.junit.jupiter.api.BeforeEach` is not needed since it's used fully qualified inline. Alternatively, add it to the imports and use `@BeforeEach` — match whichever style the file uses. The file already imports `org.junit.jupiter.api.Test`, so add the import:

```java
import org.junit.jupiter.api.BeforeEach;
```

And use:

```java
  @BeforeEach
  void setUp() {
    cache.clear();
  }
```

- [ ] **Step 2: Run the full runtime module tests**

```bash
mvn install -DskipTests -q
TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl runtime
```

Expected: All tests pass, including `HybridOrchestrationIntegrationTest`. Run the full module (not just the test class) to verify the cleanup doesn't break other tests and that the flaky test passes in the presence of other tests.

- [ ] **Step 3: Commit**

```bash
git add runtime/src/test/java/io/casehub/engine/HybridOrchestrationIntegrationTest.java
git commit -m "$(cat <<'EOF'
fix(#1222): add cache cleanup to HybridOrchestrationIntegrationTest

Add @BeforeEach cache.clear() matching the pattern in 6 other test
classes. The shared CaseInstanceCache accumulated stale case instances
from prior tests, causing Vert.x event bus contention and intermittent
spawnAndAwaitCase timeouts.

Closes #1222

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

## References

- [2026-10-06-fix-ci-spring-bean-conflicts-design.md] — design spec this plan implements
- RuntimeManualConfig.java:403-414 — passthrough beans to remove
- PersistenceAutoConfiguration.java:86-95 — real JPA implementations
- EngineCaseApi.java:72-73 — @PlatformStream restoration target
- EnginePlanApi.java:46-47 — @PlatformStream restoration target
- application.properties:7-8 — stale exclude paths
- HybridOrchestrationIntegrationTest.java:47-53 — flaky test
- OrchestrationTest.java:58-61 — established @BeforeEach cache cleanup pattern
- 3a77d000f — git diff showing exact original @PlatformStream values
- GitHub #1222 — focal issue
- GitHub #1214 — platform generator Flow.Publisher support
