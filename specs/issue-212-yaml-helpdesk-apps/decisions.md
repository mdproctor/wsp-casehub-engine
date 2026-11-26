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
