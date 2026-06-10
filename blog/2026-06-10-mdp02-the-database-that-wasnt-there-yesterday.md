---
layout: post
title: "The Database That Wasn't There Yesterday"
date: 2026-06-10
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-engine]
tags: [ci, snapshot, panels, ledger, quarkus]
---

# The Database That Wasn't There Yesterday

Two PRs. Simple enough. Resolve the conflicts, make CI green, squash the history, merge. I expected maybe an hour.

Six hours later, I'd traced a CI failure through three layers of misdirection before finding the actual cause buried in a dependency changelog we'd never even looked at.

The panels PR (#467) had merge conflicts the moment I landed the expression evaluator PR first. The rebase was clean — git dropped the shared commits automatically. Then CI failed. Tests were timing out: `SimpleCaseHubBeanTest`, `SpiWiringIntegrationTest`, the whole basic lifecycle suite. Nothing about the panels. Just cases not completing.

I pulled the surefire dump file — something I wouldn't have thought to do without the pattern recognition from an earlier failure. Buried in the first-run output was this:

```
ERROR: relation "ledger_subject_sequence" does not exist
[MERGE INTO ledger_subject_sequence AS t USING (SELECT CAST(? AS UUID) AS sid) ...]
```

Not a PostgreSQL sequence. A table. Added, at some point between branch CI and main CI, by a `casehub-ledger` SNAPSHOT update. The `LedgerSequenceAllocator` class was new. It ran every time a case fired a lifecycle event. Without its table, the CDI observer failed silently — cases started but never completed, timeouts cascaded, and nothing in the test output pointed at the ledger at all.

The fix was straightforward once we had the diagnosis: exclude `CaseLedgerEventCapture` and `WorkerDecisionEventCapture` from CDI in the runtime test profile. They're audit observers; runtime tests don't need them and don't have the schema they need to run. One `quarkus.arc.exclude-types` entry.

The other thing we found: the panels migration left a handful of corners unfinished. JQ expressions in `candidateGroupsExpression`, `Milestone.entryCriteria`, and `WorkerScheduleDedupTest`'s `inputSchema` all still referenced flat context keys (`.status`, `.routing`, `.documentId`) instead of panel-scoped paths (`.working.status`, etc.). Each one caused a different failure mode — binding triggers that never fired, dedup hashes that didn't match, milestones that wouldn't activate.

The more interesting one was the recovery path. `applyTopLevelChanges` in `DefaultWorkerExecutionRecoveryService` was doing `CaseContext.set("working", afterMap)`. That goes through the flat API — it stored a key named "working" inside the working panel instead of replacing the working panel's contents. So after a JVM restart, `getPathAsString("status")` returned null even though the case had run correctly. The test caught it; the production path wouldn't show the bug until someone actually restarted a node and asked why the context looked wrong.

The fix was `ctxImpl.writablePanel(key).clear().setAll(afterMap)` — replace, not merge. The `clear()` matters because the panel diff records the full new state, not a delta. Without it, removed keys would survive across restarts.

Both PRs are on main. CI is green. The panels architecture — `ReadablePanel`, `WritablePanel`, `WritablePanelImpl`, semantic and episodic memory, user-defined panels, panel-aware recovery — is shipped.

The SNAPSHOT instability is the thing worth keeping an eye on. We were compiling against one version of `casehub-ledger` on the feature branch and a different version hours later when main CI ran. The version string didn't change. There's no good tooling for that — you just have to notice when tests start failing in ways that don't match any code you changed.
