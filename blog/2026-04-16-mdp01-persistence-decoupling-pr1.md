---
layout: post
title: "Persistence PR 1: Container Chaos and a Quarkus Context Trick"
date: 2026-04-16
type: phase-update
entry_type: note
subtype: diary
projects: [casehub]
tags: [quarkus, hibernate, testing, podman, persistence]
---

The persistence decoupling plan was written last session — three PRs to strip JPA from the engine core. Today was PR 1.

I brought Claude in to write the implementation plan before touching code. Writing it surfaced real things: a `parentPlanItemId` field the spec notes had as `Long` but the actual code had as `UUID`; a risk that indexing the engine module in the Hibernate module's tests would cause Hibernate to discover duplicate `@Entity` mappings on the same tables; and a decision to leave the Flyway migration files in the engine JAR for now, since moving them in the same PR would require fixing the engine tests simultaneously.

Then we executed it with the subagent workflow. Eight tasks, review between each. Tasks 1–4 landed clean — SPI interfaces, module scaffold, entity classes. Task 5 is where it got interesting.

Claude came back from the first test run with `Please configure the datasource URL or ensure the Docker daemon is up and running`. The machine isn't running Docker Desktop; it's using Podman. Setting `DOCKER_HOST` to the Podman socket didn't help — `~/.testcontainers.properties` had `docker.client.strategy=UnixSocketClientProviderStrategy` hard-coded, which ignores the environment variable. The Podman machine turned out to be in a broken state, SSH handshake failing silently. I ran `podman machine rm --force` to reset it. That was a mistake — `rm` in Podman, as in Docker, means *destroy*, not force-stop. `--force` skips the confirmation prompt. The machine was gone.

We ran `podman machine init` and `podman machine start`. The new machine auto-created `/var/run/docker.sock`. That's what Testcontainers was looking for.

Then the second problem. All four tests failed: `IllegalStateException: No current Vertx context found`. Expected — `Panache.withSession()` requires a Vert.x event loop thread, and JUnit methods run on plain JVM threads. I wrapped the calls in `vertx.getDelegate().runOnContext()`. Different error: `Can't get the context safety flag: the current context is not a duplicated context`.

This is a Quarkus 3.x addition. Even on the event loop, the context must be explicitly *duplicated* and safety-marked. The documented fix is `@RunOnVertxContext`, which requires `io.quarkus:quarkus-test-vertx` — not in the standard `quarkus-junit5`. But `VertxContextSupport` in `io.quarkus:quarkus-vertx` is already on the classpath via `quarkus-hibernate-reactive-panache`. Its `subscribeAndAwait()` handles everything — no extra dependency, no change to test signatures:

```java
private <T> T run(Supplier<Uni<T>> supplier) {
    try {
        return VertxContextSupport.subscribeAndAwait(supplier);
    } catch (RuntimeException e) {
        throw e;
    } catch (Throwable e) {
        throw new RuntimeException(e);
    }
}
```

17 tests, all passing — 4 for the meta model repository, 4 for case instance, 9 for event log.

By then, upstream had merged everything from the base branch into main. Rebasing the PR branch onto main hit conflicts immediately — the branch contained the full history of the already-merged base, and replaying those commits produced conflicts on code that had already landed. The fix was to create fresh branches from `upstream/main` and cherry-pick only our 8 new commits.

The PR also became three. The upstream maintainer prefers reviewable pieces, and at ~1,150 lines the original was on the large side. The commits fell on natural boundaries — SPI interfaces alone (125 lines), module scaffold and entities (292 lines), repositories and 17 tests (733 lines). Three stacked PRs, each targeting the previous branch on the upstream repo: [#65](https://github.com/casehubio/engine/pull/65), [#66](https://github.com/casehubio/engine/pull/66), [#67](https://github.com/casehubio/engine/pull/67).
