---
layout: post
title: "Closing the Stigmergy Loop"
date: 2026-11-08
entry_type: article
subtype: diary
projects: [casehubio/engine]
tags: [stigmergy, swarm, spi, architecture, agency, per-agent-coordination]
---

# Closing the Stigmergy Loop

A few weeks ago I wrote about giving swarms a nervous system — `SwarmCoordinationListener` bridging engine's coordination events to agency. That solved one direction: engine tells agency what happened. The data plane was still one-way. Agency's per-agent decision loop could hear that roles shifted or convergence was reached, but it couldn't see the signal landscape that produced those conclusions. And it couldn't write anything back.

The events said "consensus happened." They didn't say what signals led there, which agents reinforced them, or how strong the trail is right now. An LLM reasoning about an agent's next move needs that data — not just the headline, but the environment.

## What was missing

CaseHub's stigmergy model follows the biological metaphor closely. Agents deposit pheromone-like signals into a shared environment. Other agents perceive those signals and adjust their behaviour. The engine handles the physics — deposit, decay, reinforcement, perception — and detects emergent patterns: signal consensus, interest convergence, coordination storms.

All of this lives behind an `internal` package boundary. Agency can't reach in, and shouldn't. But agency's per-agent LLM decision loop needs exactly this data to reason about each agent's neighbourhood. Without it, the LLM builds prompts from stale context and makes decisions blind to the pheromone trails its peers are leaving.

The write side was worse. `DecisionOutcome.WriteBack` — the sealed interface for what the LLM produces — already modelled signal deposits, observations, and context mutations. The shape existed. Nothing executed it. Every decision the LLM made about the environment vanished.

## Why signals stay case-scoped

The original design proposed per-agent signal filtering — each agent sees only signals matching its declared interests or team membership. I rejected this after thinking through what stigmergy actually is.

Stigmergy's power IS the shared environment. Ants don't have private pheromone channels. Every ant perceives every trail. The coordination emerges from the collective response to shared information, not from information hiding. The moment you filter signals per agent, you've turned stigmergy into messaging — a fundamentally different coordination model with fundamentally different properties.

If the LLM context window is too small for all the signals, that's agency's concern. Agency knows its context budget. Agency knows the agent's role. The engine's job is to provide the full environmental picture and let the intelligence layer decide what matters.

Observations are different — they're inherently agent-scoped, things a specific agent noticed. The API reflects this split: `perceiveSignals(UUID caseId)` returns the full signal landscape, `observations(UUID caseId, String agentId)` returns only what that agent recorded.

## Two SPIs, one cycle

The read and write paths serve different moments in the decision loop. The LLM reads state to build a prompt, reasons, then writes state back. Splitting them means a monitoring system can read without writing, a replay tool can write without reading, and test doubles stay focused.

```java
public interface AgentObservationProvider {
  Map<String, PerceivedSignal> perceiveSignals(UUID caseId);
  List<Observation> observations(UUID caseId, String agentId);
}

public interface AgentWriteBack {
  boolean depositSignal(UUID caseId, String agentId, String name,
                        double strength, @Nullable Duration halfLife);
  void writeObservation(UUID caseId, String agentId, Observation observation);
  void writeContext(UUID caseId, String agentId, String key, JsonNode value);
}
```

The naming is deliberate. The write SPI is called `AgentWriteBack`, not `StigmergyWriter`, because `writeContext` targets the case's WORKING data layer — not the stigmergy environment. Naming it after stigmergy would be a lie about what one-third of the interface does.

![Per-agent Stigmergy Read-Write Cycle](images/stigmergy-read-write-cycle.svg)

## Attribution turns data into social signals

Signals already carry their depositor identity — `Signal.sources()` tracks every agent that deposited or reinforced a signal. But `PerceivedSignal`, the read model consumers actually see, only exposed `lastSource`. An LLM seeing "signal URGENT_TRIAGE has strength 0.8, last deposited by agent-3" knows there's a signal. An LLM seeing "deposited by agents 1, 3, and 5" knows there's consensus. That's a qualitatively different input to reasoning.

We added `sources: Set<String>` to `PerceivedSignal`. It transforms the signal from a scalar data point into a social signal — not just how strong the trail is, but how many independent agents laid it down.

The same gap existed in coordination events. `SIGNAL_CONSENSUS` already included the full source set in its metadata, but `INTEREST_CONVERGENCE` only reported a count — how many agents were watching a hotspot, not which ones. Now it includes the full agent set. The difference matters: "your teammates are all watching the same key" carries more weight than "some agents are watching the same key."

## What this enables

With these SPIs in place, each agent in a swarm can run its own lightweight LLM decision loop that reads the signal landscape, sees which peers reinforced which signals, reasons about its local neighbourhood, and writes its decisions back into the shared environment for others to perceive.

This is the architecture that makes distributed stigmergic coordination real — not a central controller dispatching instructions, but autonomous agents coordinating through a shared medium. The engine provides the environment and the physics. Agency provides the intelligence. The SPIs are the membrane between them.

The engine side is complete. The agency integration — wiring `DistributedTopology` to these SPIs — is the next piece.
