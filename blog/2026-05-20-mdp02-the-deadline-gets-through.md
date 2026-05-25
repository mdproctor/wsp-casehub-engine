---
layout: post
title: "The Deadline Gets Through"
date: 2026-05-20
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-engine]
tags: [qhorus, commitments, propagation-context, podman, testcontainers]
---

Claudony is about to start tracking Commitments — Qhorus obligation records that bound how long an agent has to act on a COMMAND. To set an expiry, it needs the case budget deadline. The problem: claudony's `postToChannel` implementation has no access to the engine's `CaseInstance`. The content JSON is the only thing it receives.

`WorkerScheduleEventHandler.dispatchCommand()` was building its COMMAND content as `Map.of(...)` — four fixed fields, immutable by definition. Adding a fifth conditional field meant switching to a `HashMap`. The change itself is five lines. What's more interesting is what surfaced around it.

During brainstorming, I noticed `CaseHub.startCase()` takes `Map<String, Object>`. The quarkus-flow `Flow.instance()` method takes `Object` — it will serialise whatever you hand it, Map or POJO or anything else JSON-compatible. Our public API makes the wrong assumption. The internal handlers are correct: by the time `inputData` reaches `WorkerScheduleEventHandler` it's already been evaluated from the case context via `inputMapping` — Map is the right type at that layer. But the entry point shouldn't force callers to pack their domain objects into a map. Filed as engine#302 rather than fix it in this branch.

Testing required Testcontainers and Podman. Podman was freshly installed, machine never started. The test run failed immediately:

```
Could not find unix domain socket. Root cause NoSuchFileException (/var/run/docker.sock)
```

Without the `podman-mac-helper` installer, Podman doesn't create `/var/run/docker.sock`. It creates its own socket under `/var/folders/...` and prints the `export DOCKER_HOST=...` line during `podman machine start` — easy to miss. Setting `DOCKER_HOST` to that path unblocked everything.

The TDD cycle was straightforward once Podman was cooperating. Two integration tests in `SpiWiringIntegrationTest`: one starts a case with a `PropagationContext` carrying a one-hour budget and asserts the COMMAND content contains a `"deadline"` field; the other starts without a budget and asserts the field is absent. Claude caught a fragility in the second test during review — `noneMatch` on a shared static list across tests, which could pick up deadline entries from the first test if execution order went wrong. The fix: filter by the specific document ID used in that test, not just by message type.

The COMMAND content schema is now a five-field contract between the engine and claudony. It's informal — a raw `Map` on both sides — and it'll need formalising eventually. That's engine#301.
