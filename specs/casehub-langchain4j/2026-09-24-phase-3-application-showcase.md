# Phase 3: Application Showcase — Spec

**Date:** 2026-09-24
**Prerequisite:** Phase 1 modules stable, Phase 2 CBR memory available
**Strategy:** [STRATEGY.md](STRATEGY.md)

---

## Purpose

Phase 3 demonstrates what becomes possible when langchain4j meets CaseHub's
full application stack. These are not additional library modules — they are
reference architectures in `examples/` that a developer can run, inspect,
and learn from.

The goal: a developer who arrives via langchain4j completes these examples
and understands CaseHub's full value proposition — without being forced to
adopt anything. Each example builds on the previous one, progressively
revealing deeper capabilities.

Phase 3 is where natural migration happens. The developer sees the value
and wants more. We never force the transition.

---

## Showcase 1: Governed Supervisor

### What It Demonstrates

A langchain4j `@SupervisorAgent` orchestrating a multi-step investigation.
CaseHub adds governance, audit, and trust — invisible to the LC4j code.

### Scenario

An AML (anti-money-laundering) triage supervisor receives a suspicious
transaction alert. It decides which sub-agents to invoke:

1. **KYC agent** — checks customer identity and risk profile
2. **Transaction analysis agent** — analyses transaction patterns
3. **Containment agent** — recommends account actions (freeze, flag, escalate)

### What CaseHub Adds (transparent to the LC4j code)

| Concern | What happens | LC4j code change |
|---|---|---|
| Audit | Every supervisor decision and sub-agent call → ledger entry with causal chain | None — audit module listener |
| Oversight | Containment agent's "freeze account" recommendation pauses for human approval | None — governance module gate |
| Trust | After 50 cases, agents with better outcomes are preferred for routing | None — governance module scoring |
| Tenancy | Each bank's data is isolated; same supervisor serves multiple tenants | None — tenancy module decorator |
| Compliance | EU AI Act supplement on every LLM-driven decision | None — audit module supplement |

### Implementation Structure

```
examples/governed-supervisor/
├─ src/main/java/
│   ├─ SupervisorAgentConfig.java      — @SupervisorAgent with 3 sub-agents
│   ├─ KycAgent.java                   — @Agent: identity check
│   ├─ TransactionAnalysisAgent.java   — @Agent: pattern analysis
│   ├─ ContainmentAgent.java           — @Agent: action recommendation
│   └─ AlertResource.java             — REST endpoint to submit alerts
├─ src/main/resources/
│   └─ application.properties          — Quarkus config, LLM provider, ledger
└─ pom.xml                            — deps: quarkus-langchain4j-ollama,
                                         casehub-lc4j-audit,
                                         casehub-lc4j-tenancy,
                                         casehub-lc4j-governance
```

### Key Teaching Points

1. The LC4j `@SupervisorAgent` code is identical to what a developer would
   write without CaseHub. Zero CaseHub annotations in the agent code.
2. Add `casehub-lc4j-audit` → run → inspect ledger entries. "My app is
   now auditable."
3. Add `casehub-lc4j-governance` → the containment agent's "freeze"
   recommendation now requires approval. Show the WorkItem being created,
   approved via REST, and the agent proceeding.
4. Run 50 alerts → show trust scores diverging between agents. The
   supervisor starts preferring the agent with better outcomes.

---

## Showcase 2: Adaptive Agent Selection with CBR

### What It Demonstrates

CaseHub's case-based reasoning remembers which agents performed well on
similar past cases and routes accordingly. The system learns from outcomes
without any ML training step.

### Scenario

A customer support system with three specialist agents:

1. **Billing agent** — handles payment, refund, subscription queries
2. **Technical agent** — handles product issues, bugs, integration problems
3. **Compliance agent** — handles data requests, privacy, regulatory queries

Customer queries arrive. Initially, all three agents have equal trust.
Over time, the CBR system tracks which agent resolved which type of query
successfully, and starts routing similar queries to the best-performing
agent.

### What CaseHub Adds

| Concern | What happens |
|---|---|
| CBR memory | Past case outcomes stored with feature vectors (query type, customer segment, complexity) |
| Similarity retrieval | New query matched against past cases to find best agent |
| Trust scoring | Agent outcomes feed back into trust scores |
| Explanation | "Routed to billing agent because 8 of 10 similar past cases were resolved by billing agent (avg similarity: 0.87)" |

### Implementation Structure

```
examples/adaptive-agent-selection/
├─ src/main/java/
│   ├─ SupportSupervisor.java          — @SupervisorAgent
│   ├─ BillingAgent.java               — @Agent
│   ├─ TechnicalAgent.java             — @Agent
│   ├─ ComplianceAgent.java            — @Agent
│   ├─ OutcomeRecorder.java            — records resolution outcomes
│   └─ SupportResource.java            — REST endpoint
├─ pom.xml                             — deps: Phase 1 modules + casehub-lc4j-cbr-memory
```

### Key Teaching Points

1. First 10 queries: random routing (no history).
2. Record outcomes (resolved/escalated/failed) after each query.
3. After 20 queries: show CBR similarity scores influencing routing.
4. Show the explanation: "why was this query sent to billing agent?"
5. Demonstrate adaptation: agent improves → more queries routed to it.

---

## Showcase 3: Cross-Boundary Lineage

### What It Demonstrates

An LC4j agent calls an MCP tool which dispatches a CaseHub worker which
invokes another LC4j agent. The full causal chain across all boundaries
is one tamper-evident ledger trail.

### Scenario

A code review pipeline:

1. **Review supervisor** (LC4j `@SupervisorAgent`) — receives a PR for review
2. Supervisor calls an MCP tool → **code analysis service** (external)
3. Code analysis dispatches a CaseHub worker → **security scanner**
4. Security scanner invokes an LC4j `@Agent` → **vulnerability assessor**
5. Results flow back up the chain to the supervisor

### What CaseHub Adds

Every step in this chain produces a `CaseLedgerEntry` with
`causedByEntryId` pointing to its parent. The full trace:

```
AGENT_INVOCATION_STARTED: review-supervisor
  └─ AGENT_TOOL_EXECUTED: mcp-code-analysis
       └─ WORKER_SCHEDULED: security-scanner
            └─ AGENT_INVOCATION_STARTED: vulnerability-assessor
                 └─ AGENT_INVOCATION_COMPLETED: vulnerability-assessor
            └─ WORKER_COMPLETED: security-scanner
       └─ AGENT_TOOL_COMPLETED: mcp-code-analysis
  └─ AGENT_INVOCATION_COMPLETED: review-supervisor
```

An auditor can query the ledger: "show me every step that led to this
security finding" → one chain, cryptographically verifiable.

### Implementation Structure

```
examples/cross-boundary-lineage/
├─ src/main/java/
│   ├─ ReviewSupervisor.java           — @SupervisorAgent
│   ├─ CodeAnalysisTool.java           — @Tool calling MCP server
│   ├─ SecurityScannerWorker.java      — CaseHub WorkerFunction
│   ├─ VulnerabilityAssessor.java      — @Agent
│   └─ LineageQueryResource.java       — REST endpoint to query lineage
```

### Key Teaching Points

1. Four different execution contexts (LC4j agent, MCP tool, CaseHub
   worker, LC4j agent) — one audit trail.
2. Query the lineage: given any entry, trace the full causal chain.
3. Verify integrity: Merkle proof for any entry in the chain.

---

## Showcase 4: SLA-Enforced Workflows

### What It Demonstrates

LC4j agents operating within CaseHub's work management: SLA timers,
escalation policies, failure rerouting, dead letter queues. The LC4j
code handles AI reasoning; CaseHub handles operational discipline.

### Scenario

An insurance claims processing pipeline:

1. Claim submitted → CaseHub creates a case with SLA (24h initial
   assessment, 72h full review)
2. **Assessment agent** (LC4j) performs initial triage
3. If SLA breaches → CaseHub escalates to senior reviewer (human)
4. If agent fails → CaseHub routes to backup agent (different model)
5. If backup fails → dead letter queue for manual handling

### What CaseHub Adds

| Concern | What happens |
|---|---|
| SLA timers | Case tracks elapsed time; fires breach events |
| Escalation | SLA breach → WorkItem for human reviewer |
| Failure rerouting | Agent error → retry with different agent |
| Dead letter | Repeated failures → DLQ for manual review |
| Progress tracking | Case dashboard shows status of all claims |

### Implementation Structure

```
examples/sla-enforced-workflows/
├─ src/main/java/
│   ├─ ClaimsCaseDescriptor.java       — case definition with SLA bindings
│   ├─ AssessmentAgent.java            — @Agent: initial triage
│   ├─ FullReviewAgent.java            — @Agent: detailed review
│   ├─ SlaBreachHandler.java           — escalation policy
│   └─ ClaimsResource.java            — REST endpoint
├─ src/main/resources/
│   └─ claims-case.yaml               — case definition DSL
```

### Key Teaching Points

1. LC4j agents are workers in a CaseHub case — they don't know about SLAs.
2. Submit a claim → watch the timer tick.
3. Slow the agent (add artificial delay) → watch SLA breach → watch
   escalation create a WorkItem.
4. Kill the agent (throw exception) → watch rerouting to backup agent.
5. Kill backup → watch DLQ entry appear.

---

## Showcase 5: Multi-Agent Deliberation

### What It Demonstrates

LC4j agents participating in structured debate via Qhorus channels. Agents
propose, critique, and synthesise positions. The deliberation is auditable,
the outcome is governed, and the best argument wins.

### Scenario

A medical diagnosis review board:

1. Three LC4j agents analyse a patient case independently
2. Each posts their diagnosis to a Qhorus channel
3. A critique round: each agent reviews the other two diagnoses
4. A synthesis agent reads all positions and critiques, produces a
   final recommendation
5. If diagnoses disagree significantly → oversight gate requires
   human clinician review

### What CaseHub Adds

| Concern | What happens |
|---|---|
| Structured debate | Qhorus channels enforce turn-taking and structured critique |
| Audit | Every position, critique, and synthesis → ledger entry |
| Governance | Disagreement threshold → human review required |
| Trust | Agents that produce diagnoses later confirmed by outcomes build trust |

### Implementation Structure

```
examples/multi-agent-deliberation/
├─ src/main/java/
│   ├─ DiagnosisAgent.java             — @Agent: analyses case
│   ├─ CritiqueAgent.java              — @Agent: reviews peer diagnoses
│   ├─ SynthesisAgent.java             — @Agent: produces final recommendation
│   ├─ DeliberationOrchestrator.java   — Qhorus channel setup + turn management
│   └─ ReviewBoardResource.java        — REST endpoint
```

### Key Teaching Points

1. Three agents independently analyse the same case.
2. Structured critique: each agent sees the others' positions.
3. Synthesis: one agent produces a final recommendation.
4. Disagreement triggers human review.
5. Show the full deliberation trail in the ledger.

---

## Delivery

### Prerequisites per showcase

| Showcase | Phase 1 | Phase 2 | Additional CaseHub |
|---|---|---|---|
| Governed supervisor | audit, tenancy, governance | — | — |
| Adaptive agent selection | audit, governance | CBR memory | — |
| Cross-boundary lineage | audit, governance | — | engine (MCP, workers) |
| SLA-enforced workflows | audit, governance | — | engine (SLA, DLQ), work |
| Multi-agent deliberation | audit, governance | — | qhorus (channels) |

### Delivery order (recommended)

1. **Governed supervisor** — depends only on Phase 1; best introduction
2. **Cross-boundary lineage** — demonstrates unique CaseHub capability
3. **SLA-enforced workflows** — shows operational discipline
4. **Adaptive agent selection** — requires Phase 2 CBR memory
5. **Multi-agent deliberation** — requires Qhorus; most advanced

### Relationship to existing tutorials

Phase 3 examples complement the AML, clinical, and devtown reference
architectures. Those demonstrate CaseHub-native development (YAML DSL,
bindings, workers). Phase 3 demonstrates the LC4j on-ramp: a developer
arrives via langchain4j patterns and discovers the platform through
familiar territory.

Both tutorial paths lead to the same destination — full CaseHub platform
adoption. They serve different entry points.
