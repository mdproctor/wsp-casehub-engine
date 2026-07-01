---
layout: post
title: "REJECTED is not FAULTED"
date: 2026-05-26
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-engine]
tags: [casehub, state-machine, jpa, otel]
---

Four correctness fixes shipped today as a single PR. Three are mechanical — a PlanItem stuck in RUNNING when Quartz retries exhausted, an atomic counter race in the JPA repository, an OTel trace ID always null because `@ObservesAsync` severs thread-local context. Each had a clear root cause and a clean fix. The fourth took longer to think about.

`WorkItemLifecycleAdapter` was mapping both `WorkItemStatus.REJECTED` and `WorkItemStatus.EXPIRED` to `PlanItem.markFaulted()`. The same terminal state for two semantically different outcomes: someone explicitly refused the work, and a deadline passed without resolution. Case definitions had no way to tell which happened. You can't build "offer to a different group if rejected" when rejection and timeout are the same signal.

The fix adds `PlanItemStatus.REJECTED` as a first-class state alongside `FAULTED` and `CANCELLED`. `markRejected()` transitions from `DELEGATED` only — which is the right constraint. REJECTED arrives via human task refusal or M-of-N group threshold failure; both paths go through `HumanTaskScheduleHandler.markDelegated()` before the WorkItem is created. A CapabilityTarget PlanItem in `RUNNING` cannot be rejected — it can only fault via retry exhaustion.

There's a secondary effect: stage autocomplete previously only fired when all required items reached `COMPLETED`. With `REJECTED` as a new terminal state, the question is whether a stage should autocomplete when items fault or get rejected. I decided yes — a stage where all required items have settled has concluded regardless of how they settled. Case logic downstream can inspect what happened. Leaving stages stuck because one item faulted requires manual operator intervention, which is the wrong default. ADR-0002 captures this.

The OTel problem is worth noting separately because it's easy to reproduce in any Quarkus codebase that fires CDI events asynchronously. `CaseLedgerEventCapture.onCaseLifecycleEvent()` is `@ObservesAsync`, which means it runs on a managed executor thread. OTel span context lives in a `ThreadLocal`. The `@PrePersist` enricher that populates `CaseLedgerEntry.traceId` calls `LedgerTraceIdProvider.currentTraceId()` — and gets nothing, because there's no span on the async thread.

The fix captures the trace ID synchronously before `fireAsync()` and carries it in the `CaseLifecycleEvent` record. Seven arguments instead of six. `TraceIdEnricher` guards with `if (entry.traceId != null) return;` — so the pre-captured value survives `@PrePersist`. We updated all six fire-sites across the runtime handlers and the Quartz job listener.

The code review came back with two things we'd missed. `cancelPlanItemOnRejected()` in `SubCaseCompletionService` marks the PlanItem `CANCELLED` when a grouped SubCase group fails to reach threshold — technically correct (it's being stopped as part of case cancellation, not refused by an external actor) but inconsistent enough with the new model to need an explicit comment. The other: `PlanItemCompletedEvent`'s Javadoc didn't say it fires only on `COMPLETED`, not on `REJECTED` or `FAULTED`. Future observers would assume otherwise.