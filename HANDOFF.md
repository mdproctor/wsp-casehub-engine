# Handoff — 2026-06-01

**Head commit (engine):** bd61239 — fix: propagate tenancyId in CaseLifecycleEvent (7 commits on issue-408-s-xs-batch)
**Head commit (workspace):** 98682bb — init(issue-408-s-xs-batch): scaffold workspace branch
**Both repos on:** issue-408-s-xs-batch
**PR open:** https://github.com/mdproctor/engine/pull/2

## What Changed This Session

**Completed all S/XS issues from the handover on a single branch:**

- engine#408 — tenancyId was null at WorkerExecutionStarted and WorkerStarted fire sites (QuartzWorkerExecutionJobListener, CaseContextChangedEventHandler)
- engine#407 — WorkerDecisionEvent gains tenancyId as 2nd component; CaseLedgerEventCapture and WorkerDecisionEventCapture fixed to actually set entry.tenancyId; Flyway V2002/V2003 shipped
- engine#410 — Defensive fallback in DefaultCaseDefinitionRegistry.getCaseDefinition() (root cause not found; WARN log added for diagnosis)
- engine#403 — WorkerDecisionEntry trust audit fields (trustScoreAtRouting, thresholdApplied); Flyway V2004; eliminates aml workaround
- engine#249 — SubCaseCompletionService fires SubCaseGroupLifecycleEvent CDI event for all group transitions
- engine#302+#325 — CaseHub.startCase(Object), HumanTaskTarget.claimDeadlineHours
- ADR-0004 — CaseMetaModel registry is global; tenancyId is sentinel
- engine#399, engine#400 — confirmed already implemented; closed

**Filed:** engine#411 (NOT NULL migration follow-up), casehub-aml#48 (V2005/V2006 reconciliation), engine#412 (ADR issue, now closed)

## Immediate Next Step

Merge PR #2 then run `/work` to start engine#404 (registry lifecycle design — L·High).

## Cross-Module

**We're breaking** (CaseLifecycleEvent.tenancyId addition from last session — consumers need recompile):
- `claudony` — claudony#143
- `devtown` — devtown#61
- `aml` — aml#47 + aml#48 (migration reconciliation)
- `clinical` — clinical#51

## What's Left

- engine#405 — @CrossTenant CDI producer (BLOCKED — needs system-actor principal) · S · Low
- engine#406 — DB-level RLS · M · High
- engine#411 — NOT NULL enforcement for tenancy_id columns in V2004 migrations · S · Low
- engine#410 — registry lookup root cause (defensive guard in place; still open) · M · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| engine#404 | Registry lifecycle analysis: eviction + stateless-on-rest + Quartz restart | L | High | Design-only; groundwork done |
| engine#383 | Oversight response loop | M | Med | Unblocked |
| engine#384 | PlanItem escalation state | M | Med | Unblocked |
| engine#387 | humanTask dynamic candidateGroups | M | Med | — |

## Key References

- Blog: `blog/2026-06-01-mdp01-fixes-a-mystery-and-three-migrations.md`
- ADR-0004: `proj/docs/adr/0004-casemetamodel-global-registry-sentinel-tenancyid.md`
- Garden entries: GE-20260601-ad6203 (Quarkus ARC identity chain), GE-20260601-33dd8e (@Alternative Priority external JAR), GE-20260601-2e31ae (GIT_SEQUENCE_EDITOR script)
- Protocols: PP-20260601-e368ea (SPI event tenancyId order), PP-20260601-90ace2 (IF NOT EXISTS migrations)
