# Fix CI: Spring Bean Conflicts, @PlatformStream Restoration, Flaky Test

**Issue:** engine#1222
**Scale:** S
**Date:** 2026-10-06

## Context

CI is red on main after landing #1215 (rest-core extraction) and #1218. Three problems surfaced; one was partially fixed by a quick-fix commit (3a77d000f) that removed @PlatformStream annotations. Since then, #1214 landed generator support for `Flow.Publisher`, making the removal a stale workaround that now prevents SSE endpoint generation.

## Fixes

### 1. Remove crossTenant passthrough beans from RuntimeManualConfig

**File:** `runtime-spring/src/main/java/.../RuntimeManualConfig.java` lines 403-414

Two identity-passthrough beans (`crossTenantEventLogRepository`, `crossTenantCaseInstanceRepository`) accept a bean and return it unchanged. They existed to satisfy a Quarkus `@CrossTenant` qualifier that Spring doesn't have. `PersistenceAutoConfiguration` defines the real JPA implementations with the same bean names — Spring Boot rejects the duplicate as `BeanDefinitionOverrideException`.

**Action:** Delete both `@Bean` methods (lines 403-414).

### 2. Restore @PlatformStream annotations

**Files:**
- `api/src/main/java/.../EngineCaseApi.java` line 72-73
- `api/src/main/java/.../EnginePlanApi.java` line 46-47

Commit 3a77d000f removed `@PlatformStream` as a workaround because generators couldn't handle `Flow.Publisher`. Platform generators (#1214) now handle this correctly. Without restoration, SSE endpoints won't be auto-generated into REST resources.

**Action:** Add `@PlatformStream("description")` annotation to both methods. Add import for `io.casehub.platform.api.mcp.PlatformStream`. Remove the stale "excluded from annotation processing" comments.

### 3. Fix stale exclude paths in spring-integration-test

**File:** `spring-integration-test/src/test/resources/application.properties` lines 7-8

Commit 250b4ae12 added excludes with wrong package paths:
- `io.casehub.engine.internal.spring.RuntimeManualConfig` → actual: `io.casehub.engine.runtime.spring.RuntimeManualConfig`
- `io.casehub.engine.internal.spring.RuntimeAutoConfiguration` → class doesn't exist

**Action:** Fix the RuntimeManualConfig path. Remove the RuntimeAutoConfiguration line.

### 4. Fix flaky HybridOrchestrationIntegrationTest

**File:** `runtime/src/test/java/.../HybridOrchestrationIntegrationTest.java`

The test has no `@BeforeEach` cleanup. `CaseInstanceCache` is an `@ApplicationScoped` `ConcurrentHashMap` singleton shared across all `@QuarkusTest` classes. Stale case instances from prior tests cause Vert.x event bus contention (all events dispatch async via `VertxEventDispatcher.publish()`), delaying evaluations through the `CaseEvaluationSerializer`. The `spawnAndAwaitCase` flow has a 10-second internal timeout — event bus delays from stale evaluations can hit it.

**Action:** Add `@BeforeEach void setUp() { cache.clear(); }` — matching the pattern in 6 other test classes (`OrchestrationTest`, `CaseWaitingResumeTest`, `WorkBrokerEndToEndTest`, `ContextChangeWhenFilterTest`, `ChoreographySelectionTest`, `SignalDedupExtendedTest`).

## Testing

- `mvn install -DskipTests -q` then `TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl spring-integration-test` — verifies Fix 1 and Fix 3
- `TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl runtime` — verifies Fix 4 (run full module, not just the test class)
- `mvn compile -pl api` — verifies Fix 2 compiles (annotation processing runs at compile)

## References

- RuntimeManualConfig:403-414 — passthrough beans to remove
- PersistenceAutoConfiguration:86-95 — real JPA implementations
- CaseInstanceCacheImpl:41-43 — clear() implementation
- OrchestrationTest:58-61 — established cleanup pattern
- VertxEventDispatcher:95-102 — async event dispatch mechanism
- CaseEvaluationSerializer:35-55 — per-case evaluation serialization with pending coalescing
- 3a77d000f — @PlatformStream removal commit
- #1214 — platform generator Flow.Publisher support
- 250b4ae12 — stale exclude paths commit
