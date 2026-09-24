# casehub-langchain4j — Strategy and Positioning

## Why This Repo Exists

LangChain4j is the de facto standard Java library for building LLM-powered
applications. It provides model providers, memory stores, embedding stores,
RAG pipelines, tool calling, and agentic orchestration patterns. It does
this well.

What it does not provide — by design — is enterprise infrastructure:
cryptographic audit trails, multi-tenant isolation, human-in-the-loop
governance, or compliance evidence. These are cross-cutting concerns that
sit above any single LLM framework.

`casehub-langchain4j` adds these enterprise concerns to langchain4j
applications. It does not replace langchain4j's components — it wraps them
with capabilities langchain4j deliberately leaves to the deployer.

## The Additive Principle

CaseHub is additive. A langchain4j application works without CaseHub. Each
`casehub-langchain4j` module adds one enterprise concern:

- **audit** — tamper-evident audit trail for every LLM call and agent
  invocation. Cryptographic inclusion proofs, not just logs.
- **tenancy** — tenant isolation on any memory store, embedding store, or
  content retriever. Works with whatever backing implementation the
  developer already chose.
- **governance** — human oversight gates, trust-weighted agent routing,
  and causal lineage for agent invocations.
- **hybrid-search** — hybrid SPLADE + dense retrieval with RRF fusion,
  filling an acknowledged gap in langchain4j's RAG capabilities.

The developer keeps their chosen langchain4j providers, stores, and
patterns. CaseHub wraps them, never replaces them.

## Decorator Over Replacement

Three of the four lead modules are decorators:

| Module | Wraps | Adds |
|---|---|---|
| audit | Any `ChatModel` (via `ChatModelListener`) | Ledger entries |
| tenancy | Any `ChatMemoryStore`, `EmbeddingStore`, `ContentRetriever` | Tenant isolation |
| governance | Any `AgentListener`-compatible agent | Oversight, trust, lineage |

The fourth module — hybrid-search — is a `ContentRetriever` implementation,
not a decorator. It exists because langchain4j has explicitly acknowledged
this gap (issue [#4087](https://github.com/langchain4j/langchain4j/issues/4087)).

This distinction matters. Decorators are unambiguously additive — they
cannot be mistaken for a competing implementation. SPI implementations
require justification (see Expansion Policy below).

## What We Do Not Do

- We do not replace langchain4j's memory stores (Redis, PostgreSQL,
  Infinispan, in-memory). We decorate them with tenant isolation.
- We do not replace langchain4j's model providers (OpenAI, Anthropic,
  Ollama, etc.). We observe them via listeners.
- We do not replace langchain4j's guardrails validation. We add governance
  workflows (human approval, trust routing) that operate at a different
  level.
- We do not replace langchain4j's OTel-based observability. We add
  cryptographic compliance evidence that OTel does not provide.

## Community Evidence

The following langchain4j issues and discussions informed this positioning:

| Area | LC4j Status | Evidence | CaseHub Position |
|---|---|---|---|
| Cryptographic audit | Not planned | No issue or discussion mentions tamper-evident ledgers | Safe — different concern entirely |
| Multi-tenancy | Left to implementer | [#1889](https://github.com/langchain4j/langchain4j/issues/1889) — memory eviction in multi-user apps. [PR #6303](https://github.com/langchain4j/langchain4j/pull/6303) — `@A2ATenantId` for A2A routing only | Safe — no framework-level tenancy SPI |
| Governance / human approval | Not in scope | No issue mentions oversight gates or approval workflows | Safe — complementary to OTel tracing |
| Hybrid search | Acknowledged gap | [#4087](https://github.com/langchain4j/langchain4j/issues/4087) — "RAG capabilities primarily focus on vector similarity search" | Safe — welcomed, no LC4j solution |
| OTel observability | Active | [PR #3679](https://github.com/langchain4j/langchain4j/pull/3679) — auditing backport to core | Competitive — we do NOT build OTel features |
| Guardrails validation | Active | Guardrails API shipped in 1.x, actively expanding | Competitive — we do NOT rebuild guardrails validation |
| Cross-encoder reranking | Exists | In-process ONNX scoring models documented | Competitive — we do NOT ship a competing ScoringModel |
| Agent lifecycle tracing | Planned | [#4098](https://github.com/langchain4j/langchain4j/issues/4098) — `agentRunId`, `parentRunId` | Complementary — we add cryptographic causal chains, not OTel spans |
| Memory lifecycle | Planned | [Quarkus WG discussion](https://github.com/quarkusio/quarkus/discussions/50582) — scoping, resource reuse | Do not compete — wait for upstream |
| Resilience | Planned | Same WG discussion — retry, rollback, durability | Do not compete — wait for upstream |

## Expansion Policy

Modules beyond the four leads are permitted. Each requires documented
justification in its README under a `## Why this module exists` section.
Three valid justifications:

1. **Gap acknowledged** — cite a langchain4j issue where maintainers
   acknowledge the gap. Example: hybrid search ([#4087](https://github.com/langchain4j/langchain4j/issues/4087)).

2. **Different category** — the module provides something langchain4j does
   not attempt. Example: case-based reasoning memory is not a competing
   `ChatMemoryStore` implementation; it is a different category of memory
   (outcome tracking, similarity-based retrieval, retention policies).

3. **Customer requested** — a CaseHub deployer needs it and the langchain4j
   community alternative is insufficient for their enterprise requirements.

If none of these three justifications applies, the module should not exist.
When in doubt, check whether langchain4j's own issue tracker or roadmap
covers the capability — if it does, wait for upstream.

## Relationship to Existing Work

| Existing artifact | What happens |
|---|---|
| `platform/agent-langchain4j-core` | Moves to this repo as `agents-core` |
| `platform/agent-langchain4j` | Moves to this repo as `agents` |
| `platform/agent-langchain4j-spring` | Moves to this repo as `agents-spring` |
| `blocks#150` (LC4j agent→worker bridge) | Becomes a child issue scoped to the governance module |
| `engine#101` (supervisor pattern mapping) | Reference architecture for Phase 3 showcase |
| `engine#209` (LC4j agentic integration) | Builds on the governance module |
| `casehubio/quarkus-langchain4j` (fork) | Remains as upstream reference; this repo is NOT a fork |

## Three-Phase Delivery

| Phase | Message | Scope |
|---|---|---|
| **Phase 1** | "Your LC4j app is now compliant, multi-tenant, and governed" | audit, tenancy, governance, hybrid-search |
| **Phase 2** | "Your LC4j app has enterprise-grade RAG and memory" | CBR memory, SPLADE embedding, corpus ingestion (each justified) |
| **Phase 3** | "Here's what becomes possible with the full CaseHub stack" | Reference architectures demonstrating integrated capabilities |

Phase 3 is where natural migration happens — not because we forced it, but
because the developer saw the value and wants more.
