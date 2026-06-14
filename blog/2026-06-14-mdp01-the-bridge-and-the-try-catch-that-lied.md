---
title: "The bridge and the try/catch that lied"
date: 2026-06-14
slug: the-bridge-and-the-try-catch-that-lied
projects: [casehub-engine]
tags: [casehub, quarkus, cdi, bridge-module, qhorus]
---

The foundation constraint is simple: `casehub-qhorus` and `casehub-work` are peer Foundation modules. Neither can depend on the other. But consumer apps often need both — a message arrives on a qhorus channel and should become a WorkItem. Without a bridge, every consumer writes the same wiring.

`casehub-engine-inbound` lives in the engine repo as an optional classpath module. Add it to your dependencies, drop in an `InboundWorkItemPolicy` bean, and messages start becoming WorkItems. No policy bean, no WorkItems, no noise.

## The channels() problem

`MessageObserver` has a `channels()` method. Return a set of channel names; the dispatcher only calls `onMessage()` for those channels. It looks like the obvious way to filter.

The problem: CaseHub channels are dynamic. A case opens a channel named something like `case-{uuid}-{purpose}`. There is no finite set of channel names to declare at startup. `MessageObserver.channels()` does exact string matching — no prefix support. Pre-declaring which channels the bridge handles means the SPI becomes useless for the typical CaseHub deployment.

The bridge receives everything, and the policy decides. Return `Optional.empty()` for messages to ignore; return a `WorkItemCreateRequest` for those to act on. The policy checks `event.channelName()`, `event.messageType()`, or whatever else matters to the deployment. At typical qhorus message volumes, the per-message policy invocation is not a concern.

## SPI placement

`InboundWorkItemPolicy` returns a `WorkItemCreateRequest`. That type comes from `casehub-work`. Placing the SPI in `casehub-engine-api/spi/` would force `casehub-work` onto every `api` consumer regardless of whether they use the bridge. The SPI stays in `casehub-engine-inbound`. Consumers who activate the bridge already depend on it — no forced transitive.

## The try/catch that lied

The bridge calls the policy, then uses `Optional.ifPresent()` to call `WorkItemService.create()` if a request comes back. The original implementation wrapped both in a single try/catch:

```java
try {
    policy.get().decide(event).ifPresent(request ->
        tenantContextRunner.runInTenantContext(..., () -> workItemService.create(stamp(request))));
} catch (Exception e) {
    LOG.warnf(e, "policy threw — message ignored");
}
```

It looks like it isolates policy exceptions. It doesn't. `ifPresent()` calls the lambda synchronously. Any exception from `workItemService.create()` propagates up through the lambda, exits `ifPresent()`, and is caught — logged as a policy failure, which is wrong. Claude flagged this when checking the spec description against the implementation. The fix separates the policy call:

```java
final Optional<WorkItemCreateRequest> decision;
try {
    decision = policy.get().decide(event);
} catch (Exception e) {
    LOG.warnf(...);
    return;
}
decision.ifPresent(request -> ...);
```

Now policy exceptions are caught with channel context. Infrastructure exceptions propagate out of `onMessage()` to the dispatcher's own safety net.

## Boot failure from a missing stub

Tests exclude `JpaWorkloadProvider` to avoid database setup. But `WorkItemAssignmentService` constructor-injects `WorkloadProvider workloadProvider` directly — not `Instance<WorkloadProvider>`. Excluding `JpaWorkloadProvider` leaves zero `WorkloadProvider` candidates; CDI fails at startup.

The confusion: `JpaWorkloadProvider` only calls `workItemStore.scan()`, no direct database connection. The boot failure has nothing to do with database access — it's the injection chain one level up. A `StubWorkloadProvider` inner class in the test resolves it.

Both bugs were invisible at the description level. The spec said what the behaviour should be; the code didn't implement it.
