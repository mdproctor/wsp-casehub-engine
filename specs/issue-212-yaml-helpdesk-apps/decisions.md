# Decisions — persistence-json (#1366)

## D1: Serialization strategy for CaseInstance in JSON collections

**Choice:** Context snapshot + metadata serialization — store UUID, state, version, case metadata coordinates (namespace/name/version), context snapshot (JsonNode), and other serializable scalar fields. Reconstruct CaseInstance on retrieval using CaseDefinitionRegistry to resolve the definition from metadata coordinates.

**Alternatives:**
- Full Jackson serialization with MixIns — tries to serialize entire CaseInstance object graph, excluding non-serializable fields via MixIns. Riskier: CaseInstance has deep object graphs (CaseContext with lambda evaluators, ReentrantLock) and the exclusion list could be fragile as the class evolves.

**Rationale:** CaseInstance has runtime-only state (ReentrantLock, lambda-based expression evaluators in CaseContext) that cannot survive JSON round-tripping. The snapshot approach is explicit about what's persisted vs transient, and CaseInstance already has a `contextSnapshot` field designed for exactly this purpose.

**Trade-offs:** Requires CaseDefinitionRegistry to be available at retrieval time. Some transient state (pendingActionGate, waitingForWorkId) may be lost across restarts — acceptable for v1 zero-Java apps.

**Sources:**
- `common-core/src/main/java/io/casehub/engine/common/spi/cache/CaseInstanceCache.java` — 4-method SPI
- `common-core/src/main/java/io/casehub/engine/common/internal/model/CaseInstance.java` — domain object with contextSnapshot field
- `runtime-core/src/main/java/io/casehub/engine/runtime/engine/cache/CaseInstanceCacheImpl.java` — reference in-memory implementation
- `pages/backend/data-json/.../JsonCollectionStore.java` — backing store API

**Exploration:** quick
**Status:** captured

## D2: Module placement

**Choice:** New `persistence-json` module in `casehub-engine`, sibling to `persistence-memory` and `persistence-hibernate`. Follows existing module naming convention.

**Alternatives:**
- Put in `persistence-memory` as an alternative implementation — conflates in-memory and persistent concerns, makes dependencies messy.
- Put in `runtime` — too broad, persistence modules are intentionally separate.

**Rationale:** Follows the established pattern of one persistence module per backing store. Each module is independently deployable on the classpath.

**Trade-offs:** One more module to maintain, but the boundary is clean.

**Sources:**
- `persistence-memory/pom.xml` — reference module structure
- `persistence-hibernate/pom.xml` — reference module structure

**Exploration:** quick
**Status:** captured

## D3: AgentTaskBindingResolver — primary role

**Choice:** Validation at definition time — validates during case definition registration that an agent with the required capability/domain exists in the AgentRegistry. Catches misconfigurations before the case runs. Lightweight query only, no state.

**Alternatives:**
- Pre-resolution with caching — eagerly resolves to a specific AgentDescriptor at startup. Adds stale-cache concerns and complicates agent hot-reload.
- Binding enrichment — augments AgentTaskTarget with eidos metadata. No validation benefit; the handler already resolves everything it needs at execution time.

**Rationale:** The execution chain (AgentWorkerFunctionHandler.executeByCapability) already handles full runtime resolution. The gap is early detection of misconfigured bindings — a capability no agent provides should be caught at startup, not when the case reaches that worker step.

**Trade-offs:** Validation is advisory (warnings, not hard failures) since agents may be registered dynamically after case definitions. A hard failure would break legitimate late-binding scenarios.

**Sources:**
- `engine/runtime/.../AgentWorkerFunctionHandler.java:203-254` — existing execution-time resolution
- `eidos/api/.../AgentRegistry.java` — query API
- `engine/runtime/.../YamlCaseHubDiscovery.java` — precedent for startup observer pattern

**Exploration:** quick
**Status:** captured

## D4: AgentTaskBindingResolver — integration approach

**Choice:** CDI startup observer in eidos-runtime (`resolver/` package). Observes startup after all registrations complete (high priority), iterates CaseDefinition bindings with AgentTaskTarget, queries AgentRegistry for each binding's capability/domain. Logs warnings for unresolvable bindings. No new engine SPI needed.

**Alternatives:**
- Engine SPI with eidos implementation — new BindingValidator SPI in engine common-core. Cleaner separation but adds engine-side coupling for a single implementation, and the engine would need to know about binding validation at registration time.

**Rationale:** Follows the established pattern of startup observers (YamlCaseHubDiscovery at priority 2600). The resolver observes at a later priority (after YamlCaseHubDiscovery creates CaseHubs). No engine changes required — eidos depends on engine API, not the other way around.

**Trade-offs:** No engine-side enforcement — validation is purely eidos-side. If eidos isn't on the classpath, no validation occurs. This is acceptable because agent-task bindings are meaningless without eidos.

**Depends on:** D3 (resolver role determines integration approach)

**Sources:**
- `engine/runtime/.../YamlCaseHubDiscovery.java` — startup observer precedent at priority 2600
- `eidos/runtime/resolver/EidosAgentResolver.java` — existing resolver pattern in eidos

**Exploration:** quick
**Status:** captured
