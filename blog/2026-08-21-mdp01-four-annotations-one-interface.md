---
layout: post
title: "Four Annotation Layers, One Interface"
date: 2026-08-21
entry_type: article
subtype: diary
projects: [casehub-engine]
tags: [annotations, composition, eidos, work, ledger, examples]
---

An agent that assesses aircraft defects needs to be cautious and methodical. Its inspector sign-off requires two independent approvals. Every step must produce an audit trail that satisfies EASA Part-145 compliance. These are four different concerns — what the agent *does*, who it *is*, when *humans* must approve, and what gets *audited* — and they belong to four different teams, four different repos, four different release cycles.

CaseHub's annotation model lets you declare all four on a single Java interface.

```java
@Identity(slot = "maintenance-engineer", jurisdiction = "EU")
@Disposition(ruleFollowing = "strict", riskAppetite = "risk-averse")
@Case(namespace = "aviation", name = "AircraftMaintenance")
public interface AircraftMaintenanceCase {

    @Worker(capability = "certifyAirworthy")
    @HumanApproval(title = "Certify aircraft airworthy")
    @RequiresQuorum(instances = 2, required = 2)
    @Audited
    @ComplianceSupplement(algorithmRef = "easa-part-145-release-to-service")
    default AirworthinessCertification certify(...) { ... }
}
```

Five annotations from four repos. The engine build extension processes `@Case` and `@Worker`. The eidos build extension reads `@Identity` and `@Disposition` and generates an `AgentDescriptor`. The work build extension wires `@HumanApproval` and `@RequiresQuorum` into the human task pipeline. The ledger build extension binds `@Audited` and `@ComplianceSupplement` to the audit interceptor chain. Each extension runs independently — engine first, then the others in any order — and each produces its own CDI beans without modifying the `CaseDefinition` that engine generated.

This is the key design choice: annotations compose by *addition*, not by mutation. The engine doesn't know about eidos. Eidos doesn't know about ledger. They produce independent beans that the CDI container wires together at runtime.

## Why four layers, not one

The temptation is to put everything on `@Case` — add a `disposition` attribute, a `humanApproval` flag, an `audited` boolean. That collapses the moment you consider release cycles. The engine team ships agent execution semantics. The eidos team ships personality models. The work team ships human-in-the-loop governance. The ledger team ships compliance audit. If they share an annotation, they share a release — and a single breaking change in one team's attribute blocks every other team's deployment.

Separate annotations mean separate modules, separate build extensions, separate dependency graphs. A consumer who doesn't need audit doesn't pull in the ledger module. A consumer who doesn't need personality routing doesn't pull in eidos. Progressive disclosure at the dependency level, not just the API level.

## The domains make it concrete

I built seven examples to demonstrate this, deliberately choosing domains where the composition is undeniable rather than decorative:

**Incident response** (cybersecurity) — a cautious AI analyst triages severity, but a human must approve containment before any server gets quarantined. Every action is audited for NIST SP 800-61 compliance. A periodic cron scan triggers re-evaluation. The personality constraints (`riskAppetite = "cautious"`, `autonomy = "bounded"`) aren't flavour text — they shape how the routing layer scores this agent against tasks.

**Search and rescue** — GOAP plans the operation from typed dependencies. The planner won't deploy drones without a condition assessment. It won't authorise evacuation without a medical assessment. Safety constraints (`@AgentConstraints`) declare hard limits: no air assets in 40-knot winds. The field rescue team is a standalone `@Capability` — not an AI agent, not an in-process function, just a capability that must be satisfied by an external team before the plan can advance.

**Aircraft maintenance** — the compliance showcase. `@ComplianceSupplement(algorithmRef = "easa-part-145-release-to-service")` on the certification step isn't metadata — it triggers the ledger interceptor to attach the algorithm reference, contestation URI, and human-override flag to the audit entry. `@RequiresQuorum(instances = 2, required = 2)` means two licensed inspectors must independently approve before the case can complete. This is how real MRO sign-off works.

**Warehouse fulfillment** — GOAP with a `@SoftDependency` on hazmat clearance. The dispatch step can proceed without hazmat clearance (most orders don't contain hazardous materials), but when the quality check flags hazmat items, the planner prefers the path that includes clearance — soft preconditions bias the plan without blocking it.

**Wildfire response** — multi-agency coordination with `@AgentGoals` that make the priority hierarchy explicit: protect life first, contain spread second. Ground containment is a standalone `@Capability` — the planner can reason about it, but the engine doesn't pretend it can dispatch a fire crew.

## The layered design

Each example is deliberately structured so that blocks — the orchestration pattern layer — can enhance it later. The containment decision in incident response is a natural seam for a `@DebateAgent` between aggressive containment and business-continuity perspectives. The inspector sign-off in aircraft maintenance maps to `@OversightGate` with trust-weighted routing. The resource allocation in wildfire response is a candidate for `@VotingAgent` across agency commanders.

The examples work today as pure engine + eidos + work + ledger compositions. When blocks annotations ship, the same interfaces gain orchestration patterns without restructuring. That's the goal: annotation layers compose, and each layer adds capability without requiring the others to change.
