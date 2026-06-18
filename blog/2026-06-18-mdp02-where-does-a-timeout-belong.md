---
layout: post
title: "Where Does a Timeout Belong?"
date: 2026-06-18
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-engine]
tags: [sealed-types, spi-design, mutiny, failure-cascade]
series: issue-533-codec-and-expired-wiring
---

`OutcomePolicy` declared `onExpired` from the start. The YAML schema parsed it, the mapper wired it, the default constructor set it to `REROUTE`. But nothing in the engine ever used it. A worker that timed out hit `TimeoutException`, fell into the infrastructure retry loop alongside genuine crashes, retried at the same timeout limit, exhausted its retries, and faulted the case. The distinction between "worker crashed" and "worker ran out of time" didn't exist at runtime.

The fix was straightforward — add `WorkerOutcome.Expired` to the sealed hierarchy and wire it through the same `handleSemanticFailure` path that handles `Declined` and `Failed`. But the interesting question was *where* to intercept the timeout.

The spec initially placed the conversion in `QuartzWorkerExecutionJob` — recover from `TimeoutException` in the Quartz adapter's Uni chain. Code review caught the problem: `DefaultWorkerExecutor` is where the timeout originates (`.ifNoItem().after(Duration).fail()`), and the `WorkerExecutor` SPI contract is `Uni<WorkerResult>`. If the executor leaks `TimeoutException` through the SPI boundary, every adapter must independently handle the conversion. A future db-scheduler adapter would need to duplicate the same recovery logic, or timeouts silently revert to infrastructure failures.

The fix belongs where the concern originates. The executor owns the timeout, so it owns the semantic conversion. Adapters only ever see `WorkerResult` — they don't know or care whether the result came from a successful function return, a decline, or a timeout.

The other design decision worth noting: the handler's outcome fork. The existing code used a positive instanceof chain — `if (Declined || Failed)` — which is an allowlist. A future fifth variant to the sealed hierarchy would silently fall through to the success path. We switched to `if (!(instanceof Success))`, matching the pattern `PlanItemCompletionHandler` already uses. Any non-success outcome enters the semantic failure path, where the exhaustive switch produces a compile error for unhandled variants.

Claude caught a subtlety during implementation: Mutiny's `.ifNoItem().after().fail()` throws `io.smallrye.mutiny.TimeoutException`, not `java.util.concurrent.TimeoutException`. Same simple name, different package, no inheritance relationship. Catching the JDK type compiles cleanly and catches nothing — the recovery silently never fires.

The argument for routing timeouts through `OutcomePolicy` instead of retry: a timed-out worker that receives the same input at the same timeout will likely time out again. `onExpired: REROUTE` sends the work to a different agent with different latency characteristics. If the original worker would succeed with more time, the fix is a longer `timeoutMs` in `ExecutionPolicy` — not a retry at the same limit that already failed.

This also sets up the Qhorus path cleanly. When commitment expiration ships (qhorus#281), the bridge publishes `WorkflowExecutionCompleted` with `WorkerOutcome.Expired` to the same event bus address. No engine changes needed — the handler already processes `Expired` generically.
