---
layout: post
title: "The Pin Was Two Lines. The CDI Was Not."
date: 2026-05-21
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-engine]
tags: [quarkus, cdi, arc, testing, alternative, priority]
---

`work-adapter` had a json-schema-validator pin sitting in the wrong place — it belonged in the root `pom.xml` alongside the other BOM overrides, not buried in the module that happened to need it first. Moving it was two lines of XML. Verifying it required running `work-adapter` tests. That's where things got interesting.

The test suite failed with a class-loading error:

```
TestEngine with ID 'junit-jupiter' encountered a critical issue during test discovery.
Could not load class: HumanTaskScheduleHandlerAtomicityTest
Caused by: DeploymentException: Found 32 deployment problems
```

Thirty-two unsatisfied CDI dependencies, all for standard persistence repositories. The repositories were correctly listed in `application.properties`. They worked fine in every other test class.

The culprit was `QuarkusTestProfile.getEnabledAlternatives()`. The atomicity test uses a custom profile to activate `FailingWorkItemStore` — a deliberate WorkItem store that throws on `put()`, used to test rollback atomicity. The profile's `getEnabledAlternatives()` listed only `FailingWorkItemStore`. The Javadoc says the method provides alternatives "in addition to those configured via `quarkus.arc.selected-alternatives`" — but in practice it replaces them entirely. Every globally-required alternative had to be re-declared in the profile's return value.

That fixed the class-loading failure. Six other tests still failed.

The pattern looked like test isolation: "expecting empty but was [WorkItem]", WorkItems leaking from one test into the next. The `@BeforeEach` was clearing the store with an instanceof check:

```java
if (workItemStore instanceof InMemoryWorkItemStore mem) {
    mem.clear();
}
```

CDI client proxies for `@ApplicationScoped` beans are subclasses of the target type — `proxy instanceof InMemoryWorkItemStore` should return true. Except `workItemStore` was injected via the `WorkItemStore` interface, not the concrete type. The active bean for that interface wasn't `InMemoryWorkItemStore` at all.

It was `JpaWorkItemStore`.

`InMemoryWorkItemStore` is annotated `@Alternative @Priority(1)`. In CDI 4.x, `@Priority` on an alternative activates it globally and it should win over non-alternatives. In Quarkus ARC 3.32.2, when the alternative comes from an external dependency JAR, this doesn't hold. `JpaWorkItemStore` — plain `@ApplicationScoped`, no `@Priority` — was winning silently. No warning, no ambiguity error, just the wrong bean resolving.

This also explained the success-path failures. `WorkItemService.create()` was calling `WorkItemStore.put()` on `JpaWorkItemStore` — WorkItems went to H2. The test's `workItemStore.scanAll()`, once we changed the injection to the concrete type, was reading from `InMemoryWorkItemStore`'s LinkedHashMap. Two completely separate backends, zero overlap.

We spent some time chasing what looked like a Hibernate session-caching problem — `WorkItem.listAll()` returning empty after `WorkItemService.create()` completed successfully. That was a red herring. The data was in H2 the whole time; we just weren't reading from H2.

The fix that worked: exclude the JPA implementation from CDI discovery in the test context, and explicitly select the in-memory replacement:

```properties
quarkus.arc.exclude-types=io.casehub.work.runtime.repository.jpa.JpaWorkItemStore
quarkus.arc.selected-alternatives=\
  io.casehub.work.testing.InMemoryWorkItemStore,\
  ...other alternatives...
```

With `JpaWorkItemStore` excluded, there's only one active `WorkItemStore`. The `@Priority(1)` annotation alone was never going to get there.

All 26 tests pass. Two garden entries filed — one for each of the non-obvious CDI behaviours. The pin is in the root pom.
