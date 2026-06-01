---
layout: post
title: "Fixes, a mystery, and three missing migrations"
date: 2026-06-01
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-engine]
tags: [casehub, tenancyId, flyway, quarkus-arc, cdi]
---

The S/XS backlog from the multi-tenancy session had nine open issues. I went through them all in a single branch today — eight commits, one PR. Most were straightforward. Two weren't.

## Three migrations that should have been there

When engine#299 added `tenancyId` to `CaseMetaModel`, `CaseInstance`, and the SPI methods, it also added `tenancy_id` columns to `CaseLedgerEntry` and `WorkerDecisionEntry` as JPA entity fields — `@Column(nullable = false)`. But the Flyway migrations for those columns never shipped. The engine uses `drop-and-create` for its own schema, so nobody noticed. casehub-aml noticed: they added the columns via V2005 and V2006 in their own migration chain.

We shipped V2002, V2003, and V2004 today — using `ADD COLUMN IF NOT EXISTS` on everything, so aml's migrations remain safe. That also revealed the companion bug: `CaseLedgerEventCapture` and `WorkerDecisionEventCapture` were constructing their ledger entries without ever setting `tenancyId`, despite the field being `nullable = false` on the entity. The H2 test DB must not enforce that constraint on JOINED inheritance subtables, which is why the tests passed for months. Both services now set it from the event.

`WorkerDecisionEvent` itself needed `tenancyId` as a new component. A code review caught that I'd placed it fifth — after `traceId`. We moved it to second, matching `CaseLifecycleEvent`'s ordering. That's now a formalised protocol.

## The registry mystery

engine#410 reported that `getCaseDefinition(caseInstance.getCaseMetaModel())` was returning null in casehub-life's `AppointmentCycleIntegrationTest`, despite the definition being successfully registered at startup. The error surfaced at `SchedulerService.registerScheduledTriggers()` and blocked all integration tests.

I spent most of an hour on static analysis. Claude did too. The `ConcurrentHashMap.get()` call uses `hashCode()` first — `Objects.hash(namespace, name, version)` — and the key is the same object reference as what's in the map. There's no mutation path for those three fields after registration. The CDI beans are application-scoped singletons. The Vert.x event bus uses pass-by-reference for local delivery. I couldn't find the bug.

Claude proposed a defensive fallback: if `Map.get()` returns null, fall back to linear scan with a WARN log. That way, if the bug fires in production, the log captures the key state for diagnosis and the lookup still succeeds. We shipped that. The root cause remains open as engine#410.

## Trust score audit fields

engine#403 asked for `trustScoreAtRouting` and `thresholdApplied` on `WorkerDecisionEntry`, to eliminate casehub-aml's workaround (`AmlTrustRoutingAttestation`). The tricky part was timing: the trust score is determined at routing time, but `WorkerDecisionEvent` fires at completion time. We read from `TrustScoreCache` at event observation time — same approximation aml was using, just canonical now. Both fields are nullable `Double`; null means trust routing wasn't active for that worker selection.

Adding `TrustScoreCache` injection to `WorkerDecisionEventCapture` triggered a cascade of CDI startup failures in the ledger tests. It turned out the `casehub-ledger-runtime` SNAPSHOT had recently gained identity enricher beans (`IdentityCacheInvalidator`, `ActorDIDEnricher`, and five others) that require `DIDResolver` and `ActorDIDProvider` — types not in the engine-ledger test classpath. Quarkus ARC validates the full dependency chain at augmentation time regardless of runtime config flags, so `casehub.ledger.identity.tokenisation.enabled=false` didn't help. The fix was adding all seven beans to `quarkus.arc.exclude-types`. You have to list the complete chain in one pass — excluding one surfaces the next as the new unsatisfied dependency.

## The small ones

`SubCaseCompletionService` now fires `Event<SubCaseGroupLifecycleEvent>` for every group transition — IN_PROGRESS, COMPLETED, REJECTED. engine#249 had been waiting for this since engine#112 created the event type and never wired it. Two lines in the handler, one new constructor parameter, one new test.

`CaseHub.startCase` now accepts `Object` instead of `Map<String, Object>`, matching quarkus-flow's `Flow.instance(Object)` convention. `CaseHubRuntimeImpl.toContextMap()` handles the Jackson conversion; existing `Map<String, Object>` callers compile without changes. `HumanTaskTarget` gains `claimDeadlineHours` wired to `WorkItemCreateRequest.claimDeadlineBusinessHours` — devtown's SLA enforcement gap.

ADR-0004 formalises what was implicit: the case definition registry is global, `tenancyId` on `CaseMetaModel` is a sentinel (`DEFAULT_TENANT_ID` from startup), and tenant isolation belongs at the case instance level — not the definition level.
