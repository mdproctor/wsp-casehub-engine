---
layout: post
title: "Worker registration as a speech act"
date: 2026-04-28
type: phase-update
entry_type: note
subtype: diary
projects: [casehub]
tags: [quarkus, cdi, design]
---

The design session for WorkerProvisioner wiring started as an infrastructure
question and ended as a philosophy question: what *kind of thing* is worker
registration?

The setup: three worker entry paths exist conceptually — static (declared in a
`CaseDefinition`), provisioned (engine spins one up on demand), and
self-registering (an external agent announces itself). All three were handled
by different ad-hoc mechanisms. We spent a session working through the design
with Claude, and the architecture resolved into something cleaner than I
expected.

The key move was `WorkerRegistry` as the single source of truth. Static workers
seed into it at case start. Provisioned workers enter when `provision()` returns.
Self-registered workers enter via a REST endpoint or CDI call. `WorkOrchestrator`
queries the registry rather than `CaseDefinition.getWorkers()` directly. One
canonical pool, one place to instrument, one place for `WorkerStatusListener`
callbacks to resolve completions.

The execution fork uses Java 21 sealed classes: `EngineWorkerExecution` carries
the function and runs via Quartz; `AgentWorkerExecution` has no function and
waits for a `WorkerStatusListener.onWorkerCompleted()` callback. The switch is
exhaustive at compile time — adding a third mode requires handling it everywhere,
which is what you want from a discriminator.

The more interesting moment came when we asked what the *discovery lineage* of
a worker means for trust. A worker introduced by a trusted provisioner and one
that self-announces are not equivalent. That difference has consequences: what
work it can be assigned, what enforcement applies, how far up the chain you have
to audit to explain a decision. And then it clicked: worker registration is a
declarative speech act. Saying "I am a participant in this system" is a normative
event, not just a bookkeeping one.

Qhorus has already formalised exactly this — a four-layer normative framework
covering speech act theory, deontic logic, defeasible reasoning, and social
commitment semantics. The registration event produces an `EventLog` entry with
`discoveryMode`, `actorId`, `trustLevel`, and `causedByEntryId` linking back to
the registration entry of whoever introduced this worker. The discovery chain is
permanently traversable. We wrote ADR-0006 to record the decision, and added a
cross-reference section into the Qhorus spec so neither side of the ecosystem
forgets the connection exists.

I'd been treating the normative framework as a Qhorus-specific mechanism. It
isn't. Any participation event in the ecosystem — any moment where something
declares itself a valid actor — is a speech act with deontic consequences. The
framework gets more valuable the more consistently it's applied.
