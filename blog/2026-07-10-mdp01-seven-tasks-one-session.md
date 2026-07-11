---
title: "Seven tasks, one session: ContextBridge implementation day"
date: 2026-07-10
type: diary
entry_type: note
subtype: diary
projects: [casehub-engine]
tags: [contextbridge, generics, pipeline, cross-repo]
---

Seven of eight implementation tasks for the ContextBridge protocol landed today. The protocol was fully designed and adversarially reviewed yesterday — today was pure execution. The pipeline now carries typed context from scheduling through execution, with bridge resolution, EventLog metadata threading, and three built-in bridges (Map identity, Jackson POJO, JsonNode pass-through).

The session started with a surprise: `WorkerFunction<T>` was already parameterised from a previous attempt that got reset. The uncommitted changes in the worker repo were sound — the `Sync<T>` record, `inputType()`, the type token on `None`. What hadn't been done was fixing the downstream blast radius. `Worker.Builder.function()` still called `new Sync(fn)` with one arg against a two-arg constructor. The builder itself needed `fn()` and `TypedFunctionBuilder` for the Reified Varargs Type Token DSL.

The cross-repo migration was the session's heaviest lift. ~60 files across three repos (worker, engine, workers) needed `new WorkerFunction.Sync(fn)` updated to `new WorkerFunction.Sync<>(Map.class, fn)`. A bulk script approach was blocked by the IntelliJ-first hook — correctly, since the Edit tool's exact matching is safer than regex for this kind of substitution. We worked through it file by file, dispatching subagents for the larger batches (37 engine runtime tests, 15 blackboard/resilience tests). Two files used `java.util.Map.of()` with FQN imports and needed `java.util.Map.class` instead of `Map.class` — caught by the build, not by the replacement.

The IntelliJ MCP plugin had a gotcha with record declarations. `ide_edit_member` with `member=RecordName` returned "member_not_found" for top-level records. The fix — passing both `class=RecordName` and `member=RecordName` — only worked after an MCP reconnect. The first attempt without reconnect corrupted `AgentWorkerFunction.java` by nesting a new record inside the existing one. Garden entry GE-20260710-0b8c2a captures this for future sessions.

An architecture decision surfaced during Task 6: `BridgeResolver` was initially in `runtime`, but `QuartzWorkerExecutionJob` in `scheduler-quartz` needs it, and `scheduler-quartz` can't depend on `runtime`. Moved to `common/internal/context/` — the same pattern as other shared infrastructure (`WorkerExecutor`, `WorkerFunctionHandler`). The module dependency graph enforces this: shared infrastructure that both `runtime` and `scheduler-quartz` consume must live in `common`.

The remaining task is integration tests — `@QuarkusTest` end-to-end verification that a typed POJO worker receives its input correctly through the full scheduling → execution → output pipeline. The production code is complete and compiling clean across the workspace (0 errors, ~40 pre-existing deprecation warnings).
