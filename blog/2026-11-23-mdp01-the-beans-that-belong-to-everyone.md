---
layout: post
title: "The Beans That Belong to Everyone"
date: 2026-11-23
entry_type: note
subtype: diary
projects: [casehubio/engine]
tags: [cdi, quarkus, testing, agent-dispatch, isolation]
---

# The Beans That Belong to Everyone

Quarkus discovers CDI beans from test inner classes globally. Every `@ApplicationScoped` inner class in every `@QuarkusTest` in the module is visible to every other test. This isn't a bug — it's the CDI spec working as designed. But it creates a tension I hadn't fully appreciated until today.

The engine's `AgentWorkerFunctionHandler` injects five dependencies: `AgentRegistry`, `SystemPromptRenderer`, `AgentProvider`, `AgentSelector`, `ExpressionEngineRegistry`. In production, eidos modules provide these. In tests, most don't exist — `NoOpAgentRegistry` is a `@DefaultBean` fallback, and the others simply have no bean. The handler silently vanishes from the `Instance<WorkerFunctionHandler>` collection. No error, no warning. Tests that use `WorkerFunction.Sync` workers never notice, because `SyncWorkerFunctionHandler` picks those up instead.

To test slot-based agent dispatch end-to-end, I had to provide all five beans so the handler would actually be created. The first attempt — adding `casehub-eidos-memory` as a test dependency with a Jandex index — worked perfectly in isolation. `InMemoryAgentRegistry` with `@Alternative @Priority(1)` won the injection cleanly. The test passed. The full suite did not.

The problem: `GoalRevisionIntegrationTest` has its own `TestAgentRegistry`, a plain `@ApplicationScoped` inner class with custom `seed()` and `registrations()` methods for its specific test scenario. My `@Alternative` overrode it at the interface level. The test still injected its registry by concrete class, so it registered descriptors into `TestAgentRegistry` — but the handler resolved them from `InMemoryAgentRegistry`, which was empty. Slot lookup silently returned nothing.

The fix was to remove the global alternative and use the same pattern as the existing tests: a test-specific inner class, `@Alternative @Priority(2)`, injected by concrete type. The alternative wins the interface injection for the handler without disturbing tests that inject their own registries directly.

But this creates its own fragility. `StubAgentProvider` — the bean that returns fixed sentiment JSON for the slot test — is shadowed by `GoalFormationIntegrationTest.TestAgentProvider`, which is `@Alternative @Priority(1)`. My provider never gets used. The test still passes because the capability's output projection hardcodes `status: "analyzed"` regardless of the agent response. The assertion verifies the status change, not the response content. The CDI wiring is proven — the handler is created, the slot lookup works, the case completes — but the specific agent provider path isn't verified as tightly as it could be.

I filed this as [#1354](https://github.com/casehubio/engine/issues/1354). The proper fix is `@TestProfile` — which isolates CDI beans to a specific test class rather than sharing them globally. But `@TestProfile` changes configuration, and whether it can also scope bean alternatives is a Quarkus ARC question I'll need to investigate.

A second thing surfaced during setup: the work pipeline's `branch_name` judgment step stores the decided name in `args["branch"]` during the step completion call, but never persists it in the progress file. When the pipeline resumes from a new session, `args["branch"]` comes from `ctx.py`'s `CURRENT_BRANCH` — the checkout, not the decided name. The pipeline tried to create a branch called `issue-1339-agent-task-dispatch` for issue #1340 — because that's what was checked out, and the progress file had lost the name it decided in the previous session. Filed as [soredium#616](https://github.com/Hortora/soredium/issues/616). Two-line fix: persist the branch name in progress on step completion, read it back on context build.

The CDI isolation problem is the more interesting one. Quarkus's global bean discovery is the right default for production — you want all beans visible everywhere. But in tests, where each class provides its own stub implementations, global visibility means every stub competes with every other stub. The more tests that need custom agent infrastructure, the more alternatives pile up at escalating priorities. It works until it doesn't.
