# ACL Engine-REST SPI Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #768 — feat: move ACL checks into engine-rest SPI
**Issue group:** #768

**Goal:** Centralize ACL enforcement in engine-rest so all consumers get two-layer protection (service-level + HTTP filter) without writing their own filters.

**Architecture:** Add `@AclCheck` annotation to API interface methods in `engine-api`. Add `AclEnforcementFilter` (`@ServerRequestFilter`) in `engine-rest` that reads the annotation via interface-walking reflection. Fill all 27 service-level gaps with `requireCaseAccess()` / `canAccess()` calls.

**Tech Stack:** Quarkus 3.32.2, JAX-RS, CDI, `@ServerRequestFilter`, platform-api ACL SPIs

## Global Constraints

- `AccessControlProvider.canAccess()` defaults to `true` (permissive). Never use `accessibleResources()` in the filter — it defaults to empty list.
- `@AclCheck` goes on API interface methods AND on `EvolutionMcpAdapter` concrete methods directly.
- Filter must walk interface hierarchy for annotation discovery (generated JAX-RS classes don't inherit method annotations).
- Service-level checks are the primary enforcement (covers REST + MCP paths). Filter is defense-in-depth for REST only.
- `getResearchCorpus` — intentionally no ACL (global query, no caseId).

---

## Batch 1: Foundation — @AclCheck annotation + service-level gap fills

After this batch: all 45 endpoints have service-level ACL. Annotation exists and is applied. No filter yet — that comes in Batch 2.

### Task 1: Create @AclCheck annotation and apply to all API interfaces

**Files:**
- Create: `api/src/main/java/io/casehub/api/acl/AclCheck.java`
- Modify: `api/src/main/java/io/casehub/api/engine/rest/EngineCaseApi.java`
- Modify: `api/src/main/java/io/casehub/api/engine/rest/EngineCaseControlApi.java`
- Modify: `api/src/main/java/io/casehub/api/engine/rest/EngineCaseDefinitionApi.java`
- Modify: `api/src/main/java/io/casehub/api/engine/rest/EngineEventLogApi.java`
- Modify: `api/src/main/java/io/casehub/api/engine/rest/EnginePlanApi.java`
- Modify: `rest/src/main/java/io/casehub/engine/rest/service/EvolutionMcpAdapter.java`
- Test: `api/src/test/java/io/casehub/api/acl/AclCheckAnnotationCoverageTest.java`

**Interfaces:**
- Produces: `@AclCheck` annotation (`io.casehub.api.acl.AclCheck`) — used by Task 3 (filter) and Task 4 (coverage test)

- [ ] **Step 1: Write the annotation coverage test**

Create `api/src/test/java/io/casehub/api/acl/AclCheckAnnotationCoverageTest.java`:

```java
package io.casehub.api.acl;

import static org.assertj.core.api.Assertions.assertThat;

import io.casehub.api.engine.rest.*;
import io.casehub.platform.api.mcp.PathParam;
import java.lang.reflect.Method;
import java.lang.reflect.Parameter;
import java.util.*;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.MethodSource;

class AclCheckAnnotationCoverageTest {

    static List<Method> caseIdMethods() {
        List<Method> methods = new ArrayList<>();
        for (Class<?> api : List.of(
                EngineCaseApi.class,
                EngineCaseControlApi.class,
                EngineCaseDefinitionApi.class,
                EngineEventLogApi.class,
                EnginePlanApi.class)) {
            for (Method m : api.getDeclaredMethods()) {
                if (hasCaseIdParam(m)) {
                    methods.add(m);
                }
            }
        }
        return methods;
    }

    private static boolean hasCaseIdParam(Method m) {
        for (Parameter p : m.getParameters()) {
            if (p.isAnnotationPresent(PathParam.class)
                    && UUID.class.equals(p.getType())
                    && "caseId".equals(p.getName())) {
                return true;
            }
        }
        return false;
    }

    @ParameterizedTest(name = "{0}")
    @MethodSource("caseIdMethods")
    void methodWithCaseIdParam_hasAclCheck(Method method) {
        AclCheck check = method.getAnnotation(AclCheck.class);
        assertThat(check)
                .as("Method %s.%s has @PathParam caseId but no @AclCheck",
                        method.getDeclaringClass().getSimpleName(), method.getName())
                .isNotNull();
    }
}
```

- [ ] **Step 2: Run the test — verify it fails**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=AclCheckAnnotationCoverageTest -q`
Expected: compilation failure — `AclCheck` does not exist yet.

- [ ] **Step 3: Create the @AclCheck annotation**

Create `api/src/main/java/io/casehub/api/acl/AclCheck.java`:

```java
package io.casehub.api.acl;

import io.casehub.platform.api.acl.AclAction;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface AclCheck {
    String resourceType();
    AclAction action();
    String resourceIdParam() default "";
}
```

- [ ] **Step 4: Apply @AclCheck to all API interface methods**

Add `@AclCheck` to every method on all 5 API interfaces per the annotation mapping table in the spec. Example for `EngineCaseApi`:

```java
@AclCheck(resourceType = EngineResourceTypes.CASE, action = AclAction.READ)
@PlatformQuery("list-cases")
CasePage listCases(...);

@AclCheck(resourceType = EngineResourceTypes.CASE, action = AclAction.READ, resourceIdParam = "caseId")
@PlatformQuery("get-case-by-id")
CaseInstanceView getCaseById(@PathParam UUID caseId, ...);

@AclCheck(resourceType = EngineResourceTypes.CASE_DEFINITION, action = AclAction.WRITE)
@PlatformMutation("start-case")
CaseInstanceView startCase(StartCaseRequest request, ...);
```

Apply the same pattern to `EngineCaseControlApi`, `EngineCaseDefinitionApi`, `EngineEventLogApi`, `EnginePlanApi`.

- [ ] **Step 5: Apply @AclCheck to EvolutionMcpAdapter methods**

Add `@AclCheck` to all 22 case-scoped methods on `EvolutionMcpAdapter` per the spec's annotation mapping table. `getResearchCorpus` gets no annotation.

- [ ] **Step 6: Install api module and run annotation coverage test**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=AclCheckAnnotationCoverageTest -q`
Expected: PASS — all methods with `@PathParam caseId` have `@AclCheck`.

- [ ] **Step 7: Commit**

```bash
git add api/src/main/java/io/casehub/api/acl/AclCheck.java
git add api/src/main/java/io/casehub/api/engine/rest/
git add api/src/test/java/io/casehub/api/acl/AclCheckAnnotationCoverageTest.java
git add rest/src/main/java/io/casehub/engine/rest/service/EvolutionMcpAdapter.java
git commit -m "feat(#768): add @AclCheck annotation and apply to all API methods

Refs #768"
```

### Task 2: Fill service-level ACL gaps

**Files:**
- Modify: `rest-core/src/main/java/io/casehub/engine/rest/service/CaseService.java`
- Modify: `rest-core/src/main/java/io/casehub/engine/rest/service/CaseDefinitionService.java`
- Modify: `rest/src/main/java/io/casehub/engine/rest/service/DefaultEngineCaseApi.java`
- Modify: `rest/src/main/java/io/casehub/engine/rest/service/DefaultEnginePlanApi.java`
- Modify: `rest/src/main/java/io/casehub/engine/rest/service/EvolutionMcpAdapter.java`
- Modify: `rest/src/test/java/io/casehub/engine/rest/service/CaseServiceAclTest.java`

**Interfaces:**
- Consumes: `CaseService.requireCaseAccess(UUID, AclAction)` — existing method
- Consumes: `AccessControlProvider.canAccess(String, ResourceId, AclAction)` — platform SPI

- [ ] **Step 1: Write failing test for startCaseAndMap ACL check**

Add to `CaseServiceAclTest.java`:

```java
@Test
void startCaseAndMap_denied_throwsAccessDenied() {
    aclAllowed = false;
    var request = new StartCaseRequest("ns", "name", "1.0", null);
    assertThatThrownBy(() -> caseService.startCaseAndMap(request, "t1"))
            .isInstanceOf(AccessDeniedException.class);
}
```

This requires `CaseService` to have enough dependencies wired. The existing test constructs `CaseService` with nulls for unused dependencies. `startCaseAndMap` needs `definitionRegistry` and `runtime`, but the ACL check should happen before those are called. Add a `CaseDefinitionRegistry` stub that throws if called:

```java
CaseDefinitionRegistry stubRegistry = new CaseDefinitionRegistry() {
    @Override
    public java.util.Optional<io.casehub.engine.common.internal.model.CaseMetaModel> findByIdentity(
            String namespace, String name, String version) {
        throw new AssertionError("ACL should have blocked before reaching registry");
    }
    // other methods with default returns
};
```

Update `caseService` construction in `setUp()` to use this registry.

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl rest -Dtest=CaseServiceAclTest#startCaseAndMap_denied_throwsAccessDenied -q`
Expected: FAIL — `startCaseAndMap` doesn't check ACL yet.

- [ ] **Step 3: Add ACL check to CaseService.startCaseAndMap()**

Use `ide_edit_member` or Edit tool. Add at the start of `startCaseAndMap()`:

```java
String actorId = currentPrincipal.actorId();
ResourceId rid = new ResourceId(EngineResourceTypes.CASE_DEFINITION,
        request.namespace() + "/" + request.name());
if (!accessControlProvider.canAccess(actorId, rid, AclAction.WRITE)) {
    throw new AccessDeniedException(actorId, rid, AclAction.WRITE);
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl rest -Dtest=CaseServiceAclTest -q`
Expected: ALL tests PASS.

- [ ] **Step 5: Add ACL filtering to CaseDefinitionService.getDefinitionsByName()**

Add inline ACL filtering (same pattern as `listDefinitions()`). After the query, filter results through `accessControlProvider.canAccess()`:

```java
String actorId = currentPrincipal.actorId();
return metaModelRepository.query(query, resolvedTenancyId).stream()
        .filter(meta -> accessControlProvider.canAccess(actorId,
                new ResourceId(EngineResourceTypes.CASE_DEFINITION,
                        meta.getNamespace() + "/" + meta.getName() + "/" + meta.getVersion()),
                AclAction.READ))
        .map(meta -> {
            CaseDefinition def = definitionRegistry.getCaseDefinition(meta);
            return mapDefinition(meta, def);
        })
        .filter(Objects::nonNull)
        .toList();
```

- [ ] **Step 6: Add ACL check to CaseDefinitionService.getDefinitionByKey()**

After the `findByIdentity` lookup, before `mapDefinition`:

```java
String actorId = currentPrincipal.actorId();
ResourceId rid = new ResourceId(EngineResourceTypes.CASE_DEFINITION,
        namespace + "/" + name + "/" + version);
if (!accessControlProvider.canAccess(actorId, rid, AclAction.READ)) {
    throw new AccessDeniedException(actorId, rid, AclAction.READ);
}
```

- [ ] **Step 7: Add requireCaseAccess to SSE stream endpoints**

In `DefaultEngineCaseApi.caseStream()`:
```java
@Override
public Flow.Publisher<CaseStreamEventView> caseStream(UUID caseId) {
    caseService.requireCaseAccess(caseId, AclAction.READ);
    return caseStreamBroadcaster.stream(caseId);
}
```

In `DefaultEnginePlanApi.executionStateStream()` — inject `CaseService` and add the check:
```java
@Inject CaseService caseService;

@Override
public Flow.Publisher<JsonNode> executionStateStream(UUID caseId) {
    caseService.requireCaseAccess(caseId, AclAction.READ);
    return executionStateBroadcaster.stream(caseId).map(e -> objectMapper.valueToTree(e));
}
```

- [ ] **Step 8: Add requireCaseAccess to all EvolutionMcpAdapter case-scoped methods**

Inject `CaseService` into `EvolutionMcpAdapter`. Add `caseService.requireCaseAccess(caseId, AclAction.XXX)` as the first call in each case-scoped method. Example:

```java
@Inject CaseService caseService;

@PlatformQuery("Get evolution tick history for a case")
public List<TickTrace> getTickHistory(@PathParam UUID caseId, @Nullable Integer limit) {
    caseService.requireCaseAccess(caseId, AclAction.READ);
    return delegate.getTickHistory(caseId, limit);
}

@PlatformMutation("Add a deny pattern to block improvement categories")
public void addDenyPattern(@PathParam UUID caseId, String tenancyId, String pattern) {
    caseService.requireCaseAccess(caseId, AclAction.WRITE);
    delegate.addDenyPattern(caseId, tenancyId, pattern);
}

@PlatformMutation("Reset the health circuit breaker")
public void resetCircuitBreaker(@PathParam UUID caseId) {
    caseService.requireCaseAccess(caseId, AclAction.ADMIN);
    delegate.resetCircuitBreaker(caseId);
}
```

Apply to all 21 remaining case-scoped methods per the spec's mapping table. `getResearchCorpus` gets no check.

- [ ] **Step 9: Run full rest module tests**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl rest -q`
Expected: ALL tests PASS.

- [ ] **Step 10: Commit**

```bash
git add rest-core/src/main/java/io/casehub/engine/rest/service/CaseService.java
git add rest-core/src/main/java/io/casehub/engine/rest/service/CaseDefinitionService.java
git add rest/src/main/java/io/casehub/engine/rest/service/DefaultEngineCaseApi.java
git add rest/src/main/java/io/casehub/engine/rest/service/DefaultEnginePlanApi.java
git add rest/src/main/java/io/casehub/engine/rest/service/EvolutionMcpAdapter.java
git add rest/src/test/java/io/casehub/engine/rest/service/CaseServiceAclTest.java
git commit -m "feat(#768): fill all 27 service-level ACL gaps

Add canAccess check to startCaseAndMap, getDefinitionsByName,
getDefinitionByKey, caseStream, executionStateStream, and all 21
case-scoped EvolutionMcpAdapter methods.

Refs #768"
```

## Batch 2: Defense-in-depth — AclEnforcementFilter

After this batch: HTTP-layer filter provides defense-in-depth for all REST endpoints. Full two-layer ACL enforcement.

### Task 3: Implement AclEnforcementFilter with tests

**Files:**
- Create: `rest/src/main/java/io/casehub/engine/rest/filter/AclEnforcementFilter.java`
- Create: `rest/src/test/java/io/casehub/engine/rest/filter/AclEnforcementFilterTest.java`

**Interfaces:**
- Consumes: `@AclCheck` annotation (from Task 1)
- Consumes: `AccessControlProvider.canAccess(String, ResourceId, AclAction)` — platform SPI
- Consumes: `CurrentPrincipal.actorId()` — platform SPI

- [ ] **Step 1: Write failing unit test for filter — deny when canAccess returns false**

Create `rest/src/test/java/io/casehub/engine/rest/filter/AclEnforcementFilterTest.java`:

```java
package io.casehub.engine.rest.filter;

import static org.assertj.core.api.Assertions.assertThat;

import io.casehub.api.acl.AclCheck;
import io.casehub.api.acl.EngineResourceTypes;
import io.casehub.platform.api.acl.AccessControlProvider;
import io.casehub.platform.api.acl.AclAction;
import io.casehub.platform.api.acl.ResourceId;
import io.casehub.platform.api.identity.CurrentPrincipal;
import jakarta.ws.rs.container.ContainerRequestContext;
import jakarta.ws.rs.container.ResourceInfo;
import jakarta.ws.rs.core.MultivaluedHashMap;
import jakarta.ws.rs.core.UriInfo;
import java.lang.reflect.Method;
import java.util.Optional;
import java.util.Set;
import java.util.UUID;
import org.jboss.resteasy.reactive.RestResponse;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

class AclEnforcementFilterTest {

    private AclEnforcementFilter filter;
    private boolean aclAllowed;
    private ResourceId capturedResourceId;

    // Test interface with @AclCheck — simulates an API interface
    interface TestApi {
        @AclCheck(resourceType = EngineResourceTypes.CASE, action = AclAction.READ, resourceIdParam = "caseId")
        void getCaseById(UUID caseId);

        @AclCheck(resourceType = EngineResourceTypes.CASE, action = AclAction.READ)
        void listCases();

        void noAclMethod();
    }

    // Simulates generated class implementing the interface
    static class TestApiImpl implements TestApi {
        @Override public void getCaseById(UUID caseId) {}
        @Override public void listCases() {}
        @Override public void noAclMethod() {}
    }

    @BeforeEach
    void setUp() {
        aclAllowed = true;
        capturedResourceId = null;

        AccessControlProvider aclProvider = new AccessControlProvider() {
            @Override
            public boolean canAccess(String actorId, ResourceId resourceId, AclAction action) {
                capturedResourceId = resourceId;
                return aclAllowed;
            }
        };
        CurrentPrincipal principal = new CurrentPrincipal() {
            @Override public String actorId() { return "alice"; }
            @Override public Set<String> groups() { return Set.of(); }
            @Override public String tenancyId() { return "t1"; }
            @Override public boolean isCrossTenantAdmin() { return false; }
        };

        filter = new AclEnforcementFilter();
        filter.accessControlProvider = aclProvider;
        filter.currentPrincipal = principal;
    }

    @Test
    void withResourceIdParam_denied_returns403() throws Exception {
        aclAllowed = false;
        UUID caseId = UUID.randomUUID();

        Method implMethod = TestApiImpl.class.getMethod("getCaseById", UUID.class);
        var pathParams = new MultivaluedHashMap<String, String>();
        pathParams.add("caseId", caseId.toString());

        Optional<RestResponse<?>> result = filter.enforce(
                mockRequestContext(pathParams), mockResourceInfo(implMethod, TestApiImpl.class));

        assertThat(result).isPresent();
        assertThat(result.get().getStatus()).isEqualTo(403);
    }

    @Test
    void withResourceIdParam_allowed_passesThrough() throws Exception {
        UUID caseId = UUID.randomUUID();

        Method implMethod = TestApiImpl.class.getMethod("getCaseById", UUID.class);
        var pathParams = new MultivaluedHashMap<String, String>();
        pathParams.add("caseId", caseId.toString());

        Optional<RestResponse<?>> result = filter.enforce(
                mockRequestContext(pathParams), mockResourceInfo(implMethod, TestApiImpl.class));

        assertThat(result).isEmpty();
        assertThat(capturedResourceId).isEqualTo(
                new ResourceId(EngineResourceTypes.CASE, caseId.toString()));
    }

    @Test
    void emptyResourceIdParam_passesThrough() throws Exception {
        Method implMethod = TestApiImpl.class.getMethod("listCases");

        Optional<RestResponse<?>> result = filter.enforce(
                mockRequestContext(new MultivaluedHashMap<>()),
                mockResourceInfo(implMethod, TestApiImpl.class));

        assertThat(result).isEmpty();
    }

    @Test
    void noAnnotation_passesThrough() throws Exception {
        Method implMethod = TestApiImpl.class.getMethod("noAclMethod");

        Optional<RestResponse<?>> result = filter.enforce(
                mockRequestContext(new MultivaluedHashMap<>()),
                mockResourceInfo(implMethod, TestApiImpl.class));

        assertThat(result).isEmpty();
    }

    @Test
    void annotationDiscovery_walksInterfaces() throws Exception {
        // The impl class does NOT have @AclCheck — only the interface does
        Method implMethod = TestApiImpl.class.getMethod("getCaseById", UUID.class);
        assertThat(implMethod.getAnnotation(AclCheck.class)).isNull();

        // But the filter should find it on the interface
        aclAllowed = false;
        var pathParams = new MultivaluedHashMap<String, String>();
        pathParams.add("caseId", UUID.randomUUID().toString());

        Optional<RestResponse<?>> result = filter.enforce(
                mockRequestContext(pathParams), mockResourceInfo(implMethod, TestApiImpl.class));

        assertThat(result).isPresent();
        assertThat(result.get().getStatus()).isEqualTo(403);
    }

    // Stub helpers — create minimal mocks without Mockito
    private ContainerRequestContext mockRequestContext(MultivaluedHashMap<String, String> pathParams) {
        return java.lang.reflect.Proxy.newProxyInstance(
                getClass().getClassLoader(),
                new Class[]{ContainerRequestContext.class},
                (proxy, method, args) -> {
                    if ("getUriInfo".equals(method.getName())) {
                        return java.lang.reflect.Proxy.newProxyInstance(
                                getClass().getClassLoader(),
                                new Class[]{UriInfo.class},
                                (p2, m2, a2) -> {
                                    if ("getPathParameters".equals(m2.getName())) return pathParams;
                                    return null;
                                });
                    }
                    return null;
                });
    }

    private ResourceInfo mockResourceInfo(Method method, Class<?> resourceClass) {
        return new ResourceInfo() {
            @Override public Method getResourceMethod() { return method; }
            @Override public Class<?> getResourceClass() { return resourceClass; }
        };
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl rest -Dtest=AclEnforcementFilterTest -q`
Expected: FAIL — `AclEnforcementFilter` does not exist yet.

- [ ] **Step 3: Implement AclEnforcementFilter**

Create `rest/src/main/java/io/casehub/engine/rest/filter/AclEnforcementFilter.java`:

```java
package io.casehub.engine.rest.filter;

import io.casehub.api.acl.AclCheck;
import io.casehub.engine.rest.dto.ProblemDetail;
import io.casehub.platform.api.acl.AccessControlProvider;
import io.casehub.platform.api.acl.ResourceId;
import io.casehub.platform.api.identity.CurrentPrincipal;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.ws.rs.container.ContainerRequestContext;
import jakarta.ws.rs.container.ResourceInfo;
import java.lang.reflect.Method;
import java.util.Optional;
import org.jboss.resteasy.reactive.RestResponse;
import org.jboss.resteasy.reactive.server.ServerRequestFilter;

@ApplicationScoped
public class AclEnforcementFilter {

    @Inject AccessControlProvider accessControlProvider;
    @Inject CurrentPrincipal currentPrincipal;

    @ServerRequestFilter
    public Optional<RestResponse<ProblemDetail>> enforce(
            ContainerRequestContext ctx, ResourceInfo resourceInfo) {
        AclCheck check = discoverAclCheck(resourceInfo);
        if (check == null) {
            return Optional.empty();
        }

        String resourceIdParam = check.resourceIdParam();
        if (resourceIdParam.isEmpty()) {
            return Optional.empty();
        }

        String resourceId = ctx.getUriInfo().getPathParameters().getFirst(resourceIdParam);
        if (resourceId == null) {
            return Optional.empty();
        }

        String actorId = currentPrincipal.actorId();
        ResourceId rid = new ResourceId(check.resourceType(), resourceId);

        if (!accessControlProvider.canAccess(actorId, rid, check.action())) {
            return Optional.of(RestResponse.status(
                    RestResponse.Status.FORBIDDEN,
                    new ProblemDetail("Access denied", 403, "Insufficient permissions")));
        }

        return Optional.empty();
    }

    private AclCheck discoverAclCheck(ResourceInfo resourceInfo) {
        Method method = resourceInfo.getResourceMethod();
        AclCheck check = method.getAnnotation(AclCheck.class);
        if (check != null) {
            return check;
        }

        Class<?> resourceClass = resourceInfo.getResourceClass();
        for (Class<?> iface : resourceClass.getInterfaces()) {
            try {
                Method ifaceMethod = iface.getMethod(method.getName(), method.getParameterTypes());
                check = ifaceMethod.getAnnotation(AclCheck.class);
                if (check != null) {
                    return check;
                }
            } catch (NoSuchMethodException ignored) {
            }
        }

        return null;
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl rest -Dtest=AclEnforcementFilterTest -q`
Expected: ALL tests PASS.

- [ ] **Step 5: Run full rest module test suite**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl rest -q`
Expected: ALL tests PASS.

- [ ] **Step 6: Commit**

```bash
git add rest/src/main/java/io/casehub/engine/rest/filter/AclEnforcementFilter.java
git add rest/src/test/java/io/casehub/engine/rest/filter/AclEnforcementFilterTest.java
git commit -m "feat(#768): add AclEnforcementFilter — HTTP-layer defense-in-depth

@ServerRequestFilter walks interface hierarchy to discover @AclCheck
annotations. Enforces canAccess() for endpoints with resourceIdParam.
Defers to service layer for list/composite-key endpoints.

Refs #768"
```

## Batch 3: Verification — full build + cross-module tests

After this batch: full project builds clean, all modules pass tests.

### Task 4: Full build verification and cleanup

**Files:**
- No new files — verification only
- Possibly modify: any files with compilation errors discovered during full build

**Interfaces:**
- Consumes: all changes from Tasks 1-3

- [ ] **Step 1: Full project install**

Run: `mvn install -DskipTests -q`
Expected: BUILD SUCCESS — all modules compile.

- [ ] **Step 2: Full test suite**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -q`
Expected: ALL tests PASS.

- [ ] **Step 3: Fix any failures**

If compilation or test failures occur, fix them and re-run. Common issues:
- Import ordering after adding `@AclCheck` imports to API interfaces
- CDI wiring issues from injecting `CaseService` into `EvolutionMcpAdapter` or `DefaultEnginePlanApi`

- [ ] **Step 4: Final commit (if any fixes were needed)**

```bash
git add -A
git commit -m "fix(#768): resolve build issues from ACL integration

Refs #768"
```

## References

- [2026-10-19-acl-engine-rest-spi-design.md] — design spec this plan implements
- [rest-core/src/main/java/io/casehub/engine/rest/service/CaseService.java:134-147] — existing `requireCaseAccess()`
- [rest/src/test/java/io/casehub/engine/rest/service/CaseServiceAclTest.java] — existing ACL test pattern
- [api/src/main/java/io/casehub/api/acl/EngineResourceTypes.java] — resource type constants
- [platform-api AccessControlProvider] — SPI interface with permissive defaults
- [scaffold AclRequestFilter.java] — pattern being replaced
- [GitHub #768] — focal issue
- [scaffold#69] — follow-up for scaffold cleanup
