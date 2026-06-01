---
layout: post
title: "Fixes, a mystery, and three migrations"
date: 2026-06-01
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-engine]
tags: [multi-tenancy, flyway, quarkus, cdi, casehub]
---

A batch of eight S/XS issues, all connected by the multi-tenancy foundation laid in the previous sprint. Most were straightforward — but one required an ADR.

Claude and I worked through the batch together on a single branch. The pattern was consistent: a field that should carry `tenancyId` wasn't getting it, either because it was fired before the principal context was established, or because the record's constructor didn't thread it through.

**The first two** (`engine#408` and `engine#407`) were propagation gaps. `WorkerExecutionStarted` and `WorkerStarted` were fired from `QuartzWorkerExecutionJobListener` and `CaseContextChangedEventHandler` — both sites where the principal wasn't in scope. `WorkerDecisionEvent` needed `tenancyId` added as a second component, which meant touching `CaseLedgerEventCapture` and `WorkerDecisionEventCapture` to actually set `entry.tenancyId`. Two Flyway migrations (V2002 and V2003) added the columns to `worker_decision_entry` and `case_ledger_entry`.

**`engine#403`** added trust audit fields to `WorkerDecisionEntry`: `trustScoreAtRouting` and `thresholdApplied`. V2004. The point of these fields is to eliminate a workaround in the AML module — it had been reconstructing this information from context at query time. Now it's written at routing time and the reconstruction goes away.

**`engine#249`** closed a gap in the SubCase completion path. `SubCaseCompletionService` was handling group transitions (IN_PROGRESS, COMPLETED, REJECTED) but firing them directly into application code. The fix was straightforward: fire a CDI `Event<SubCaseGroupLifecycleEvent>` instead, so observers — monitoring, audit, Claudony dashboard — can subscribe without coupling to the engine. `fireAsync()` with `Uni.awaitTermination()` to preserve the non-blocking contract.

**`engine#302` and `engine#325`** were API improvements: `CaseHub.startCase(Object)` as a convenience overload that serialises the input via Jackson (callers no longer need to convert to `JsonNode` themselves), and `humanTask.claimDeadlineHours` for work adapter HumanTask bindings.

**`engine#410`** was the mystery. The issue was a `NullPointerException` in `DefaultCaseDefinitionRegistry.getCaseDefinition()` when a definition wasn't found. The root cause was never found — we added a defensive fallback and a `WARN` log. What the investigation did surface was a more important question: what are the right semantics for a multi-tenant registry where `CaseMetaModel` is shared across tenants?

The answer became ADR-0004: the registry is global. CaseMetaModel records are owned by no particular tenant — they're shared reference data. `tenancyId` on `CaseMetaModel` is a sentinel (`__system__`), not a real tenant boundary. This distinction matters downstream: case instances are tenant-scoped; the case definitions they run are not.

Two more issues (`engine#399` and `engine#400`) turned out to be already implemented and were closed without changes.
