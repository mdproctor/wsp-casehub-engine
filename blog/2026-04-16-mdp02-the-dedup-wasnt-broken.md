---
layout: post
title: "The Dedup Wasn't Broken. The Test Was."
date: 2026-04-16
type: phase-update
entry_type: note
subtype: diary
projects: [casehub]
tags: [quarkus, hibernate, testing, persistence, debugging]
excerpt: "PR 2 shipped, Flyway is gone, and two CI failures that turned out to be different problems than they appeared."
---

PR 2 is open: `casehub-persistence-memory`, 30 unit tests, no Docker, no PostgreSQL. `InMemoryEventLogRepository`, `InMemoryCaseMetaModelRepository`, `InMemoryCaseInstanceRepository` — all three SPI interfaces covered with plain JUnit 5, direct instantiation, `Uni.await().indefinitely()`. The upstream maintainer prefers reviewable pieces, so it stacks on PR 1 as its own PR.

The two-stage review in the subagent workflow caught things I wouldn't have caught reviewing my own code. Claude flagged that `save()` was unconditionally assigning a new id on every call — no guard for the case where `id` is already set. JPA semantics only assign an id on INSERT for new entities; the in-memory version was diverging silently. One check fixed it. The reviewer also caught that `update()` silently accepted an unknown UUID with no signal that nothing had been stored — a caller bug that would be loud in the JPA version and invisible in-memory.

---

The upstream maintainer's review of PR 1 asked to merge the SQL files — there were four Flyway migrations, and with no installed instances there was nothing to migrate. I removed all of it: the SQL files, `quarkus-flyway`, `quarkus-jdbc-postgresql`, `quarkus-agroal`, and the Flyway config. Hibernate now manages the schema directly with `drop-and-create`.

Removing Flyway meant removing the JDBC connection pool, which meant removing the JDBC Quartz store. Quartz was configured with `store-type=jdbc-cmt`, which needs the JDBC pool and a set of Quartz tables in the database. With no Flyway to create those tables and no pool to connect with, Quartz had to switch to `ram`. That's the right call anyway — there's no cross-node scheduling requirement yet, and RAM store is simpler.

The full chain wasn't obvious until I pulled on the first thread. Flyway → JDBC pool → Quartz JDBC store → Quartz tables in migration SQL → gone.

---

Two CI failures were waiting in PR 1.

The first: `JpaCaseMetaModelRepositoryTest.save_thenFindByKey_roundTrip` was failing on `expected: 2026-04-16T18:35:11.983912167Z but was: 2026-04-16T18:35:11.983912Z`. Java's `Instant.now()` has nanosecond precision; PostgreSQL `timestamp` columns store microseconds. We write a nanosecond-precise value, PostgreSQL truncates it, we read back a microsecond-precise value, the equality check fails. The fix: `Instant.now().truncatedTo(ChronoUnit.MICROS)`. Round-trip is now stable.

The second looked like a broken deduplication mechanism. `SignalTest.workerRunsOnceOnDuplicateSignal` — send the same signal twice, expect the worker to run once, get two. The test had previously been `@Ignore // TODO`; a commit from the upstream maintainer added a dedup fix and activated the test. As the analysis below shows, the dedup mechanism was correct — the failure was a pre-existing test isolation problem that the newly active test made visible.

The dedup code uses `applyAndDiff()` under a write lock. If signal 1 sets `payment` in the case context, signal 2 calls `applyAndDiff`, sees the value unchanged, and returns `Optional.empty()` — no second `CONTEXT_CHANGED` event, no second worker scheduling. That logic is correct.

The problem was the counter. `SignalCaseHubBean.runCount` is a static field shared across all tests in the class. Workers from a previous test executing asynchronously — after that test's `await()` has already passed and `@BeforeEach` has reset the counter — can increment it during the dedup test's 10-second window. The dedup mechanism hadn't failed; the measurement was contaminated.

The fix: add a `ConcurrentHashMap<String, AtomicInteger>` keyed by `orderId` alongside the global counter, extract `orderId` in the worker's input schema, and make the dedup test use a unique `orderId` per run. Now the test checks how many times the worker ran for *this specific case*, not for all cases combined.

The mechanism was correct from the start.
