---
layout: post
title: "Six handlers and a miss"
date: 2026-05-30
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-engine]
tags: [cdi, mutiny, quarkus, bugs]
---

The batch we cleared this session was five issues — all small, all overdue,
all the kind of bugs that pass unit tests and quietly break in production.

The connecting thread: isolation hides them. Put each module in its own test
container with its own alternatives configured and you never see the failure.
Add `casehub-engine-ledger` to AML's test classpath and suddenly a CDI ambiguity
brings the whole suite down.

## The fireAsync fix, and the one I missed

Five lifecycle event handlers in the engine were calling `lifecycleEvents.fireAsync()`
inside `.invoke()`. Mutiny's `.invoke()` discards the `CompletionStage` returned by
`fireAsync()` — which means `@ObservesAsync` observers (including
`CaseLedgerEventCapture`) might run after the handler's Uni completes. In testing,
Awaitility's polling usually masked this. In production, under load, the ordering
guarantee disappears.

The fix: wrap every `fireAsync()` in `.chain(() -> Uni.createFrom().completionStage(...))`
with an `onFailure().recoverWithItem()` so observer failures don't break case transitions.
We applied it to all five handlers — GoalReachedEventHandler, MilestoneReachedEventHandler,
CaseStartedEventHandler, SignalReceivedEventHandler, WorkflowExecutionCompletedHandler.

Then I sent the whole batch to a separate Claude for code review. It came back with one
critical finding: `tryProvision()` in `CaseContextChangedEventHandler` was firing a
`WorkerStarted` lifecycle event using the exact same discarded-CompletionStage pattern.
I had introduced it in the same batch while fixing the provisioner SPI — and missed it
completely while fixing the other five. We fixed that one too before the PR went up.

## The callerRef argument

The bug in `HumanTaskScheduleHandler.handleTemplateMode()` was a single wrong argument
position. The method was calling `workItemTemplateService.instantiate(template, title,
callerRef, "casehub-engine")`. The third argument is `assigneeIdOverride`. So every
WorkItem created via template mode had its assignee set to `case:{caseId}/pi:{planItemId}`.

The existing test didn't catch it because it filtered WorkItems by `callerRef`, not by
`assigneeId`. And `callerRef` was set correctly — by an explicit assignment on the next
line after the bad call. The test found the right WorkItem; it just never checked whether
`assigneeId` was garbage.

We added `assertThat(created.assigneeId).isNull()` to the test before fixing the production
code. That's the assertion that would have caught this from the start.

## The CDI inheritance that isn't

`CaseLedgerEntryRepository extends JpaLedgerEntryRepository`. The parent class is annotated
`@Alternative` in casehub-ledger. The child was bare `@ApplicationScoped` — which seems
reasonable until you remember that CDI does not inherit annotations from parent classes.
`@Alternative` is not `@Inherited`. The child's CDI behaviour is governed solely by its
own annotations.

The result: `CaseLedgerEntryRepository @ApplicationScoped` is always active, regardless of
which alternative AML configures via `quarkus.arc.selected-alternatives`. Add
`casehub-engine-ledger` to AML's classpath, and CDI sees two active beans implementing
`LedgerEntryRepository` — the always-on child and whatever AML selected. Startup fails.

The fix is the same pattern we use for every engine SPI no-op: `@DefaultBean
@ApplicationScoped`. It yields automatically to any non-default bean. AML's configured
alternative wins; the engine's default stands aside.

## The escalation signal

The fifth issue was genuinely new work rather than a fix: when a WorkItem escalates —
its SLA expires and it re-enters PENDING with a new candidate group — the engine now writes
a `workItemEscalated` signal to the case context. Case definitions can bind on
`contextChange(".workItemEscalated")` to react: notify a supervisor, adjust scope, or
record an audit entry.

The pattern is identical to how Qhorus message signals work — external events write to a
named context path, definitions bind on it. Simple to implement once the precedent exists.
