---
title: "The Flat Map That Grew Three Dimensions"
date: 2026-06-10
tags: [casehub-engine, blackboard, memory, design]
---

The CaseContext in casehub-engine has been, since the beginning, a flat `Map<String, Object>`. Workers write their output into it, the engine writes its signals — `actionGateRejected`, `workItemEscalated`, whatever — and domain initialization data lands there too. One namespace for everything.

This was always a collision waiting to happen. A worker that returns a key named `actionGateRejected` would silently trigger engine behaviour. Semantic domain knowledge — the fraud threshold for a fraud-check case, the entity ID for a clinical trial — lives alongside mutable worker outputs with no distinction.

We wanted to fix this properly: named panels with typed access. Working memory (mutable, worker-owned), semantic memory (read-only domain context injected at case start), episodic memory (event history — what happened in this case run, plus inter-case history from prior cases). The design went through six revision rounds before implementation began.

Three things turned out to be non-obvious.

**The `asJsonNode()` change is a breaking change for every JQ expression.** `CaseContext.asJsonNode()` now returns a panel document: `{"working":{...},"semantic":{...},"episodic":{...}}`. Every binding filter, every `inputSchema`, every goal condition that previously wrote `.result` now needs `.working.result`. Mechanically simple; the surprise is the blast radius — 47 test files across four modules needed updating. A parse-time warning in `CaseDefinitionYamlMapper` catches unmigrated expressions without blocking load.

**`@ConsumeEvent` is compiled at build time.** The design called for panel-scoped Vert.x addresses — `casehub.context.changed.extracted` for user-defined panels, so bindings could subscribe only to the panel they care about. But `@ConsumeEvent` is a Quarkus Arc annotation resolved at build time, not runtime. There is no equivalent to `eventBus.consumer("dynamic.address", handler)` via annotation. The fix: a single `@ConsumeEvent` handler on the base address, with a `changedPanel` field in the event. The handler reads that field to filter binding evaluation. Panel-scoped addresses are still published for external consumers.

**Recovery must reconstruct panels, not flatten them.** `CaseStartedEventHandler` stores `instance.getCaseContext().asJsonNode()` as the `CASE_STARTED` payload. After the semantics change, that payload is the full panel document. The recovery service was constructing context via `new CaseContextImpl(payloadAsMap(payload))` — which would have taken `{"working":{...},"semantic":{...},"episodic":{...}}` and made "working", "semantic", and "episodic" into flat working panel keys. No error thrown; the recovered case would have had completely wrong state. The fix is `CaseContextImpl.fromPanelDocument(payload)`, which reads the panel keys and reconstructs each panel correctly.

Claude's code review surfaced two correctness bugs we had missed. The deep-copy utility recursed into nested `Map` values correctly but created `new ArrayList<>(list)` for `List` values — shallow-copying any maps inside those lists. The episodic workers list aliased its entries between a snapshot and the original, corrupting trigger evaluation. The second: the read-modify-write on the episodic panel was not atomic. `getList()` reads under the read lock and releases it; the modification runs unlocked; `set()` re-acquires the write lock. Under concurrent Quartz worker completions, the later write overwrites the earlier's increment. Both fixed via an `engineUpdate()` method that holds the write lock for the full sequence.

The episodic panel ended up with two sub-sections. The intra-case section is rebuilt from EventLog on recovery and updated by the engine after each worker completion or milestone. The inter-case section queries `ReactiveCaseMemoryStore` at case start, giving workers access to prior history for the same entity across all previous cases. The query must not include `withCaseId(currentCaseId)` — a brand-new case has no stored memories, so that filter always returns empty.

The Drools integration that depends on all this — WorkingMemoryBridge, typed facts from named panels — is the next piece.
