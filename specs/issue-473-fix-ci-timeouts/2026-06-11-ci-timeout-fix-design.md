# Design: Fix CI ConditionTimeout Failures — In-Memory Test Profile for Behavior Tests

**Issue:** engine#473  
**Date:** 2026-06-11

---

## Problem

Five `@QuarkusTest` classes in the `runtime` module time out on CI runners:

- `SimpleCaseHubBeanTest`
- `YamlSimpleCaseHubBeanTest`
- `MultipleCaseInstancesTest`
- `CaseCancelSuspendResumeTest`
- `SpiWiringIntegrationTest`

All fail with Awaitility `ConditionTimeout` at exactly 10–15s. Tests pass locally.

**Root cause:** The `persistence-hibernate` Maven profile is active by default. It adds `testcontainers-postgresql` and boots a real PostgreSQL container for every test run. GitHub-hosted CI runners are 2–3× slower than local Docker, so the Testcontainers startup eats into the Awaitility budget before the case engine has processed anything.

**Why these tests don't need real JPA:** All 5 classes test case lifecycle and SPI wiring behaviour. Their assertions are against `CaseInstanceCache` (in-memory) and in-memory state. None query the JPA persistence layer directly — they don't assert on Hibernate entities, Flyway schema state, or PostgreSQL-specific behaviour.

---

## Solution

Activate the **existing in-memory test infrastructure** for the 5 failing test classes using Quarkus `@TestProfile`.

### Infrastructure already in place

| Artifact | Status |
|----------|--------|
| `application-memory.properties` | ✅ exists — H2 datasource, in-memory alternatives |
| `casehub-engine-persistence-memory` | ✅ always on test classpath |
| `InMemoryCaseInstanceRepository` | ✅ implements `CaseInstanceRepository` + `CrossTenantCaseInstanceRepository` |
| `InMemoryEventLogRepository` | ✅ implements `EventLogRepository` + `CrossTenantEventLogRepository`, including `findByCaseAndTypes` |
| `InMemoryCaseMetaModelRepository` | ✅ implements `CaseMetaModelRepository` |

### What's missing

- `quarkus-jdbc-h2` is only in the `persistence-memory` Maven profile — needs to move to main test deps so it's always available.
- A `QuarkusTestProfile` implementation selecting the `"memory"` config profile.

---

## Design

### 1. Promote H2 dependency

Move `quarkus-jdbc-h2` from the `persistence-memory` Maven profile to the main `<dependencies>` test scope in `runtime/pom.xml`. This makes it available to `@TestProfile`-annotated tests regardless of which Maven profile is active.

### 2. Create `MemoryTestProfile`

New class at `runtime/src/test/java/io/casehub/engine/testing/MemoryTestProfile.java`:

```java
package io.casehub.engine.testing;

import io.quarkus.test.junit.QuarkusTestProfile;
import java.util.Map;

public class MemoryTestProfile implements QuarkusTestProfile {
    @Override
    public String getConfigProfile() {
        return "memory";
    }
}
```

Quarkus loads `application.properties` + `application-memory.properties` when `getConfigProfile()` returns `"memory"`. The memory properties activate the in-memory alternatives and H2 datasource. All base `quarkus.arc.exclude-types` from `application.properties` are inherited (Quarkus merges profile config on top of base).

### 3. Annotate the 5 failing test classes

Add `@TestProfile(MemoryTestProfile.class)` to:

- `SimpleCaseHubBeanTest`
- `YamlSimpleCaseHubBeanTest`
- `MultipleCaseInstancesTest`
- `CaseCancelSuspendResumeTest`
- `SpiWiringIntegrationTest`

### 4. `application-memory.properties` gap check

The existing `application-memory.properties` activates 3 alternatives:

```properties
quarkus.arc.selected-alternatives=\
  io.casehub.persistence.memory.InMemoryCaseMetaModelRepository,\
  io.casehub.persistence.memory.InMemoryCaseInstanceRepository,\
  io.casehub.persistence.memory.InMemoryEventLogRepository
```

`SubCaseGroupRepository` is NOT needed: it is only used by `SubCaseCompletionService` in the `casehub-engine-blackboard` module, which is not a dependency of the `runtime` module.

The `casehub_crosstenancy` PostgreSQL init script is skipped automatically when an explicit H2 datasource is configured (DevServices is suppressed). The in-memory implementations don't use PostgreSQL role-based RLS. ✅

---

## Test execution model after the change

```
mvn test -pl casehub-engine
 ├── Quarkus instance A (default profile, Testcontainers PostgreSQL)
 │    ├── CaseCancelSuspendResumeTest     ← moved to MemoryTestProfile
 │    ├── ... (JPA-heavy tests stay here) 
 │    └── ...
 └── Quarkus instance B (memory profile, H2 in-memory)
      ├── SimpleCaseHubBeanTest
      ├── YamlSimpleCaseHubBeanTest
      ├── MultipleCaseInstancesTest
      ├── CaseCancelSuspendResumeTest
      └── SpiWiringIntegrationTest
```

Quarkus starts one application instance per unique `@TestProfile`. Tests sharing a profile share that instance (same JVM, different test class runs sequentially). Tests without a `@TestProfile` use the default profile instance.

---

## Behaviour and coverage

**No coverage loss:** The 5 classes test case lifecycle, SPI wiring, context propagation, and worker scheduling. These are behavioural contracts — they belong in the in-memory tier. The JPA persistence layer has its own test module (`casehub-engine-persistence-hibernate`) with its own integration tests.

**Expected CI improvement:**
- Docker startup (currently ~8–12s on CI): eliminated for these 5 classes
- Case execution in in-memory profile: < 200ms per case
- Awaitility timeouts (10–15s): now have 10–15x the necessary headroom

**Timeouts are NOT increased.** If a test fails in the in-memory profile, it's a genuine logic failure, not latency. The tight timeouts remain as the sentinel.

---

## Files changed

| File | Change |
|------|--------|
| `runtime/pom.xml` | Move `quarkus-jdbc-h2` from `persistence-memory` Maven profile to main test deps |
| `runtime/src/test/java/io/casehub/engine/testing/MemoryTestProfile.java` | New class |
| `runtime/src/test/java/io/casehub/engine/SimpleCaseHubBeanTest.java` | Add `@TestProfile` |
| `runtime/src/test/java/io/casehub/engine/YamlSimpleCaseHubBeanTest.java` | Add `@TestProfile` |
| `runtime/src/test/java/io/casehub/engine/MultipleCaseInstancesTest.java` | Add `@TestProfile` |
| `runtime/src/test/java/io/casehub/engine/CaseCancelSuspendResumeTest.java` | Add `@TestProfile` |
| `runtime/src/test/java/io/casehub/engine/SpiWiringIntegrationTest.java` | Add `@TestProfile` |