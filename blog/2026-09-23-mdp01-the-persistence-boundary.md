---
layout: post
title: "The Persistence Boundary"
date: 2026-09-23
entry_type: note
subtype: diary
projects: [casehubio/engine]
tags: [spi, persistence, evolution, conductor]
series: issue-1140-conductor-state-persist
---

# The Persistence Boundary

The evolution conductor landed in the last session — inbox, escalation, deny lists, manual blocks, the full command centre surface. All of it backed by `ConcurrentHashMap`. Which means all of it gone on restart.

That was always the plan: get the domain logic right first, extract persistence later. But "later" has a way of becoming "never," and the conductor's state isn't just debug data. Watch patterns and deny patterns are operator safety configuration — an operator who blocks self-modification of `EvolutionTicker` and then restarts the system has silently lost that constraint. That's not acceptable as a permanent state. The question was what "durable" actually means here.

## The EventLog trap

The obvious path was EventLog-replay — the same pattern the #1132 spec designed. On mutation, write an event. On restart, replay events to reconstruct state. It's the established pattern for circuit breaker state and compliance levels.

It's also the pattern we're moving away from. The persistence coherence audit found two silent data-loss bugs caused by the implicit contract that every context-mutating event must have a replay handler. Miss one handler and state silently disappears on restart. The new platform direction is clear: EventLog for audit and business queries, database for durability. `CaseContextRecoveryStrategy` will provide database-backed snapshot recovery as the default, with EventLog replay as an experimental opt-in.

So EventLog-replay was off the table. But JPA entities now would be premature — the SPIs need to stabilise first. The right move was to draw the persistence boundary and defer the durable implementation.

## Four SPIs, not three

The issue proposed three repositories. I split it to four.

The original design bundled watch patterns with inbox entries in a single `ConductorInboxRepository`. But these have fundamentally different lifecycles. Inbox entries are transient — created when a gate blocks, resolved when the conductor decides, eventually timed out. Watch patterns are persistent operator configuration that should survive across improvement cycles. Coupling them in one repository meant the persistence strategy couldn't differentiate between "this can be lost" and "this must survive."

The four SPIs: `ConductorInboxRepository` for gate decisions, `WatchPatternStore` for escalation patterns, `DenyPatternStore` for dynamic deny lists, `ImprovementBlockStore` for manual coordination blocks. All in `common-core/spi` alongside the other persistence SPIs, all tenant-scoped with `tenancyId` on every method.

The split is small — one extra interface — but it means a future durable implementation can treat watch patterns and deny patterns with the care they deserve while leaving inbox entries as cheap in-memory state.

## The domain beans go stateless

After the extraction, `ConductorInboxManager` and `ImprovementCoordinator` are pure delegation — they inject their repositories and contain only business logic. `ImprovementBudgetEnforcer` is the interesting case: it still holds ephemeral runtime state (active improvements, daily counts, cooldown timer) that appropriately stays in the bean. Only the dynamic deny patterns — the operator-configured safety rails — moved behind the SPI. The distinction matters: ephemeral tracking state resets to zero on restart and that's correct. Operator configuration silently vanishing is not.

Code review caught one genuine regression: during the structural editing of `matchesGlob()`, the regex escaping doubled — `"\\."` (literal dot) became `"\\\\."` (literal backslash followed by any character). A silent correctness bug in glob deny pattern matching that would have shipped without the review pass.

## What the boundary opens up

The SPI boundary means durable implementations slot in without touching business logic. When the `CaseContextRecoveryStrategy` work lands, someone writes a JPA-backed `WatchPatternStore` with its own table and tenant-scoped queries, drops it in as a non-`@DefaultBean` CDI bean, and the in-memory default silently yields. The domain beans never know the difference.

The conductor stores are all case-scoped and low-mutation — operator timescale, not tick timescale. Database writes are trivially cheap for data that changes when a human decides to add a watch pattern or block an improvement stream. The performance argument for in-memory is irrelevant here; the durability argument is everything.
