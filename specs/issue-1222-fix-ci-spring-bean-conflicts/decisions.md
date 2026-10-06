## D1: CrossTenant passthrough bean removal

**Choice:** Remove the two identity-passthrough beans from RuntimeManualConfig entirely
**Alternatives:**
- Add @ConditionalOnMissingBean — doesn't work because RuntimeManualConfig loads before PersistenceAutoConfiguration
- Fix auto-configuration ordering — fragile, adds complexity for beans that serve no purpose in Spring
**Rationale:** The beans exist to satisfy a Quarkus @CrossTenant qualifier. Spring has no equivalent. PersistenceAutoConfiguration provides the real JPA implementations.
**Trade-offs:** None — the beans are pure noise in Spring
**Sources:** RuntimeManualConfig:403-414, PersistenceAutoConfiguration:86-95
**Exploration:** quick
**Status:** captured

## D2: @PlatformStream annotation restoration

**Choice:** Restore @PlatformStream on EngineCaseApi.caseStream() and EnginePlanApi.executionStateStream()
**Alternatives:**
- Leave removed — SSE endpoints wouldn't be auto-generated into REST resources
**Rationale:** Platform generators (#1214) now properly handle Flow.Publisher return types. The removal (3a77d000f) was a workaround that's no longer needed. Without restoration, SSE endpoints won't be generated.
**Trade-offs:** None — this restores intended functionality
**Sources:** 3a77d000f (removal commit), #1214 (generator fix)
**Exploration:** quick
**Status:** captured

## D3: Spring-integration-test exclude path correction

**Choice:** Fix the exclude paths in application.properties to match actual class packages
**Alternatives:**
- Leave wrong paths — RuntimeManualConfig loads unexcluded, causing bean conflicts
**Rationale:** Commit 250b4ae12 added excludes with wrong package (io.casehub.engine.internal.spring instead of io.casehub.engine.runtime.spring). RuntimeAutoConfiguration doesn't exist — remove that line entirely.
**Trade-offs:** None
**Sources:** spring-integration-test/src/test/resources/application.properties:7-8
**Exploration:** quick
**Status:** captured

## D4: HybridOrchestrationIntegrationTest flaky fix

**Choice:** Add @BeforeEach cache.clear() for test isolation
**Alternatives:**
- Increase timeouts — user explicitly rejected; doesn't address root cause
- Use unique case names per test run — unnecessary; cache pollution is the actual issue
**Rationale:** CaseInstanceCache is an @ApplicationScoped ConcurrentHashMap singleton shared across all @QuarkusTest classes. Without cleanup, stale case instances from prior tests cause Vert.x event bus contention (VertxEventDispatcher publishes all events async) and delay evaluations via the CaseEvaluationSerializer. The spawnAndAwaitCase flow has a 10s internal timeout that can be hit when stale evaluations occupy event loop threads. 6 other test classes in the module use this exact pattern successfully.
**Trade-offs:** None — matches established codebase pattern
**Sources:** OrchestrationTest:58-61, CaseWaitingResumeTest:56-59, CaseInstanceCacheImpl:41-43
**Exploration:** quick
**Status:** captured
