# casehub-engine — ARC42STORIES.MD

**Spec:** Arc42Stories v0.1
**Profile:** CaseHub — Foundation tier
**Profile ref:** `../parent/docs/arc42stories-casehub-profile.md` · fallback: `https://raw.githubusercontent.com/casehubio/parent/main/docs/arc42stories-casehub-profile.md`
**Build position:** After casehub-work-core, optionally after casehub-ledger and casehub-eidos-api. The top of the foundation stack.
**Consumed by:** All application-tier harness apps (via YamlCaseHub / CaseHubRuntime), claudony (implements engine SPIs)
**Depends on:** casehub-work-core (WorkItemLifecycleEvent + HumanTaskTarget), optionally casehub-ledger (CaseLedgerEntry), optionally casehub-eidos-api.

---

## §1 Introduction and Goals

### Description

`casehub-engine` is the hybrid choreography+blackboard orchestration engine. It executes CasePlanModel definitions — declarative goal+binding case definitions expressed in YAML — by evaluating binding conditions against an accumulated CaseContext (the blackboard) and dispatching work when conditions are satisfied.

Core capabilities: `CasePlanModel` (YAML definition: goals, bindings, capabilities), `CaseHubReactor` (event-loop reactor — evaluates binding conditions on CaseContext change), `CaseContextChangedEvent` (fired on every blackboard update — triggers reactor), `GoalReachedEvent` (fired when a goal condition is satisfied), `WAITING` state (durable suspension pending async result — survives restarts via PendingWorkRegistry), `CaseLedgerEntry` (per-binding-dispatch audit record via casehub-ledger integration), `TrustWeightedAgentStrategy` (via casehub-engine-ledger — selects agents by Bayesian trust score), `YamlCaseHub` (base class for loading YAML CasePlanModels from classpath).

Module structure: `api` (CasePlanModel, CaseContext, CaseHubRuntime SPI, binding condition types), `core` (reactor, event handlers, goal evaluator), `blackboard` (CaseContext implementation, event bus), `persistence-memory` (in-memory SPIs for @QuarkusTest), `scheduler-quartz` (Quartz-based scheduled goal evaluation), `work-adapter` (bridges casehub-work WorkItem events to engine PlanItem completion), `ledger` (CaseLedgerEntry + TrustWeightedAgentStrategy + WorkerDecisionEventCapture).

### Stakeholders

| Stakeholder | Interest |
|---|---|
| Application-tier harness apps | Consume this module's foundation primitives |
| Platform team | Maintain correctness and backward compatibility |
| LLM sessions (Claude) | Navigate architecture and implement extensions confidently |

### Quality Goals

| Goal | Scenario |
|---|---|
| Content-driven routing | Binding conditions evaluate accumulated CaseContext — routing adapts to what code/data actually contains, not what authors label |
| Durable suspension | WAITING state survives Quarkus restarts via PendingWorkRegistry — no case is lost on restart |
| Declarative goals | Goals state what must be achieved; bindings state the mechanism. These are independent — binding structure can change without changing goal definition |

---

## §2 Constraints

### Platform
Java 21 (on Java 26 JVM), Quarkus 3.32.2. Published to GitHub Packages as `io.casehub:casehub-engine:0.2-SNAPSHOT`.

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install
```

### Dependencies
Resolved from GitHub Packages: `https://maven.pkg.github.com/casehubio/*`. CI authentication via `GITHUB_TOKEN`.

---

## §3 Context and Scope

🔲 C4Context diagram — add in dedicated session.

**Boundary rules:** casehub-engine owns only the concerns described in §1. Domain logic (business rules, application-specific workflows) belongs in application-tier repos (devtown, aml, clinical, life, drafthouse, quarkmind).

See `../parent/docs/PLATFORM.md` Capability Ownership table for the full boundary map.

---

## §4 Solution Strategy

🔲 Core architectural patterns and module structure — complete in dedicated session from DESIGN.md and repo architecture.

---

## §5 Building Block View

🔲 C4Container diagram and module structure — complete in dedicated session.

---

## §6 Runtime View

🔲 Key runtime scenarios — complete in dedicated session.

---

## §7 Deployment View

Published to GitHub Packages. CI: GitHub Actions. No GraalVM native target (foundation modules).

---

## §8 Crosscutting Concepts

References to governing protocols:

| Concern | Protocol / Reference |
|---|---|
| Module structure | `docs/protocols/universal/module-tier-structure.md` |
| Flyway migrations | `docs/protocols/universal/flyway-migration-rules.md` |
| CDI displacement (`@DefaultBean`) | `docs/protocols/casehub/alternative-extension-patterns.md` |
| Capability ownership | `../parent/docs/PLATFORM.md` Capability Ownership table |
| Architectural patterns | `../parent/docs/ARCHITECTURE.md` |

🔲 Anti-patterns inline (Symptom → Cause → Fix) — complete in dedicated session. Must be present per profile rules; do not defer to AGENTIC-HARNESS-GUIDE.md reference alone.

---

## §9 Journeys and Chapters

🔲 **Bootstrap — full Journeys and Chapters pending dedicated session.**

Foundation modules use Chapters to track delivery milestones of the foundation capability itself (transport types, SPI increments, new adapters) rather than integrating external foundation modules.

Read `../parent/docs/arc42stories-spec.md §4 Solution Strategy` — "Foundation: Chapters = delivery milestones of the foundation capability itself."

The full §9 entry requires:
- Reading DESIGN.md (or equivalent) for the delivery arc
- Mapping milestones to Chapters
- Writing §9.4 Layer/Chapter entries with all 18 required subsections

---

## §10 Architectural Decisions

Per the Arc42Stories spec: only decisions not captured inline elsewhere. Should be sparse.

🔲 Cross-cutting decisions — complete in dedicated session.

---

## §11 Quality Requirements

See §1 Quality Goals for the primary scenarios.

🔲 Full quality requirements table — complete in dedicated session.

---

## §12 Risks and Technical Debt

🔲 Active risks and technical debt — complete in dedicated session.

---

## §13 Glossary

| Term | Definition |
|---|---|
| `CasePlanModel` | YAML definition of a case: goals (what must be achieved) + bindings (conditions + capability dispatch). Evaluated by CaseHubReactor. |
| `CaseContext` | Shared blackboard for a case instance. Binding conditions read it; agent outputs write to it. Append-only during case lifetime. |
| `Binding` | A condition + capability pair. When the JQ condition evaluates true against CaseContext, the engine dispatches work to an agent with the named capability. |
| `WAITING state` | Durable engine state when a case is suspended pending an async result (human approval, CI run). Persisted via PendingWorkRegistry. |
| `YamlCaseHub` | Base class that loads a YAML CasePlanModel from the classpath and registers it with CasePlanModelRegistry. |
| `TrustWeightedAgentStrategy` | Agent selection strategy — routes to the highest-trust agent meeting minimumObservations; falls back to availability in Phase 0 (BOOTSTRAP). |

---

*Bootstrap document — generated 2026-06-02. §3–§9 require a dedicated session to complete from repository source material.*
