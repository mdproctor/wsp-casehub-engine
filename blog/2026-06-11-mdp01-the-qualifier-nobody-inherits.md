---
layout: post
title: "The Qualifier Nobody Inherits"
date: 2026-06-11
entry_type: note
subtype: diary
projects: [casehub-engine]
tags: [multi-tenancy, jpa, cdi, quarkus]
---

Four issues. Two were already done — one we'd fixed in the previous session and never closed the ticket, one adapted to a SNAPSHOT change without noticing the issue still existed on the board. Closed both without writing a line.

The remaining two took longer but taught something.

**The one that worked until it didn't**

`CaseLedgerEntryRepository` extends `JpaLedgerEntryRepository`, which qualifies its `EntityManager` with `@LedgerPersistenceUnit`. The subclass added its own `EntityManager` for `findByCaseId` queries — without the qualifier. In a single-datasource deployment, this is invisible: unqualified injection resolves to the only PU available. In a two-datasource deployment (engine default plus qhorus), it resolves to the wrong one, silently.

CDI does not inherit qualifiers across fields. The parent's annotation stays on the parent's field. The subclass field starts fresh, unqualified, default PU. The queries run, the results come back empty, nobody throws, nobody logs.

I've seen variations of this pattern before — the test that passes in isolation, the behaviour that works until the second thing shows up. Multi-datasource deployments are a different execution environment from what most tests run against. The fix was one annotation. Understanding why it was needed took longer.

**The tenant that had to travel**

The other issue was adding `tenancyId` to `PlanItemCompletedEvent`. The reason it needed to be there: devtown was using `CrossTenantCaseInstanceRepository` inside an `@ObservesAsync` handler to look up the tenancyId — using the cross-tenant repo as a workaround for the fact that `CurrentPrincipal` doesn't exist on the async executor thread.

That's the wrong instrument. Cross-tenant repos are documented as startup-recovery-only. The right answer is that tenancyId must travel with the event. The caller always has it. Putting it on the event record is the only approach that doesn't require request scope.

Once we added it to `PlanItemCompletedEvent`, we found `SubCaseExecutionCompleted` had the same problem: `SubCaseCompletionService` had `event.tenancyId()` from the triggering `CaseLifecycleEvent` and wasn't carrying it forward. Same fix one call deeper.

The rule it surfaces — any CDI event consumed by `@ObservesAsync` handlers must carry its own tenancyId — is simple enough that it probably should have been written down before now. It's now PP-20260611-d4e5cf.

What we didn't add was `tenancyId` to `SubCaseExecutionCompleted` when the event was originally designed. I'm not sure we had multi-datasource, multi-tenant deployments in mind at the time. We do now.
