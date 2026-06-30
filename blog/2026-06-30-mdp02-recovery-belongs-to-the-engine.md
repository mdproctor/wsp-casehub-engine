---
layout: post
title: "Recovery Belongs to the Engine, Not the Scheduler"
date: 2026-06-30
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-engine]
tags: [architecture, health-check, recovery, quartz]
---

## Recovery Belongs to the Engine, Not the Scheduler

The engine's Quartz backend ran startup recovery — scanning event logs for workers that were scheduled but never completed, then rescheduling them. It tracked the outcome with a `volatile RecoveryStatus` field. The problem wasn't that recovery was broken. It was that it lived in the wrong place.

I started with a straightforward task: wire `RecoveryStatus` to a health check endpoint. The obvious implementation was `@Liveness` health check → inject `QuartzWorkerExecutionManager` → read `getRecoveryStatus()`. Three files, thirty minutes.

But the `CompositeWorkerExecutionManager` work from the previous session had just established `@WorkerBackend` as the abstraction layer for execution backends. A health check that bypasses the abstraction and reaches into one concrete backend is the kind of quiet coupling that compounds until the abstraction is fiction.

So I traced the actual ownership chain. `WorkerExecutionRecoveryService` does the recovery work — it's backend-agnostic, lives in `common`, and routes through the composite. `QuartzWorkerExecutionManager.onStart()` calls it and tracks the status. But recovery doesn't belong to Quartz any more than a health check belongs to one database driver. The Quartz backend just happened to be the one that existed when recovery was written.

The fix was to extract recovery initiation into a `WorkerRecoveryCoordinator` in `runtime`. The coordinator owns the startup observer (`@Priority(22)`, slotting between Quartz listener registration at 20 and human task recovery at 25), calls the recovery service, tracks the status, and exposes it to a `@Liveness` health check. The Quartz backend retains only what's genuinely Quartz-specific: registering the job listener.

One thing the design review caught that I'd missed: a configurable timeout. Without it, a hung recovery (the `Uni` never completing) maps PENDING to the "everything is fine" health signal — permanently. `casehub.engine.recovery.timeout` (default 60s) with `.ifNoItem().after(timeout).fail()` converts the hang into FAILED, which maps to liveness DOWN, which triggers a restart.

The behavioural change is worth noting: recovery now fires unconditionally from `runtime`, not gated on `scheduler-quartz` being on the classpath. That's correct — the recovery service already routes through the composite, which dispatches to whichever backends are available. Gating recovery on one backend was the accidental coupling showing through.
