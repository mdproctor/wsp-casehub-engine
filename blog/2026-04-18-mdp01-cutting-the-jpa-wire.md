---
layout: post
title: "Cutting the JPA Wire"
date: 2026-04-18
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-engine]
tags: [persistence, jpa, spi, maven]
---

Three domain objects. Twelve handlers. One pom.xml. PR3 is done.

The goal was to strip JPA entirely from the engine module — no Hibernate
annotations, no Panache, no `PanacheEntity` superclass. Every persistence
operation routes through the three SPI interfaces Claude and I scaffolded
in PR2: `CaseMetaModelRepository`, `CaseInstanceRepository`,
`EventLogRepository`.

## Domain objects as plain Java

`CaseMetaModel`, `CaseInstance`, and `EventLog` had all been extending
`PanacheEntity`. That gave them an implicit `id` field, JPA annotations
throughout, and static Panache query methods baked in —
`EventLog.findSchedulingEvents(caseId, workerId)` was a real method on the
domain class.

After the conversion they're plain Java. The `id` field is still public
(repository implementations set it after save), but the class has no framework
dependency. `EventLog.findSchedulingEvents` is gone; callers inject
`EventLogRepository` and call it there instead.

## The atomic operation that wasn't in the SPI

`CaseStatusChangedHandler` needed something PR2 hadn't anticipated: update a
`CaseInstance` state *and* append an `EventLog` entry in a single transaction.
Two separate calls wouldn't work — a crash between them leaves the system in
inconsistent state.

I added `updateStateAndAppendEvent(CaseInstance, EventLog)` to
`CaseInstanceRepository`. The JPA implementation in `casehub-persistence-hibernate`
wraps both writes in one Panache transaction:

```java
return Panache.withTransaction(() ->
    CaseInstanceEntity.<CaseInstanceEntity>findById(instance.id)
        .chain(entity -> {
            entity.state = instance.getState();
            return Panache.getSession().chain(s -> s.merge(entity));
        })
        .chain(merged -> logEntity.persistAndFlush())
);
```

The in-memory version is simpler — store the updated instance and delegate to
`EventLogRepository.append`. In-process, synchronous, no transaction needed.

This is the point of the SPI boundary: the engine sees one method, each
implementation handles atomicity in whatever way makes sense for its storage model.

## The Maven cycle that wasn't obvious

The plan called for adding `casehub-persistence-memory` as a test-scope
dependency of `engine`, so tests could activate the in-memory repositories via
`quarkus.arc.selected-alternatives`. It created a cycle:
`engine → casehub-persistence-memory → engine`.

Maven detects cycles at reactor resolution — before any module compiles:

```
[ERROR] The projects in the reactor contain a cyclic reference:
Edge between 'io.casehub:casehub-persistence-memory' and 'io.casehub:engine'
introduces cycle: engine → casehub-persistence-memory → engine
```

`test` scope doesn't help. Maven's cycle detection is graph-level, not scope-aware.

The fix was to copy the three in-memory implementations directly into
`engine/src/test/java/`. Quarkus automatically indexes test sources for
`@QuarkusTest`, so `selected-alternatives` picks them up without any additional
configuration. No module dependency, no cycle.

The assumption that test scope makes a circular dependency safe is exactly the
kind of thing that wastes an hour.

## Where things stand

Engine tests run without Docker. 353 tests pass. `EngineDecouplingIT` confirms
the in-memory repositories are active and the decoupling holds end-to-end.

PR #75 is open against casehubio/engine. It depends on the persistence-memory
work from PR #72 or #73 landing first — treblereel has his own version in
review. Once one of those merges, #75 rebases cleanly on top.
