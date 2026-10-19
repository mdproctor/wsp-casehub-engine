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

Previously, ACL was intentionally a deployment concern — each consumer owned its own enforcement. This design reverses that decision: engine-rest now owns ACL enforcement centrally, because the duplication has proven fragile (22 evolution endpoints shipped with zero protection).

## Solution

Two-layer ACL enforcement owned by engine-rest:

1. **Service-level checks** (primary) — `requireCaseAccess()` / `canAccess()` in every service method. Protects ALL invocation paths: REST, MCP, and any future transport.
2. **`AclEnforcementFilter`** (`@ServerRequestFilter`) — defense-in-depth for the REST/HTTP path only. Does NOT protect MCP invocations.
3. **`@AclCheck` annotation** on every API method — declares the resource type, action, and which path param holds the resource ID. Consumed by the filter.

The service layer is the authoritative enforcement point. The filter is a safety net — if a service method is missing its check, the filter catches it at the HTTP boundary. Both layers are intentional; the double `canAccess()` call on requests with a `resourceIdParam` is deliberate defense-in-depth, not accidental duplication.

After this lands, scaffold deletes its `AclRequestFilter` (tracked as scaffold#69).

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
- `action` — READ, WRITE, or ADMIN (CLAIM is not used by any REST endpoint currently)
- `resourceIdParam` — name of the `@PathParam` holding the resource ID. Empty for list endpoints and endpoints where the resource ID is composite (namespace/name/version) or absent.

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

**EvolutionMcpAdapter (22 case-scoped + 1 global = 23 methods):**

| Method | resourceType | action | resourceIdParam |
|--------|-------------|--------|----------------|
| getTickHistory | CASE | READ | caseId |
| getDenyPatterns | CASE | READ | caseId |
| getSummary | CASE | READ | caseId |
| getInbox | CASE | READ | caseId |
| getReadinessReport | CASE | READ | caseId |
| getEvolutionState | CASE | READ | caseId |
| getResearchCorpus | (none) | — | — |
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

`getResearchCorpus` has no `caseId` — it's a global corpus query, intentionally unauthenticated at the ACL level. Authentication is still required (handled by the platform's identity layer). No `@AclCheck` annotation.

**Totals:** 44 annotated methods + 1 unannotated (`getResearchCorpus`) = 45 methods across 6 classes.

## Component 2: `AclEnforcementFilter`

**Module:** `casehub-engine-rest`
**Package:** `io.casehub.engine.rest.filter`

A Quarkus `@ServerRequestFilter` that reads `@AclCheck` from the matched resource method and enforces ACL at the HTTP layer.

### Annotation discovery

REST routes are generated by `casehub-platform-graphql-generator` from API interface annotations. `ResourceInfo.getResourceMethod()` returns the **generated class method**, not the interface method. Since `@AclCheck` is declared on the interface, the filter must walk the interface hierarchy to find it:

1. Call `resourceInfo.getResourceMethod()` to get the generated class method.
2. Check the method directly for `@AclCheck`. If found, use it.
3. If not found, iterate `resourceInfo.getResourceClass().getInterfaces()`.
4. For each interface, find the matching method (same name and parameter types).
5. Check the interface method for `@AclCheck`.

For `EvolutionMcpAdapter`, annotations are on the concrete class methods directly (no interface), so step 2 finds them immediately.

### Behaviour

1. Discover `@AclCheck` via the interface-walking mechanism above. If absent, pass through.
2. Get `actorId` from `CurrentPrincipal`.
3. If `resourceIdParam` is non-empty: extract resource ID from `UriInfo.getPathParameters()`, construct `ResourceId`, call `accessControlProvider.canAccess()`. Deny returns 403.
4. If `resourceIdParam` is empty: pass through to the service layer. The filter cannot construct a `ResourceId` for composite-key or list endpoints, so enforcement is deferred to the service layer.

Note: the filter does NOT call `accessibleResources()` for empty-param WRITE/ADMIN operations (like `startCase`). `accessibleResources()` defaults to `List.of()` on the no-op provider, which would incorrectly deny all requests in deployments without a real ACL provider. The service layer handles these checks using `canAccess()` (which defaults to `true`).

### Failure mode

Fail-closed. If `canAccess()` throws, the request is denied with 500 (not fail-open like the scaffold's filter). Security enforcement should not silently allow on error.

### Invocation paths

The filter intercepts HTTP/REST requests only. `@McpDomain` endpoints can also be invoked via MCP tool calls, which bypass JAX-RS entirely. Service-level checks (Component 3) are the only protection for MCP invocations. This is why the service layer is the primary enforcement point, not the filter.

### Dependencies

- `AccessControlProvider` — injected via `@Inject`. The platform interface has all-permissive defaults (`canAccess()` returns `true`), so deployments without a real ACL provider pass everything through. No `Instance<>` or `@DefaultBean` needed — the interface defaults already provide the no-op behaviour.
- `CurrentPrincipal` — injected via `@Inject`.

## Component 3: Service-level gap fills

Service-level checks protect ALL invocation paths (REST + MCP). This is the authoritative enforcement layer.

### Core API gaps (5 endpoints)

**`CaseService.startCaseAndMap()`** — add WRITE check on CASE_DEFINITION before starting the case. Construct `ResourceId` from `request.namespace() + "/" + request.name()`. Use `canAccess()` (not `accessibleResources()`).

**`CaseDefinitionService.getDefinitionsByName()`** — add inline ACL filtering (same pattern as `listDefinitions()`). Filter results through `accessControlProvider.canAccess()` with CASE_DEFINITION / READ.

**`CaseDefinitionService.getDefinitionByKey()`** — add point check. Construct `ResourceId` from namespace/name/version, call `canAccess()`, throw `AccessDeniedException` on deny.

**`DefaultEngineCaseApi.caseStream()`** — add `caseService.requireCaseAccess(caseId, AclAction.READ)` before returning the SSE publisher. Note: ACL is checked once at connection time. If access is revoked while the stream is active, events continue until the connection closes. This is a known TOCTOU limitation, acceptable because SSE connections are short-lived and revocation is rare.

**`DefaultEnginePlanApi.executionStateStream()`** — same pattern: add `requireCaseAccess` before returning the publisher. Same TOCTOU limitation applies. Inject `CaseService` into `DefaultEnginePlanApi`.

### EvolutionMcpAdapter gaps (21 case-scoped endpoints)

Inject `CaseService` into `EvolutionMcpAdapter`. Add `caseService.requireCaseAccess(caseId, action)` as the first call in each case-scoped method:

- READ endpoints (13): `requireCaseAccess(caseId, AclAction.READ)`
- WRITE endpoints (8): `requireCaseAccess(caseId, AclAction.WRITE)`
- ADMIN endpoints (2): `requireCaseAccess(caseId, AclAction.ADMIN)`

`getResearchCorpus()` — no caseId, global query. No ACL check. Authentication is still enforced by the platform identity layer.

### ResourceId format convention

CASE resources use `caseId.toString()` as the resource ID (UUID string).
CASE_DEFINITION resources use `namespace/name` or `namespace/name/version` as the resource ID (slash-separated composite). This matches the existing convention in `CaseDefinitionService.listDefinitions()` and scaffold's `AclRequestFilter`.

## Component 4: Test infrastructure

### Annotation coverage test

Reflective scan of all API interfaces + EvolutionMcpAdapter verifying:
- Every method with a `@PathParam UUID caseId` parameter has an `@AclCheck` annotation
- Every `@AclCheck` annotation has a valid `resourceType` and `action`

This prevents future endpoints from being added without ACL declarations. The test scans interfaces (for API contracts) and concrete classes (for `EvolutionMcpAdapter`).

### Integration tests for `AclEnforcementFilter`

Port the scaffold's `AclIntegrationTest` pattern into engine-rest's test suite:
- Deny-by-default: unauthorized requests get 403
- Grant-then-allow: after granting access, requests succeed
- Cover all resource type / action combinations

Use `casehub-platform-acl-inmem` (already a test dependency) as the backing ACL provider.

### Existing service-level tests

`CaseServiceAclTest` already covers `requireCaseAccess()`. Add cases for `startCaseAndMap()` ACL check.

## Component 5: Follow-up — scaffold cleanup

Tracked as scaffold#69:
- Delete `AclRequestFilter.java`
- Delete `AclIntegrationTest.java`
- Delete `JwtAclTestProfile.java`
- Bump engine dependency to the version containing this change

## Existing infrastructure retained

- `AclEnforcementHealthCheck` — startup probe detecting permissive vs active ACL. Note: the existing probe passes `null` for `AclAction`, which works with the interface default but may NPE with real providers that don't guard for null. Consider fixing in this PR if touched, but not in scope.
- `CaseScopeExtractor` — worker credential scope extraction. Orthogonal to this work.
- `AccessDeniedExceptionMapper` — already maps `AccessDeniedException` to 403. No changes needed.
- `EngineResourceTypes.PLAN_ITEM` and `EVENT_LOG` constants — not used by any `@AclCheck` annotation (plan and event endpoints use CASE as the resource type, gated by caseId). These constants exist for future fine-grained ACL and are retained without change.

## References

- `AclRequestFilter.java` (scaffold) — the pattern being replaced
- `CaseService.requireCaseAccess()` (rest-core:134-147) — existing service-level enforcement
- `AccessControlProvider` (platform-api) — SPI interface with permissive defaults
- `EngineResourceTypes` (engine-api) — resource type constants
- scaffold#35 (closed) — demonstrated the temporary filter pattern
- scaffold#69 — follow-up to delete scaffold's filter
- engine#768 — this issue
