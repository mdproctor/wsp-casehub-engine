---
layout: post
title: "Three Gaps, One Branch"
date: 2026-07-01
entry_type: note
subtype: diary
projects: [casehub-engine]
tags: [signal-settlement, planning-strategy, event-log, audit]
---

The hybrid orchestration epic shipped last session. What it left behind were three follow-up issues — not defects exactly, more like threads that were cut short when the scope boundary was drawn.

The most interesting one was #620: `signalAndAwait()` hangs when a worker throws and retries are exhausted. I knew signalId threaded through the success path — `WorkflowExecutionCompleted` carries it, the handler calls `recordCompletion()`, settlement resolves. What I hadn't traced was the failure path. `WorkerRetryContext` had no `signalId` field. The entire retry/exhaust flow was invisible to the settlement tracker.

The fix itself was mechanical: add the field, thread it through the Quartz JobDataMap for retries, through `WorkerRetriesExhaustedEvent` for exhaustion, and have the handler call `recordCompletion()`. But the investigation turned up a second gap — the guard quarantine path in `WorkerScheduleEventHandler` publishes `WorkerRetriesExhaustedEvent` when a worker is blocked, and that path also dropped signalId despite `event.signalId()` being right there.

While touching `WorkerRetriesExhaustedEvent`, I noticed it had `tenancyId` at position 5. The `spi-event-tenancyid-component-order` protocol says position 2. Breaking change to a record constructor — every call site needs updating. No end users, so the migration was just grep-and-fix across test files.

The second issue (#621) was a small perf fix that taught me something about `DefaultCasePlanModel`. `SequentialPlanningStrategy.select()` was calling `getAllPlanItems()` — which does `List.copyOf(itemsById.values())` — then streaming the result into a `Map<String, PlanItem>` on every context-change cycle. The obvious fix was to use `getPlanItemByBindingName()`, which already exists. But that method filters to active items only — PENDING, RUNNING, DELEGATED. The strategy needs to see COMPLETED items to skip forward in the sequence.

The underlying issue: the internal map (`activeByBinding`) retains terminal items because nothing cleans them up — but `getPlanItemByBindingName()` filters them out, and `hasActivePlanItem()` lazily removes them. The same data structure, three different access semantics depending on which method you call. I added `findPlanItemByBindingName()` as a default method on `CasePlanModel` — any-status lookup, O(1), zero allocation. Renamed the internal map to `latestByBinding` to stop the name from lying about what it contains.

The third (#622) was the simplest: `buildBulkSignalEventLog()` stored `{"type": "bulk_signal"}` as the payload — no record of what was actually updated. A signal that changes `result` and `score` looked identical in the audit trail to one that changes `status` and `deadline`. The fix adds the full updates map to the payload and the key names to metadata for fast filtering.

The pattern across all three: each gap was invisible from the API surface. The success path worked, tests passed, the feature shipped. The gaps lived in paths that only fire under specific failure conditions, or under load, or when an auditor queries the event log. The kind of thing you find when you trace a flow from end to end rather than reading a single handler.
