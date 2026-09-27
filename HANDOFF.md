# HANDOFF — casehub-engine

## Last Session

Completed 2 of 5 recovery hardening issues on branch `issue-1182-snapshot-recovery-fallback`:

- **#1182** (S/Low) — `SnapshotRecoveryStrategy` now falls back to `EventLogReplayRecoveryStrategy` on null snapshot instead of throwing. Changed `EventLogReplayRecoveryStrategy` from `@IfBuildProperty` to `@Typed` so it's always available by concrete type. Also fixed pre-existing codegen bug (YamlGatePolicy `extra:` → `fields:`) and stale `EvolutionTicker` producer in `RuntimeBeans`.

- **#732** (M/Med) — Wired `CaseContextStoreFactory` through recovery path. First-principles analysis revealed `CaseContextImpl.storeFactory` is `final`, forcing the branching into the recovery service. Added `CaseContextImpl.loadFromStore(factory, caseId)` static method (calls `loadStore()` per layer). `DefaultWorkerExecutionRecoveryService` resolves factory from CaseInstance → CaseMetaModel → CaseDefinition; durable factories bypass the recovery strategy, volatile factories use existing path. Removed the `isDurable()` guard from `CaseHubRuntimeImpl.resolveFactory()`.

## Immediate Next Step

#1183 — harden `CaseRecoveryService.unfault`. Re-register evicted engine registries, fix JPA persistence race, document CANCELLED policy. The `.plan` has it as active.

## References

| Artifact | Path |
|----------|------|
| Design spec (#732) | `wksp/specs/issue-1182-snapshot-recovery-fallback/2026-09-27-factory-wired-recovery-design.md` |
| Decisions (#732) | `wksp/specs/issue-1182-snapshot-recovery-fallback/decisions.md` |
| Implementation plan (#732) | `wksp/plans/2026-09-27-factory-wired-recovery.md` |
| Recovery SPI spec | `wksp/specs/issue-1150-persistence-coherence/2026-09-23-case-context-recovery-strategy-design.md` |
| Parent epic | #210 — cancellation, timeout, and error recovery |
