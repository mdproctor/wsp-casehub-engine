---
layout: post
title: "The Trampoline That Changed the Design"
date: 2026-06-26
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-engine]
tags: [subcase, recursion, depth-limiting]
---

## The Trampoline That Changed the Design

The merge queue CasePlanModel needs recursive bisection — a batch case that spawns two sub-batch cases of itself, each of which may bisect further. The engine's `SubCaseExecutionHandler` has a hard guard against this: if the parent case definition matches the child SubCase identity, it faults immediately. No exceptions.

The obvious fix is a `maxRecursionDepth` field on `SubCase`: 0 means hard block (current behaviour), N means allow N levels. Default 0 — backward compatible. The field lives on the binding target, not on `CaseDefinition`, because different bindings within the same case could legitimately need different depth limits.

I initially proposed two counting approaches — consecutive (reset when a different definition intervenes in the ancestry chain) and total (count all same-definition ancestors regardless of what's between them). The distinction seemed academic until I traced a specific scenario.

Consider a trampoline: case A spawns B, B spawns A, the inner A spawns itself. With consecutive counting, B resets the depth counter — the inner A thinks it's at depth 0 and recurses freely. A→B→A→A→A→B→A→A→A→B→... runs indefinitely. The `maxRecursionDepth` limit never binds.

With total counting, the inner A finds root A above B in the ancestry chain and counts it. Each new A in the hierarchy has more same-definition ancestors than the last. Eventually depth hits the limit. The trampoline is bounded.

The implementation walks the `parentCaseId` chain via `CaseInstanceCache` — a bare `ConcurrentHashMap` with no eviction, no `remove()` method, no TTL. Every ancestor in a recursive chain is in WAITING state, so they're all in the cache. The walk short-circuits at `maxRecursionDepth`: for the default case (0), the loop body never executes and depth 0 >= limit 0 faults immediately. No special-case branch needed.

One thing worth noting: this only protects against direct self-reference. Mutual recursion (A→B→A→B→...) bypasses the guard entirely — B spawning A isn't a self-reference from B's perspective, so `maxRecursionDepth` is never consulted. Cycle detection across multiple definitions would require walking the ancestor chain regardless of definition match, tracking all visited definitions. A different feature with different performance characteristics.

The single-node assumption is the other constraint to name explicitly. `CaseInstanceCache` is per-JVM. In a clustered deployment, the depth walk on node B wouldn't find ancestors cached on node A. The engine has no clustering support today (RAM Quartz store, in-memory cache), but anyone adding it would need to revisit this.
