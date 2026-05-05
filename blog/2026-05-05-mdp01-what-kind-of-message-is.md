---
layout: post
title: "What Kind of Message Is This?"
date: 2026-05-05
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-engine]
tags: [spi, protocol, qhorus, dependency-management]
---

The `CaseChannelProvider` SPI had a blind spot. Every call to `postToChannel` sent a message — but said nothing about what *kind* of message it was. When `WorkerScheduleEventHandler` dispatched a directive to a worker's channel, it serialised the intent into a JSON payload field: `"type": "COMMAND"`. The intent existed in the body, invisible to the protocol boundary.

Adding a `MessageType` parameter should have been straightforward. But the type had to come from somewhere.

## The Vocabulary Question

Qhorus already defines a speech-act vocabulary — `QUERY`, `COMMAND`, `RESPONSE`, `STATUS`, `DONE`, `FAILURE`, `EVENT` — mapping to deontic obligations between agents. I wanted to use it rather than invent a parallel one. The question was where the dependency should live.

I drew out the dependency graph before writing anything. Qhorus doesn't depend on the engine. The engine doesn't depend on Qhorus. Claudony bridges both. Adding engine → Qhorus for one enum couples two peer systems that are deliberately independent. The right long-term home is a `casehubio/protocol` repo that neither system owns — but that's real work: new CI, new release cycle, BOM updates. For now, the engine takes a narrow dependency on `casehub-qhorus-api` and uses `MessageType` directly. The extraction is filed for when a third consumer makes the work worthwhile.

The SPI change is a two-method design. The 4-arg signature becomes abstract:

```java
void postToChannel(CaseChannel channel, String from, String content, MessageType type);
```

The 3-arg becomes a default, delegating with `null`:

```java
default void postToChannel(CaseChannel channel, String from, String content) {
    postToChannel(channel, from, content, null);
}
```

Existing callers don't break. `WorkerScheduleEventHandler.dispatchCommand()` now passes `MessageType.COMMAND` explicitly. Intent that was buried in a JSON body is expressed at the protocol boundary.

## The Causal Chain, Almost

The second change is about lineage across repo boundaries. When Claudony provisions a worker, it writes a ledger entry. To link that entry back to the Qhorus COMMAND that triggered provisioning, it needs the channel ID and correlation ID. `ProvisionContext` now carries `triggerChannelId` and `triggerCorrelationId` — both nullable strings.

The engine's call site passes `null`, correctly. The engine doesn't receive Qhorus COMMANDs — Claudony does. For the values to reach `ProvisionContext`, the engine's CaseFile-update API needs to accept trigger context and thread it through the context-changed event. That threading is filed separately. The fields establish the contract; the plumbing follows.
