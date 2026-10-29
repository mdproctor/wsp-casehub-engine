# Design Journal — issue-1262-playbook-failure-triage

## 2026-10-28 — Session 1: Design + Batch 1-2 implementation

**Scope:** #1262 (structured playbook failure + triage wiring) + #1263 (PlaybookWorkerFunctionProvider) folded into one branch.

**Design phase:** Brainstorm → 7 decisions captured → standard decision review (3 rounds, 29 issues) → spec written → light post-spec review (15 issues, 5 HIGH addressed). Key review findings: failure propagation via PlaybookExecutionException (not HandlerResult parameter), step data in FlowExecutionRegistry (not SWF metadata), triage inside DefaultRecoveryCoordinator (not RetryOrchestrator), Map-convention modifications (not typed SPI).

**Implementation:** 8 of 14 tasks complete across Batch 1 (PlaybookProvider) and Batch 2 (StructuredFailure). SWFWorkerFunction sealed interface, PlaybookWorkerFunction with CompilationResult, PlaybookWorkerFunctionProvider, PlaybookExecutionResult types, FlowExecutionRegistry step data, PlaybookExecutionListener, PlaybookExecutionException, HandlerResult enrichment.

**Remaining:** Batch 3 (TriageWiring — 5 tasks: RecoveryPolicy/RecoveryOverride triage fields, BindingRecoveryState tracking, DefaultRecoveryCoordinator integration, PlaybookTriageModification parser, retry count partitioning) + Batch 4 (CBRRecording — 1 task: PlaybookOutcomeRecorder).

**Key decision:** Platform playbook compiler is on 7.x (serverlessworkflow-types 7.35.3.Final), compatible with engine's 7.25.1.Final. No 8.x/Jackson 3 migration until Quarkus 4.x.
