# HANDOFF — casehub-engine

## Last Session (2026-10-26)

Cross-repo fix from devtown session (casehubio/devtown#262). 4 commits on main:

- **d1d3c025b** — `feat(#262): add EngineMappers utility with JavaTimeModule` — new `EngineMappers.defaultMapper()` factory in common-core
- **58e067409** — `refactor(#262): replace bare ObjectMapper in common-core` — 11 files (common-core + common)
- **e63f6e450** — `refactor(#262): replace bare ObjectMapper in runtime-core` — 22 files (fixes WorkerScheduleEventHandler Instant serialization)
- **ed62da2e2** — `refactor(#262): replace bare ObjectMapper in planning-core and annotations` — 10 files

All 43 `new ObjectMapper()` instances in engine production code now use `EngineMappers.defaultMapper()` (with JavaTimeModule), except annotations/runtime which registers JavaTimeModule inline (no common-core dependency).

## Previous Session (2026-10-18)

Merged all open PRs, fixed CI, cleaned up stale work-end state.

### Completed

- **PR #1217** (fix #1216) — Prevent silent judgment drop when CasePlanModel evicted. PlanItemStore fallback. Test fix PR #1229 also merged.
- **PR #1212** (fix #1205) — FIFO queue for case evaluations, ReentrantReadWriteLock for DefaultCaseDefinitionRegistry, reset serialization. Landed as PR #1230 (rebased from treblereel's fork).
- **PR #1196** — Explicit `@ConditionalOnMissingBean(DeadLetterEntryStore.class)` in ResilienceManualConfig. Landed as PR #1231 (rebased from treblereel's fork).
- **CI fix** — Removed re-added DefaultTestPrincipal CDI producer that caused Quarkus augmentation deadlock. CI green (36m26s) for first time since Oct 7.
- **Blog** — Published "The Conductor Finds Its Audience" to mdproctor.github.io.
- **Cleanup** — Removed stale `.plan` and `.work-progress` from #1180 (soredium#461 filed for the root cause).

### State

- Main branch clean, CI green, zero open PRs
- No active work in progress

### Next

- **#1235** — Replace ChatModelProvider stack with platform AgentProvider

## References

| Artifact | Path |
|----------|------|
| soredium bug | Hortora/soredium#461 — work-end closing:stamped corruption |
| #1205 spec | `docs/specs/issue-1205-concurrent-registry-reregistration/` |
