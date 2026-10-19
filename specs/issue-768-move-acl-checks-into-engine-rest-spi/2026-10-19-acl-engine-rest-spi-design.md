# Design: Move ACL checks into engine-rest SPI

**Issue:** engine#768
**Date:** 2026-10-19
**Branch:** issue-768-move-acl-checks-into-engine-rest-spi

## Problem

engine-rest is an opt-in JAX-RS library used by multiple consumers (scaffold, future deployments). ACL enforcement is inconsistent:

- **18 endpoints** have service-level ACL checks via `CaseService.requireCaseAccess()` or inline filtering
- **27 endpoints** have no ACL at all (5 in core APIs, 22 in EvolutionMcpAdapter)
- Consumers must write their own HTTP-layer filter (scaffold's `AclRequestFilter`) to fill gaps and add defense-in-depth

This means every deployment duplicates ACL infrastructure, and the EvolutionMcpAdapter surface is completely unprotected.

## Solution

Two-layer ACL enforcement owned by engine-rest:

1. **`@AclCheck` annotation** on every API method — declares the resource type, action, and which path param holds the resource ID
2. **`AclEnforcementFilter`** (`@ServerRequestFilter`) — reads `@AclCheck` at runtime, enforces ACL at the HTTP layer
3. **Service-level gap fills** — add `requireCaseAccess()` / `canAccess()` calls to the 27 uncovered endpoints

After this lands, scaffold deletes its `AclRequestFilter` (tracked as a follow-up issue on casehubio/scaffold).

## Component 1: `@AclCheck` annotation

**Module:** `casehub-engine-api`
**Package:** `io.casehub.api.acl`

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface AclCheck {
    String resourceType();
    AclAction action();
    String resourceIdParam() default "";
}
```

- `resourceType` — one of `EngineResourceTypes` constants (CASE, CASE_DEFINITION, etc.)
- `action` — READ, WRITE, or ADMIN
- `resourceIdParam` — name of the `@PathParam` holding the resource ID. Empty for list endpoints and `startCase`-style operations where no single resource is targeted.

### Annotation mapping — all 45 endpoints

**EngineCaseApi (8 methods):**

| Method | resourceType | action | resourceIdParam |
|--------|-------------|--------|----------------|
| listCases | CASE | READ | (empty) |
| getCaseById | CASE | READ | caseId |
| startCase | CASE_DEFINITION | WRITE | (empty) |
| getCaseContext | CASE | READ | caseId |
| getCaseContextPath | CASE | READ | caseId |
| getPlanItems | CASE | READ | caseId |
| getGoals | CASE | READ | caseId |
| caseStream | CASE | READ | caseId |

**EngineCaseControlApi (4 methods):**

| Method | resourceType | action | resourceIdParam |
|--------|-------------|--------|----------------|
| suspendCase | CASE | ADMIN | caseId |
| resumeCase | CASE | ADMIN | caseId |
| cancelCase | CASE | ADMIN | caseId |
| sendSignal | CASE | WRITE | caseId |

**EngineCaseDefinitionApi (3 methods):**

| Method | resourceType | action | resourceIdParam |
|--------|-------------|--------|----------------|
| listDefinitions | CASE_DEFINITION | READ | (empty) |
| getDefinitionsByName | CASE_DEFINITION | READ | (empty) |
| getDefinitionByKey | CASE_DEFINITION | READ | (empty) |

Note: `getDefinitionsByName` and `getDefinitionByKey` use empty `resourceIdParam` because namespace/name/version are composite — the service layer constructs the `ResourceId` from multiple params. The filter defers to service-level enforcement for these.

**EngineEventLogApi (1 method):**

| Method | resourceType | action | resourceIdParam |
|--------|-------------|--------|----------------|
| getEventLog | CASE | READ | caseId |

**EnginePlanApi (7 methods):**

| Method | resourceType | action | resourceIdParam |
|--------|-------------|--------|----------------|
| getPlanModel | CASE | READ | caseId |
| getPlanDefinitions | CASE | READ | caseId |
| getDecomposition | CASE | READ | caseId |
| getDagPlan | CASE | READ | caseId |
| getDagResult | CASE | READ | caseId |
| getExecutionState | CASE | READ | caseId |
| executionStateStream | CASE | READ | caseId |

**EvolutionMcpAdapter (22 methods):**

| Method | resourceType | action | resourceIdParam |
|--------|-------------|--------|----------------|
| getTickHistory | CASE | READ | caseId |
| getDenyPatterns | CASE | READ | caseId |
| getSummary | CASE | READ | caseId |
| getInbox | CASE | READ | caseId |
| getReadinessReport | CASE | READ | caseId |
| getEvolutionState | CASE | READ | caseId |
| getResearchCorpus | — | — | — |
| getArtifactTrail | CASE | READ | caseId |
| getStreamProgress | CASE | READ | caseId |
| addDenyPattern | CASE | WRITE | caseId |
| removeDenyPattern | CASE | WRITE | caseId |
| resolveGate | CASE | WRITE | caseId |
| addWatchPattern | CASE | WRITE | caseId |
| removeWatchPattern | CASE | WRITE | caseId |
| blockImprovement | CASE | WRITE | caseId |
| unblockImprovement | CASE | WRITE | caseId |
| pauseCategory | CASE | WRITE | caseId |
| unpauseCategory | CASE | WRITE | caseId |
| resetCircuitBreaker | CASE | ADMIN | caseId |
| triggerReadinessValidation | CASE | WRITE | caseId |
| setGatePolicy | CASE | ADMIN | caseId |
| getWatchPatterns | CASE | READ | caseId |
| getGatePolicy | CASE | READ | caseId |
| getStages | CASE | READ | caseId |
| getCategories | CASE | READ | caseId |

`getResearchCorpus` has no `caseId` — it's a global query. No `@AclCheck` annotation.

## Component 2: `AclEnforcementFilter`

**Module:** `casehub-engine-rest`
**Package:** `io.casehub.engine.rest.filter`

A Quarkus `@ServerRequestFilter` that reads `@AclCheck` from the matched resource method and enforces ACL at the HTTP layer.

### Behaviour

1. Read `@AclCheck` from `ResourceInfo.getResourceMethod()`. If absent, pass through.
2. Get `actorId` from `CurrentPrincipal`.
3. If `resourceIdParam` is non-empty: extract resource ID from `UriInfo.getPathParameters()`, construct `ResourceId`, call `accessControlProvider.canAccess()`. Deny returns 403.
4. If `resourceIdParam` is empty AND action is WRITE or ADMIN: call `accessControlProvider.accessibleResources()` to check if the actor has any access of that type. Empty result returns 403. (Matches scaffold's `startCase` pattern.)
5. If `resourceIdParam` is empty AND action is READ: pass through — service-layer filtering handles list results.

### Failure mode

Fail-closed. If `canAccess()` throws, the request is denied with 500 (not fail-open like the scaffold's filter). Security enforcement should not silently allow on error.

### Dependencies

- `AccessControlProvider` — injected via `@Inject`. The platform interface has all-permissive defaults (`canAccess()` returns `true`), so deployments without a real ACL provider pass everything through. No `Instance<>` or `@DefaultBean` needed — the interface defaults already provide the no-op behaviour.
- `CurrentPrincipal` — injected via `@Inject`.

## Component 3: Service-level gap fills

### Core API gaps (5 endpoints)

**`CaseService.startCaseAndMap()`** — add WRITE check on CASE_DEFINITION before starting the case. Construct `ResourceId` from `request.namespace() + "/" + request.name()`.

**`CaseDefinitionService.getDefinitionsByName()`** — add inline ACL filtering (same pattern as `listDefinitions()`). Filter results through `accessControlProvider.canAccess()` with CASE_DEFINITION / READ.

**`CaseDefinitionService.getDefinitionByKey()`** — add point check. Construct `ResourceId` from namespace/name/version, call `canAccess()`, throw `AccessDeniedException` on deny.

**`DefaultEngineCaseApi.caseStream()`** — add `caseService.requireCaseAccess(caseId, AclAction.READ)` before returning the SSE publisher.

**`DefaultEnginePlanApi.executionStateStream()`** — same pattern: add `requireCaseAccess` before returning the publisher. Inject `CaseService` into `DefaultEnginePlanApi`.

### EvolutionMcpAdapter gaps (21 case-scoped endpoints)

Inject `CaseService` into `EvolutionMcpAdapter`. Add `caseService.requireCaseAccess(caseId, action)` as the first call in each case-scoped method:

- READ endpoints (12): `requireCaseAccess(caseId, AclAction.READ)`
- WRITE endpoints (8): `requireCaseAccess(caseId, AclAction.WRITE)`
- ADMIN endpoints (2): `requireCaseAccess(caseId, AclAction.ADMIN)`

`getResearchCorpus()` — no caseId, no ACL check needed.

## Component 4: Test infrastructure

### Unit tests for `@AclCheck` annotation

Reflective scan of all API interfaces + EvolutionMcpAdapter verifying every method with a `caseId` parameter has an `@AclCheck` annotation. This prevents future endpoints from being added without ACL declarations.

### Integration tests for `AclEnforcementFilter`

Port the scaffold's `AclIntegrationTest` pattern into engine-rest's test suite:
- Deny-by-default: unauthenticated/unauthorized requests get 403
- Grant-then-allow: after granting access, requests succeed
- Cover all resource type / action combinations

Use `casehub-platform-acl-inmem` (already a test dependency) as the backing ACL provider.

### Existing service-level tests

`CaseServiceAclTest` already covers `requireCaseAccess()`. Add cases for `startCaseAndMap()` ACL check.

## Component 5: Follow-up — scaffold cleanup

Create issue on casehubio/scaffold to:
- Delete `AclRequestFilter.java`
- Delete `AclIntegrationTest.java`
- Delete `JwtAclTestProfile.java`
- Bump engine dependency to the version containing this change

## Existing infrastructure retained

- `AclEnforcementHealthCheck` — startup probe detecting permissive vs active ACL. No changes needed.
- `CaseScopeExtractor` — worker credential scope extraction. Orthogonal to this work.
- `AccessDeniedExceptionMapper` — already maps `AccessDeniedException` to 403. No changes needed.

## References

- `AclRequestFilter.java` (scaffold) — the pattern being replaced
- `CaseService.requireCaseAccess()` (rest-core:134-147) — existing service-level enforcement
- `AccessControlProvider` (platform-api) — SPI interface with permissive defaults
- `EngineResourceTypes` (engine-api) — resource type constants
- `no-defaultbean-multi-instance-spi` protocol — confirmed not applicable (1:1 SPI)
- scaffold#35 (closed) — demonstrated the temporary filter pattern
- engine#768 — this issue
