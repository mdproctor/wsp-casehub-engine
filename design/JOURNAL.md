# Design Journal — issue-651-cross-repo-blocker-batch

## §Session-1 — 2026-07-04/05: Design, review, and partial implementation

### Key decisions

1. **Repository naming convention: full rename sweep** — chose to rename `CaseInstanceRepository` → `ReactiveCaseInstanceRepository` AND `EventLogRepository` → `ReactiveEventLogRepository` (plus cross-tenant variants) rather than just adding `BlockingCaseInstanceRepository`. Rationale: PlanItemStore/ReactivePlanItemStore establishes unqualified=blocking; fixing one while leaving another inconsistent makes things worse. Also filed issue for `CaseMetaModelRepository` and `SubCaseGroupRepository` (same inconsistency, deferred).

2. **AgentAssignment rationale: mandatory, not nullable** — every routing strategy must explain its decision. No zero-arg factory methods. Rationale propagates through `TrustCandidateClassifier.ScoredCandidate` so strategies set it at construction time. Each strategy has per-phase format strings (QUALIFIED vs BOOTSTRAP).

3. **CaseEventRecorder: separate SPI, not on CaseHubRuntime** — CaseHubRuntime already has 14 methods. The event write is an infrastructure concern (blocks#12 records orchestration events), not a case lifecycle operation. Follows dual-stack convention: blocking `CaseEventRecorder` + `ReactiveCaseEventRecorder`. Uses `CaseEventRequest` record (not 7 loose params). No-ops in `api/spi/` (not `runtime/`) following NoOpPlanItemStore pattern.

4. **CI dispatch: complete list, not just blocks** — added all 6 repos that depend on engine packages (blocks, soc, iot, clinical, quarkmind, ops), not just blocks as the issue requested.

5. **tenancyId not tenantId** — design review caught the naming inconsistency. The codebase universally uses `tenancyId` (CaseInstance.tenancyId, every repository parameter). The original issue said `tenantId`.

### Implementation progress

Tasks 1–5 of 10 complete. The repository rename (Task 5) was the riskiest — IntelliJ's `ide_refactor_rename` corrupted ~15 `@Inject` annotations when field declarations had whitespace alignment padding. All fixed manually. Garden entry submitted (GE-20260705-e15dde).

### What's left for next session

Tasks 6–10: create blocking interfaces, split implementations into separate blocking/reactive classes, DefaultCaseEventRecorder implementation, consumer repo migration (8 repos), CI dispatch update. quarkmind is on a branch (not main) — skip its migration.
