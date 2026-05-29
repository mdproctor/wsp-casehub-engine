---
layout: post
title: "Worker carries a definition, not an outcome"
date: 2026-05-29
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-engine]
tags: [spi, mutiny, cdi, reactive]
---

## Worker carries a definition, not an outcome

`WorkerProvisioner.provision()` had been returning `Worker` since the provisioner SPI was first written. I drafted the follow-on feature — causal audit linkage, connecting a provisioned worker back to the Qhorus COMMAND that triggered it — as a nullable `causedByEntryId` field on `Worker`. Set by the provisioner after building the worker object, returned through it.

The design review caught the problem. `Worker` is a case-definition artifact: capabilities, execution policy, the function that runs. That's what the case author declares. Provisioning outcomes are different in kind — they're runtime results, not declarations.

So we introduced `ProvisionResult(UUID causedByEntryId)`. The provisioner SPIs now return this instead of `Worker`. No-op defaults still throw `ProvisioningException` — the return type change is only in the declaration. `ProvisionResult.empty()` handles the common case where causal linkage isn't resolved yet.

The linkage itself will remain null until a later piece of work threads the Qhorus trigger context through the provisioning call. The SPI shape is right; the values arrive separately.

## The CompletionStage going nowhere

`CaseStatusChangedHandler` had been calling `lifecycleEvents.fireAsync()` inside a Mutiny `.invoke()` callback. `.invoke()` is a side-effect callback — it runs the lambda and continues the Uni regardless of return value. The `CompletionStage` from `fireAsync()` was silently discarded.

The consequence: `@ObservesAsync` observers might run after the handler's Uni resolved. The Vert.x event bus considers a message processed when the returned Uni completes, not when CDI observers finish. In practice this creates a race — and in tests, where executor threads aren't guaranteed to drain, it consistently loses.

The fix: move `fireAsync()` to `.chain(() -> Uni.createFrom().completionStage(...))`. The handler's Uni now waits for CDI delivery before completing. Observer failures get logged and recovered — a ledger write error must not roll back a case state transition.

One detail: the Supplier form of `completionStage()` is required. The non-Supplier form evaluates the `CompletionStage` eagerly at chain construction time. The Supplier is lazy — it calls `fireAsync()` at subscription time, on the right thread.

Five other handlers have the same pattern. They're tracked for a follow-up batch.

## What @ObservesAsync doesn't do in tests

Writing a regression test for the CDI delivery fix hit an unexpected wall. Quarkus ArC processes `@ObservesAsync` observer registrations at build time. Inner-class `@ApplicationScoped` beans in `@TestProfile` tests are injected correctly — the injection point resolves — but their observer methods aren't in ArC's observer registry. `fireAsync()` returns a `CompletionStage` that completes immediately because there is nothing to notify. No exception, no warning.

The test covers case completion and confirms the Uni chain runs without deadlock. The direct CDI delivery assertion is documentation of intent — one that becomes testable if the observer is moved to main sources.
