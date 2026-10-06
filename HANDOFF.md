# HANDOFF — casehub-engine

## Last Session

Landed #1218 (fix @DefaultBean producers for new engine subsystems) and partially fixed CI. CI is red — #1222 tracks the remaining fix.

### Completed

- **#1218** (M/Med) — Fixed 13 unsatisfied CDI dependencies breaking all consumer apps. Added `Instance<>` wrapping for optional stigmergy/improvement/qhorus deps, no-op constructors with `active` flag guards on `SwarmProvisioner` and `EvolutionTicker`, `@DefaultBean NoOpEngineEvolutionApi`, and `ConvergenceDetector`/`TickTraceBuffer` producers. Also fixed yaml-cbr compilation errors from PluginStep API change. Landed as `7beae492b` on main.

- **CI fix (partial)** — Landed caps-api dependency, rag expansion mode config, compensation event types, and `@PlatformStream` annotation removal. These fixed 21 of 22 test failures.

### Remaining — #1222

One CI failure remains: `SpringBootCompositionTest` fails with `BeanDefinitionOverrideException` for `crossTenantCaseInstanceRepository`. 

**Root cause:** `RuntimeManualConfig` defines identity passthrough beans for `crossTenantCaseInstanceRepository` and `crossTenantEventLogRepository` (satisfy Quarkus `@CrossTenant` qualifier — unnecessary in Spring). `PersistenceAutoConfiguration` defines the real JPA-backed beans. Spring rejects the duplicate.

**Fix:** Remove the two passthrough `@Bean` methods (lines ~403-416) from `RuntimeManualConfig.java`. The persistence module provides the real beans. This is a one-line deletion — the IntelliJ edit was applied but not persisted to disk.

```
File: runtime-spring/src/main/java/io/casehub/engine/runtime/spring/RuntimeManualConfig.java
Remove: the crossTenantEventLogRepository and crossTenantCaseInstanceRepository @Bean methods at the end of the class
```

## References

| Artifact | Path |
|----------|------|
| CI fix issue | casehubio/engine#1222 |
| CDI fix commit | `7beae492b` on main |
| CI fix commit | quick-fix commit on main (caps-api, rag mode, compensation types, @PlatformStream) |
