# Decisions — engine#768: Move ACL checks into engine-rest SPI

## D1: Scope — fill gaps AND add filter

**Choice:** Two-layer ACL: fill the 4 service-level gaps (startCase, getDefinitionsByName, getDefinitionByKey, caseStream) AND add a built-in `@ServerRequestFilter` in engine-rest for defense-in-depth.
**Alternatives:**
- Fill gaps only — simpler, but consumers still lack HTTP-layer protection unless they write their own filter
- Filter only — doesn't fix the service-level gaps, filter would be the sole enforcement point
**Rationale:** engine-rest is a library used by multiple consumers (scaffold and future deployments). Service-level checks are the primary enforcement; the filter provides defense-in-depth so no consumer needs to write their own. Scaffold deletes its `AclRequestFilter` once this lands.
**Trade-offs:** Slight runtime overhead from checking ACL twice (filter + service). Acceptable for security.
**Sources:** scaffold `AclRequestFilter.java`, `CaseService.requireCaseAccess()`, issue #768 body
**Exploration:** quick
**Status:** captured

## D2: Filter implementation — @ServerRequestFilter

**Choice:** Quarkus `@ServerRequestFilter` annotation (post-matching, injectable)
**Alternatives:**
- `@PreMatching ContainerRequestFilter` — scaffold's approach; requires manual URL parsing and `CDI.current()` lookups because injection doesn't work in pre-matching filters
- Annotation-driven `@AclCheck` with custom interceptor — more infrastructure than needed
**Rationale:** `@ServerRequestFilter` runs post-matching, so path params are available from `UriInfo`. Supports normal `@Inject`, eliminating the CDI.current() hack. Quarkus-native, clean.
**Trade-offs:** Quarkus-specific (not portable JAX-RS). Acceptable since engine-rest already depends on Quarkus.
**Sources:** scaffold `AclRequestFilter.java` (lines 58-62 show CDI.current() pain)
**Exploration:** quick
**Status:** captured

## D3: ACL mapping — metadata annotations on API methods

**Choice:** Add `@AclCheck(resourceType, action, resourceIdParam)` annotation on API interface methods. The filter reads this annotation at runtime to determine what to check.
**Alternatives:**
- URL path convention (GET=READ, POST=WRITE) — too coarse; cancel-case needs ADMIN not WRITE
- Hardcoded mapping table in filter — fragile, duplicates route knowledge, breaks on endpoint changes
**Rationale:** Type-safe, single source of truth. Each API method declares its ACL requirement alongside its `@McpDomain` metadata. Adding a new endpoint forces the author to declare its ACL requirement.
**Trade-offs:** New annotation to maintain. Minor — it's a simple marker annotation.
**Sources:** `EngineCaseApi.java`, `EngineCaseControlApi.java` (existing `@McpDomain`/`@PlatformQuery`/`@PlatformMutation` pattern)
**Exploration:** quick
**Status:** captured

## D4: Annotation module — engine-api

**Choice:** `@AclCheck` annotation lives in `casehub-engine-api` module
**Alternatives:**
- `platform-api` — makes it platform-wide but adds platform dependency for an engine concern
- New `engine-rest-api` module — clean separation but unnecessary module overhead for one annotation
**Rationale:** API interfaces are already in `engine-api` with `@McpDomain`. Co-locating `@AclCheck` keeps all REST API metadata in one module.
**Trade-offs:** If non-engine REST surfaces need ACL annotations later, they'd either import engine-api or duplicate the annotation. Cross that bridge when we get to it.
**Sources:** `engine-api` pom.xml, `EngineCaseApi.java`
**Exploration:** quick
**Depends on:** D3 (annotation approach)
**Status:** captured

## D6: Scope expanded — EvolutionMcpAdapter has 22 unprotected endpoints

**Choice:** Include all 27 gaps (5 core + 22 evolution) in this issue, not just the original 4.
**Alternatives:**
- Split evolution into a separate issue — delays full ACL coverage, leaves a security gap
**Rationale:** The `@AclCheck` annotation + filter approach handles all endpoints uniformly. Adding annotations and `requireCaseAccess` calls to evolution methods is mechanical work, not a design change. One issue, one PR, complete coverage.
**Trade-offs:** Larger PR. Acceptable — the changes are repetitive and low-risk.
**Sources:** Full endpoint audit (fork)
**Exploration:** quick
**Status:** captured

## D5: Follow-up issue for scaffold cleanup

**Choice:** Create a follow-up issue on casehubio/scaffold to delete `AclRequestFilter`, `AclIntegrationTest`, and `JwtAclTestProfile` once engine#768 lands.
**Alternatives:** None — scaffold cleanup must happen but is a separate concern.
**Rationale:** Scaffold can't be cleaned up until engine-rest has complete ACL coverage. Separate issue keeps the work tracked.
**Sources:** scaffold `AclRequestFilter.java`, scaffold#35 (closed)
**Exploration:** quick
**Status:** captured
