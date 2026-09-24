# Design: casehub-langchain4j — Enterprise Enrichment for LangChain4j

**Date:** 2026-09-24
**Repo:** casehubio/casehub-langchain4j (new)
**Tier:** Integration
**Decisions:** [decisions.md](decisions.md)

---

## Principle

CaseHub is additive. A langchain4j application works without CaseHub. Each
module adds an enterprise concern that langchain4j deliberately does not
handle — audit, tenancy, governance. The developer keeps their chosen LC4j
providers, stores, and patterns. CaseHub wraps them, never replaces them.

---

## Positioning

This repo is **complementary** to langchain4j, not competitive:

- We do not replace LC4j's memory stores, embedding stores, or model providers
- We add cross-cutting enterprise concerns that LC4j's maintainers consider
  out of scope (cryptographic audit, tenant isolation, governance workflows)
- Where we provide SPI implementations (hybrid search), it is only where LC4j
  has acknowledged the gap (#4087) and the community would welcome the contribution
- Future modules beyond the four leads require documented justification:
  (1) gap acknowledged by LC4j, (2) different category not competing, or
  (3) customer requested

---

## Module Structure

Flat sibling modules with the established CaseHub naming convention:

```
casehub-langchain4j/
├─ audit-core/           Pure Java — ChatModelListener → LedgerEntry
├─ audit/                Quarkus CDI wiring
├─ audit-spring/         Spring auto-configuration
├─ tenancy-core/         Pure Java — tenant-isolating decorators
├─ tenancy/              Quarkus CDI wiring
├─ tenancy-spring/       Spring auto-configuration
├─ governance-core/      Pure Java — AgentMonitor → oversight + trust + lineage
├─ governance/           Quarkus CDI wiring
├─ governance-spring/    Spring auto-configuration
├─ hybrid-search-core/   Pure Java — ContentRetriever with SPLADE + dense + RRF
├─ hybrid-search/        Quarkus CDI wiring
├─ hybrid-search-spring/ Spring auto-configuration
├─ agents-core/          Pure Java — ChatModel ↔ AgentProvider bridge (from platform)
├─ agents/               Quarkus CDI wiring (from platform)
├─ agents-spring/        Spring auto-configuration (from platform)
├─ bom/                  BOM importing all modules
└─ examples/             Minimal working examples per module
```

### Dependency direction

```
casehub-langchain4j modules
  → langchain4j-core           (LC4j SPIs)
  → casehub-platform-api       (CurrentPrincipal, tenancy)
  → casehub-ledger-api         (LedgerEntry, audit)
  → casehub-neocortex-rag-api  (hybrid-search module only)
  → casehub-engine-api         (governance module only — WorkerFunction)
```

No upstream CaseHub repo depends on this repo. The dependency arrow points
one way. The existing `platform/agent-langchain4j-*` modules move here (D7).

### Activation model

Classpath activation with graceful degradation (D3):

- **Quarkus:** `@DefaultBean` / `@Alternative @Priority` — enterprise bean
  activates when CaseHub deps are on classpath
- **Spring:** `@ConditionalOnClass` — auto-configuration activates when
  CaseHub deps are on classpath
- **Missing deps:** module logs a warning and no-ops. Never fails. Never
  forces the developer to add CaseHub dependencies they don't want.

---

## Module: audit

**LC4j hook:** `ChatModelListener`, `AgentMonitor`

**What it does:** Every LLM call and every agent invocation produces a
tamper-evident `CaseLedgerEntry`:

- Merkle Mountain Range inclusion proofs (RFC 9162)
- Ed25519 tlog-checkpoint signing
- EU AI Act Art.12 `ComplianceSupplement` on every AI decision
- Token usage, cost estimate, latency, model name
- Tenant ID, correlation ID for traceability

**How it activates:** The `audit-core` module implements `ChatModelListener`
and `AgentMonitor`. The framework module registers them as beans. Any
`ChatModel` the developer already uses — Ollama, OpenAI, Anthropic,
whatever — gets audited transparently.

**What the developer does:** Adds one dependency. No code changes. No
configuration. Their existing LC4j app is now compliant.

**Graceful degradation:**
- Ledger on classpath → cryptographic ledger entries
- Ledger absent, EventLog present → EventLog entries (no crypto)
- Neither present → slf4j logging only

**Why this is safe:** LC4j's observability is OTel spans and metrics
(PR #3679, issue #4098). Cryptographic compliance evidence (Merkle proofs,
Ed25519 signing, GDPR supplements) is entirely outside their scope.

---

## Module: tenancy

**LC4j hook:** Decorator pattern wrapping any SPI implementation

**What it does:** Wraps any `ChatMemoryStore`, `EmbeddingStore`, or
`ContentRetriever` with tenant isolation via `CurrentPrincipal.tenancyId()`.

For `ChatMemoryStore`:
- `getMessages(memoryId)` → scoped to current tenant
- `updateMessages(memoryId, messages)` → stored under current tenant
- `deleteMessages(memoryId)` → deletes only current tenant's messages
- Cross-tenant access is structurally impossible through the decorator

For `EmbeddingStore`:
- `add(embedding)` → tagged with tenant ID in metadata
- `search(request)` → filtered to current tenant's embeddings only

For `ContentRetriever`:
- `retrieve(query)` → filtered to current tenant's content only

**How it activates:** The `tenancy-core` module provides decorator classes.
The framework module discovers the developer's existing store/retriever beans
and wraps them with the tenant decorator. The developer's chosen implementation
(Redis, PostgreSQL, Qdrant, in-memory — whatever) is preserved inside the
decorator.

**What the developer does:** Adds one dependency. Their existing stores become
multi-tenant.

**Graceful degradation:**
- `CurrentPrincipal` on classpath → tenant isolation active
- `CurrentPrincipal` absent → decorator passes through (single-tenant mode)

**Why this is safe:** LC4j has no framework-level tenancy SPI. Issue #1889
acknowledges multi-user memory challenges. PR #6303 adds `@A2ATenantId` for
A2A routing but not store-level isolation. Their approach is "use memory IDs" —
tenant isolation is left to the implementer.

---

## Module: governance

**LC4j hook:** `AgentMonitor`, CDI/Spring interception of agent beans

**What it does:** Injects three enterprise concerns into any LC4j agent
pattern (supervisor, sequence, parallel, loop, conditional):

**1. Causal lineage** — every agent invocation produces a `CaseLedgerEntry`
with `causedByEntryId` chains. The full decision tree is traceable:
"supervisor called containment agent because triage scored high-risk."

**2. Oversight gates** — before a sensitive action executes, the governance
layer checks if human approval is required. If yes: execution pauses, a
`WorkItem` is created for the right approver group, execution resumes
only after human approval. The LC4j agent sees a delayed response.

**3. Trust routing** — when multiple agents can handle a task, trust scores
(Bayesian Beta, EigenTrust peer verdicts) influence selection. Agents with
good outcome history are preferred. Agents with rejected outputs are
deprioritised.

**How it activates:** The `governance-core` module implements `AgentMonitor`.
`beforeAgentInvocation()` checks oversight gates and creates lineage entries.
`afterAgentInvocation()` records outcomes for trust scoring. The framework
module registers the monitor. The developer's LC4j patterns are untouched —
governance operates at the invocation boundary.

**What the developer does:** Adds one dependency. Their existing LC4j agents
gain audit trail, human oversight, and trust-based routing.

**Includes the existing bridge:** The `agents-core` / `agents` / `agents-spring`
modules (moved from platform) provide the ChatModel ↔ AgentProvider
bidirectional bridge. This is the foundation the governance module builds on.

**Graceful degradation:**
- Ledger on classpath → cryptographic lineage entries
- Engine on classpath → oversight gates and trust routing active
- Neither present → AgentMonitor records to slf4j only

**Why this is safe:** LC4j's agent observability (#4098) is OTel-based
correlation (`agentRunId`, `parentRunId`). CaseHub's governance adds human
oversight gates, trust-weighted routing, and cryptographic causal chains —
complementary concerns that don't overlap with OTel tracing.

---

## Module: hybrid-search

**LC4j SPI:** `ContentRetriever`

**What it does:** Implements `ContentRetriever` with neocortex's full hybrid
retrieval pipeline:

- SPLADE sparse embeddings (learned term expansion)
- Dense vector embeddings (any LC4j `EmbeddingModel`)
- Reciprocal Rank Fusion (Qdrant server-side)
- Optional cross-encoder reranking (ONNX, in-process)

26–31% NDCG improvement over dense-only retrieval (architecture-justification.md §1).

**How it activates:** The `hybrid-search-core` module implements
`ContentRetriever`. The framework module registers it as a bean. The developer
replaces their `EmbeddingStoreContentRetriever` with `CasehubHybridContentRetriever`.

**What the developer does:** Swaps one bean (this is a replacement, not a
decorator — justified because LC4j has acknowledged the gap).

**Why this is safe:** LC4j issue #4087 explicitly requests a unified hybrid
search API. The maintainers acknowledge "RAG capabilities primarily focus on
vector similarity search" and "lacks complementary benefits of keyword-based
search." Open, no PR merged, no LC4j solution exists. SPLADE has no JVM
implementation outside neocortex.

---

## Expansion Policy

Modules beyond these four are not prohibited — they require documented
justification (D8). Three valid reasons:

1. **Gap acknowledged** — cite an LC4j issue where maintainers acknowledge
   the gap. Example: hybrid search (#4087).
2. **Different category** — the module provides something LC4j doesn't
   attempt. Example: CBR memory is not a competing `ChatMemoryStore`
   implementation; it's a different category of memory.
3. **Customer requested** — a CaseHub deployer needs it and the LC4j
   community alternative is insufficient for their enterprise requirements.

Each justification is documented in the module's README under a
"## Why this module exists" section.

---

## Impact on Existing Repos

### platform
- `agent-langchain4j-core/`, `agent-langchain4j/`, `agent-langchain4j-spring/`
  move to this repo as `agents-core/`, `agents/`, `agents-spring/`
- Platform retains `agent-api/` (the AgentProvider SPI)
- Platform pom.xml removes three modules
- Downstream consumers update dependency: `casehub-platform-agent-langchain4j`
  → `casehub-langchain4j-agents`

### engine
- Adds `casehub-langchain4j-agents` as dependency (replaces
  `casehub-platform-agent-langchain4j`)
- No code changes — same classes, new artifact coordinates

### blocks
- Issue #150 becomes a child issue scoped to the governance module in this
  repo rather than a standalone blocks feature

### parent
- New repo registration per `docs/new-repo-checklist.md` (22 steps)
- BOM entries for all published artifacts
- CI dispatch chain: platform, neocortex, ledger → casehub-langchain4j
- Dashboard, README badges, architecture diagram updates

---

## References

- [ADR-0004](../../blocks/docs/adr/0004-own-orchestration-annotations.md) — dual-track LC4j strategy
- [Engine #101](https://github.com/casehubio/engine/issues/101) — LC4j supervisor pattern mapping
- [Engine #209](https://github.com/casehubio/engine/issues/209) — LC4j agentic integration epic
- [Blocks #150](https://github.com/casehubio/blocks/issues/150) — LC4j agent→worker bridge
- [Platform interop spec](../../platform/docs/specs/2026-06-26-agent-langchain4j-interop-design.md) — ChatModel ↔ AgentProvider bridge
- [Neocortex architecture-justification.md](../../neocortex/docs/architecture-justification.md) — hybrid search evidence base
- [Parent new-repo-checklist.md](../../parent/docs/new-repo-checklist.md) — 22-step repo creation
- [LC4j #4087](https://github.com/langchain4j/langchain4j/issues/4087) — hybrid search gap
- [LC4j #4098](https://github.com/langchain4j/langchain4j/issues/4098) — agent observability
- [LC4j #1889](https://github.com/langchain4j/langchain4j/issues/1889) — multi-user memory
- [LC4j PR #3679](https://github.com/langchain4j/langchain4j/pull/3679) — auditing backport
- [LC4j PR #6303](https://github.com/langchain4j/langchain4j/pull/6303) — A2A TenantId
- [Quarkus WG Discussion](https://github.com/quarkusio/quarkus/discussions/50582) — agentic roadmap
