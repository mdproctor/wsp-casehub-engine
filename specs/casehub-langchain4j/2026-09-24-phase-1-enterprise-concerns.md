# Phase 1: Enterprise Concerns — Implementation Spec

**Date:** 2026-09-24
**Prerequisite:** Skeleton repo created, agents bridge moved from platform
**Deliverables:** 4 module groups × 3 framework variants = 12 new modules + BOM + examples
**Strategy:** [STRATEGY.md](STRATEGY.md)
**Decisions:** [decisions.md](decisions.md)

---

## Module Inventory

| Group | Modules | LC4j SPI | CaseHub SPI | Pattern |
|---|---|---|---|---|
| audit | audit-core, audit, audit-spring | `ChatModelListener`, `AgentListener` | `CaseLedgerEntry` | Listener (zero code change) |
| tenancy | tenancy-core, tenancy, tenancy-spring | `ChatMemoryStore`, `EmbeddingStore`, `ContentRetriever` | `CurrentPrincipal` | Decorator (zero code change) |
| governance | governance-core, governance, governance-spring | `AgentListener` | `OversightGateService`, `TrustSignalProvider`, `CaseLedgerEntry` | Listener + interceptor |
| hybrid-search | hybrid-search-core, hybrid-search, hybrid-search-spring | `ContentRetriever` | `CaseContextRetriever` | Implementation (bean swap) |
| agents (moved) | agents-core, agents, agents-spring | `ChatModel`, `StreamingChatModel` | `AgentProvider`, `AgentSession` | Bidirectional bridge |

---

## 1. audit

### Purpose

Every LLM call and every agent invocation produces a tamper-evident
`CaseLedgerEntry`. Zero code change — classpath activation only.

### audit-core (pure Java, no framework)

**Package:** `io.casehub.langchain4j.audit`

#### CasehubChatModelListener

```java
public class CasehubChatModelListener implements ChatModelListener {

    private final LedgerEntryWriter ledgerWriter;    // from casehub-ledger-api
    private final CurrentPrincipal currentPrincipal; // from casehub-platform-api (nullable)
    private final EventLog eventLog;                 // fallback (nullable)

    // Constructor accepts all three; null = graceful skip

    @Override
    public void onRequest(ChatModelRequestContext ctx) {
        // Extract: model name, messages, tools from ChatRequest
        // Store start timestamp in ctx.attributes() for latency calc
        // Create in-progress ledger entry with correlationId
    }

    @Override
    public void onResponse(ChatModelResponseContext ctx) {
        // Extract from ChatResponse: token usage (input, output, total),
        //   finish reason, model name
        // Calculate latency from stored start timestamp
        // Build CaseLedgerEntry:
        //   - entryType: "AI_CHAT_COMPLETION"
        //   - tenantId: currentPrincipal.tenancyId() (if available)
        //   - correlationId: from ctx.attributes()
        //   - payload: { model, tokens, cost, latency, finishReason }
        //   - complianceSupplement: ComplianceSupplement with:
        //       - aiModelId: model name
        //       - inputTokens, outputTokens
        //       - decisionType: "chat_completion"
        // Write via ledgerWriter (or eventLog fallback, or log)
    }

    @Override
    public void onError(ChatModelErrorContext ctx) {
        // Build CaseLedgerEntry with entryType "AI_CHAT_ERROR"
        // Include error class, message (not stack trace — PII risk)
        // Write via ledgerWriter
    }
}
```

#### CasehubAgentListener

```java
public class CasehubAgentListener implements AgentListener {

    private final LedgerEntryWriter ledgerWriter;
    private final CurrentPrincipal currentPrincipal;

    @Override
    public void beforeAgentInvocation(AgentRequest request) {
        // Extract: agent name, input text, memory ID from AgenticScope
        // Create CaseLedgerEntry:
        //   - entryType: "AGENT_INVOCATION_STARTED"
        //   - tenantId: from currentPrincipal
        //   - payload: { agentName, memoryId, inputSummary }
        //   - causedByEntryId: from parent invocation (if nested)
        // Store entryId in AgenticScope for causal chaining
    }

    @Override
    public void afterAgentInvocation(AgentResponse response) {
        // Create CaseLedgerEntry:
        //   - entryType: "AGENT_INVOCATION_COMPLETED"
        //   - causedByEntryId: the STARTED entry's ID
        //   - payload: { agentName, outputSummary, durationMs }
        //   - complianceSupplement (if LLM was used)
    }

    @Override
    public void onAgentInvocationError(AgentInvocationError error) {
        // entryType: "AGENT_INVOCATION_FAILED"
        // payload: { agentName, errorClass, errorMessage }
    }

    @Override
    public void afterAgentToolExecution(AfterAgentToolExecution toolExec) {
        // entryType: "AGENT_TOOL_EXECUTED"
        // payload: { toolName, resultSummary }
        // causedByEntryId: the parent AGENT_INVOCATION_STARTED
    }

    @Override
    public boolean inheritedBySubagents() {
        return true;  // single listener covers entire agent tree
    }
}
```

#### Graceful Degradation Helper

```java
public class AuditSink {
    private final LedgerEntryWriter ledger;  // nullable
    private final EventLog eventLog;          // nullable
    private final Logger log;

    public void write(LedgerEntryTemplate template) {
        if (ledger != null) {
            ledger.write(template.toLedgerEntry());
        } else if (eventLog != null) {
            eventLog.append(template.toEventLogEntry());
        } else {
            log.info("audit: {}", template.summary());
        }
    }
}
```

### audit (Quarkus CDI)

**Package:** `io.casehub.langchain4j.audit.quarkus`

```java
@ApplicationScoped
public class AuditBeanProducer {

    @Inject @Any Instance<LedgerEntryWriter> ledgerInstance;
    @Inject @Any Instance<CurrentPrincipal> principalInstance;
    @Inject @Any Instance<EventLog> eventLogInstance;

    @Produces @ApplicationScoped
    public CasehubChatModelListener chatModelListener() {
        return new CasehubChatModelListener(
            ledgerInstance.isResolvable() ? ledgerInstance.get() : null,
            principalInstance.isResolvable() ? principalInstance.get() : null,
            eventLogInstance.isResolvable() ? eventLogInstance.get() : null
        );
    }

    @Produces @ApplicationScoped
    public CasehubAgentListener agentListener() {
        return new CasehubAgentListener(
            ledgerInstance.isResolvable() ? ledgerInstance.get() : null,
            principalInstance.isResolvable() ? principalInstance.get() : null
        );
    }
}
```

The `ChatModelListener` registers automatically via LC4j's SPI discovery
(`META-INF/services/dev.langchain4j.model.chat.listener.ChatModelListener`).
For Quarkus, the CDI bean is discovered by ArC and registered with
quarkus-langchain4j's listener infrastructure.

The `AgentListener` is registered via `AgenticScope` configuration — the
Quarkus CDI extension discovers `AgentListener` beans and registers them
with the agentic runtime.

### audit-spring (Spring auto-configuration)

**Package:** `io.casehub.langchain4j.audit.spring`

```java
@AutoConfiguration
@ConditionalOnClass(ChatModelListener.class)
public class CasehubAuditAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean(CasehubChatModelListener.class)
    public CasehubChatModelListener casehubChatModelListener(
            @Autowired(required = false) LedgerEntryWriter ledger,
            @Autowired(required = false) CurrentPrincipal principal,
            @Autowired(required = false) EventLog eventLog) {
        return new CasehubChatModelListener(ledger, principal, eventLog);
    }

    @Bean
    @ConditionalOnMissingBean(CasehubAgentListener.class)
    public CasehubAgentListener casehubAgentListener(
            @Autowired(required = false) LedgerEntryWriter ledger,
            @Autowired(required = false) CurrentPrincipal principal) {
        return new CasehubAgentListener(ledger, principal);
    }
}
```

Register in `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`.

### Dependencies

| Module | Compile | Optional | Test |
|---|---|---|---|
| audit-core | langchain4j-core, langchain4j-agentic, casehub-ledger-api, casehub-platform-api | — | junit5, assertj, mockito |
| audit | audit-core, quarkus-arc | casehub-ledger, casehub-platform | quarkus-junit5 |
| audit-spring | audit-core, spring-boot-autoconfigure | casehub-ledger-spring, casehub-platform-spring | spring-boot-test |

### Tests

- **Unit (audit-core):** Mock `LedgerEntryWriter` and `CurrentPrincipal`.
  Verify: correct entry types, tenant ID propagation, token usage extraction,
  compliance supplement fields, causal chaining (`causedByEntryId`), graceful
  degradation when ledger is null.
- **Integration (audit):** `@QuarkusTest` with ledger on classpath. Fire a
  real `ChatModel` call (use MockWebServer or in-memory model stub). Verify
  ledger entries are persisted.
- **Integration (audit-spring):** `@SpringBootTest` equivalent.

---

## 2. tenancy

### Purpose

Decorates any `ChatMemoryStore`, `EmbeddingStore`, or `ContentRetriever`
with tenant isolation. The developer's chosen implementation is preserved
inside the decorator.

### tenancy-core (pure Java, no framework)

**Package:** `io.casehub.langchain4j.tenancy`

#### TenantChatMemoryStore

```java
public class TenantChatMemoryStore implements ChatMemoryStore {

    private final ChatMemoryStore delegate;
    private final CurrentPrincipal currentPrincipal; // nullable

    @Override
    public List<ChatMessage> getMessages(Object memoryId) {
        return delegate.getMessages(tenantScopedId(memoryId));
    }

    @Override
    public void updateMessages(Object memoryId, List<ChatMessage> messages) {
        delegate.updateMessages(tenantScopedId(memoryId), messages);
    }

    @Override
    public void deleteMessages(Object memoryId) {
        delegate.deleteMessages(tenantScopedId(memoryId));
    }

    private Object tenantScopedId(Object memoryId) {
        if (currentPrincipal == null) return memoryId; // passthrough
        String tenantId = currentPrincipal.tenancyId();
        if (tenantId == null) return memoryId;         // passthrough
        return tenantId + "::" + memoryId;
    }
}
```

#### TenantEmbeddingStore

```java
public class TenantEmbeddingStore<T> implements EmbeddingStore<T> {

    private final EmbeddingStore<T> delegate;
    private final CurrentPrincipal currentPrincipal;

    private static final String TENANT_KEY = "casehub_tenant_id";

    @Override
    public String add(Embedding embedding, T embedded) {
        // If embedded is TextSegment, inject tenant ID into metadata
        if (embedded instanceof TextSegment segment && currentPrincipal != null) {
            Metadata enriched = segment.metadata().copy();
            enriched.put(TENANT_KEY, currentPrincipal.tenancyId());
            return delegate.add(embedding, (T) TextSegment.from(segment.text(), enriched));
        }
        return delegate.add(embedding, embedded);
    }

    @Override
    public EmbeddingSearchResult<T> search(EmbeddingSearchRequest request) {
        if (currentPrincipal == null) return delegate.search(request);
        // Add tenant filter to existing filters
        Filter tenantFilter = metadataKey(TENANT_KEY).isEqualTo(currentPrincipal.tenancyId());
        Filter combined = request.filter() != null
            ? and(request.filter(), tenantFilter)
            : tenantFilter;
        EmbeddingSearchRequest scoped = EmbeddingSearchRequest.builder()
            .queryEmbedding(request.queryEmbedding())
            .maxResults(request.maxResults())
            .minScore(request.minScore())
            .filter(combined)
            .build();
        return delegate.search(scoped);
    }

    // add(Embedding), add(String, Embedding), addAll() — delegate with
    // tenant metadata injection where TextSegment is available
}
```

#### TenantContentRetriever

```java
public class TenantContentRetriever implements ContentRetriever {

    private final ContentRetriever delegate;
    private final CurrentPrincipal currentPrincipal;

    @Override
    public List<Content> retrieve(Query query) {
        List<Content> results = delegate.retrieve(query);
        if (currentPrincipal == null) return results;
        // Filter results to current tenant only
        String tenantId = currentPrincipal.tenancyId();
        return results.stream()
            .filter(c -> tenantId.equals(
                c.textSegment().metadata().getString("casehub_tenant_id")))
            .toList();
    }
}
```

**Design note:** The `ContentRetriever` decorator applies post-retrieval
filtering. This is a safety net — the primary isolation mechanism is
the `TenantEmbeddingStore` filter on ingestion and search. The
`TenantContentRetriever` catches any retriever implementation that
does not use the tenant-filtered `EmbeddingStore` (e.g., SQL-based
retrievers, web search retrievers).

### tenancy (Quarkus CDI)

**Package:** `io.casehub.langchain4j.tenancy.quarkus`

CDI wiring uses a `@Produces` method that injects the existing store
beans and wraps them:

```java
@ApplicationScoped
public class TenancyBeanProducer {

    @Inject @Any Instance<CurrentPrincipal> principalInstance;

    @Produces @Alternative @Priority(100) @ApplicationScoped
    public ChatMemoryStore tenantChatMemoryStore(
            @Any Instance<ChatMemoryStore> stores) {
        ChatMemoryStore delegate = selectNonTenant(stores);
        if (delegate == null) return null; // no store configured
        return new TenantChatMemoryStore(delegate, principalOrNull());
    }

    @Produces @Alternative @Priority(100) @ApplicationScoped
    public EmbeddingStore<TextSegment> tenantEmbeddingStore(
            @Any Instance<EmbeddingStore<TextSegment>> stores) {
        EmbeddingStore<TextSegment> delegate = selectNonTenant(stores);
        if (delegate == null) return null;
        return new TenantEmbeddingStore<>(delegate, principalOrNull());
    }

    @Produces @Alternative @Priority(100) @ApplicationScoped
    public ContentRetriever tenantContentRetriever(
            @Any Instance<ContentRetriever> retrievers) {
        ContentRetriever delegate = selectNonTenant(retrievers);
        if (delegate == null) return null;
        return new TenantContentRetriever(delegate, principalOrNull());
    }

    // selectNonTenant: filter out the Tenant* classes to avoid circularity
    // principalOrNull: check isResolvable()
}
```

**CDI circularity prevention:** The producer filters out beans that are
instances of the `Tenant*` decorators themselves, matching the pattern
used by `ChatModelAgentProvider` in the platform agent-langchain4j module
(see `2026-06-26-agent-langchain4j-interop-design.md` §CDI circular
dependency prevention).

### tenancy-spring (Spring auto-configuration)

```java
@AutoConfiguration
@ConditionalOnClass(ChatMemoryStore.class)
public class CasehubTenancyAutoConfiguration {

    @Bean
    @ConditionalOnBean(ChatMemoryStore.class)
    @Primary
    public ChatMemoryStore tenantChatMemoryStore(
            ChatMemoryStore delegate,
            @Autowired(required = false) CurrentPrincipal principal) {
        return new TenantChatMemoryStore(delegate, principal);
    }

    // Equivalent for EmbeddingStore, ContentRetriever
}
```

Spring's `@Primary` ensures the tenant wrapper wins injection. The
original bean remains accessible via `@Qualifier` if needed.

### Dependencies

| Module | Compile | Optional | Test |
|---|---|---|---|
| tenancy-core | langchain4j-core, casehub-platform-api | — | junit5, assertj |
| tenancy | tenancy-core, quarkus-arc | — | quarkus-junit5 |
| tenancy-spring | tenancy-core, spring-boot-autoconfigure | — | spring-boot-test |

### Tests

- **Unit (tenancy-core):** Mock delegate stores and `CurrentPrincipal`.
  Verify: tenant-scoped memory IDs, metadata injection on embedding add,
  filter injection on embedding search, post-retrieval filtering on
  content retriever, passthrough when principal is null.
- **Integration (tenancy):** `@QuarkusTest` with `InMemoryChatMemoryStore`.
  Two tenants store messages with the same `memoryId`. Verify no
  cross-tenant leakage.
- **Edge cases:** null tenant ID, null principal, stores that don't support
  metadata filtering (should fail clearly, not silently).

---

## 3. governance

### Purpose

Injects causal lineage, oversight gates, and trust-weighted routing into
any langchain4j agent pattern. Includes the existing ChatModel ↔ AgentProvider
bridge moved from platform.

### governance-core (pure Java, no framework)

**Package:** `io.casehub.langchain4j.governance`

#### CasehubGovernanceListener

```java
public class CasehubGovernanceListener implements AgentListener {

    private final LedgerEntryWriter ledgerWriter;     // nullable
    private final OversightGateService oversightGate; // nullable (engine dep)
    private final TrustSignalProvider trustProvider;   // nullable (engine dep)
    private final CurrentPrincipal currentPrincipal;   // nullable

    @Override
    public void beforeAgentInvocation(AgentRequest request) {
        String agentName = request.agentName();
        Object memoryId = request.agenticScope().memoryId();

        // 1. Lineage — create STARTED entry
        String entryId = writeLedgerEntry("AGENT_INVOCATION_STARTED", agentName, memoryId);
        request.agenticScope().put("casehub.lineage.entryId", entryId);

        // 2. Oversight gate check (if engine present)
        if (oversightGate != null) {
            OversightDecision decision = oversightGate.evaluate(agentName, request);
            if (decision.requiresApproval()) {
                // Block until human approves or timeout
                // Creates a WorkItem in the engine
                // Throws GovernanceRejectedException if denied or timeout
                oversightGate.awaitApproval(decision, Duration.ofMinutes(30));
            }
        }

        // 3. Trust check (informational — routing happens in AgenticScope)
        if (trustProvider != null) {
            double trustScore = trustProvider.score(agentName);
            request.agenticScope().put("casehub.trust.score", trustScore);
        }
    }

    @Override
    public void afterAgentInvocation(AgentResponse response) {
        String agentName = response.agentName();
        String parentEntryId = (String) response.agenticScope()
            .get("casehub.lineage.entryId");

        // 1. Complete lineage entry
        writeLedgerEntry("AGENT_INVOCATION_COMPLETED", agentName,
            response.agenticScope().memoryId(), parentEntryId);

        // 2. Record outcome for trust scoring
        if (trustProvider != null) {
            trustProvider.recordOutcome(agentName,
                TrustOutcome.success(response.output()));
        }
    }

    @Override
    public void onAgentInvocationError(AgentInvocationError error) {
        writeLedgerEntry("AGENT_INVOCATION_FAILED", error.agentName(),
            error.agenticScope().memoryId());

        if (trustProvider != null) {
            trustProvider.recordOutcome(error.agentName(),
                TrustOutcome.failure(error.cause()));
        }
    }

    @Override
    public void afterAgentToolExecution(AfterAgentToolExecution toolExec) {
        String parentEntryId = (String) toolExec.agenticScope()
            .get("casehub.lineage.entryId");
        writeLedgerEntry("AGENT_TOOL_EXECUTED",
            toolExec.toolName(), toolExec.agenticScope().memoryId(),
            parentEntryId);
    }

    @Override
    public boolean inheritedBySubagents() {
        return true;
    }
}
```

#### GovernanceRejectedException

```java
public class GovernanceRejectedException extends RuntimeException {
    private final String agentName;
    private final String reason; // "denied", "timeout", "trust_below_threshold"
}
```

#### Existing Bridge Classes (moved from platform)

These classes move verbatim from `platform/agent-langchain4j-core`:

| Class | Purpose |
|---|---|
| `ChatModelAgentProvider` | Wraps any `ChatModel` as `AgentProvider` |
| `ChatModelAgentSession` | Multi-turn session backed by ChatModel + ChatMemory |
| `AgentProviderChatModel` | Wraps any `AgentProvider` as `ChatModel` |
| `AgentSessionChatModel` | Wraps `AgentSession` as `ChatModel` |
| `AgentEventBridge` | Maps between AgentEvent and LC4j event types |
| `AgentLangchain4jProperties` | Configuration (closeTimeout, sessionMemoryWindowSize) |

Package changes from `io.casehub.platform.agent.langchain4j` to
`io.casehub.langchain4j.agents` (or keep old package with deprecation
forwarding — implementer's call on migration pain).

### governance (Quarkus CDI)

```java
@ApplicationScoped
public class GovernanceBeanProducer {

    @Inject @Any Instance<LedgerEntryWriter> ledgerInstance;
    @Inject @Any Instance<OversightGateService> oversightInstance;
    @Inject @Any Instance<TrustSignalProvider> trustInstance;
    @Inject @Any Instance<CurrentPrincipal> principalInstance;

    @Produces @ApplicationScoped
    public CasehubGovernanceListener governanceListener() {
        return new CasehubGovernanceListener(
            orNull(ledgerInstance),
            orNull(oversightInstance),
            orNull(trustInstance),
            orNull(principalInstance)
        );
    }
}
```

CDI wiring for the bridge classes follows the existing pattern from
`platform/agent-langchain4j/` — `Langchain4jBeans`, `AgentLangchain4jConfig`.
Move these as-is, updating package references.

### governance-spring

Same pattern: `@AutoConfiguration` with `@Autowired(required = false)` for
all optional dependencies.

### Dependencies

| Module | Compile | Optional | Test |
|---|---|---|---|
| governance-core | langchain4j-core, langchain4j-agentic, casehub-platform-agent-api, casehub-ledger-api | casehub-engine-api (OversightGate, TrustSignalProvider) | junit5, assertj, mockito, awaitility |
| governance | governance-core, quarkus-arc | casehub-engine, casehub-ledger | quarkus-junit5 |
| governance-spring | governance-core, spring-boot-autoconfigure | — | spring-boot-test |

### Tests

- **Unit (governance-core):** Mock all dependencies. Verify: lineage
  entries with correct `causedByEntryId` chains, oversight gate blocks
  until approval, trust score recording on success/failure,
  `inheritedBySubagents()` returns true, graceful skip when optional
  deps are null.
- **Integration (governance):** `@QuarkusTest` with a simple
  `@SupervisorAgent` that invokes two sub-agents. Verify: three ledger
  entries (supervisor + 2 sub-agents) with correct causal chain.
- **Oversight gate test:** Mock `OversightGateService.evaluate()` to
  require approval. Verify agent invocation blocks. Complete the
  `WorkItem`. Verify agent invocation proceeds.

---

## 4. hybrid-search

### Purpose

Implements `ContentRetriever` with neocortex's hybrid retrieval pipeline.
This is the only Phase 1 module that is an implementation (not a decorator) —
justified by langchain4j's acknowledged gap ([#4087](https://github.com/langchain4j/langchain4j/issues/4087)).

### hybrid-search-core (pure Java, no framework)

**Package:** `io.casehub.langchain4j.hybridsearch`

#### CasehubHybridContentRetriever

```java
public class CasehubHybridContentRetriever implements ContentRetriever {

    private final CaseContextRetriever delegate; // neocortex SPI

    @Override
    public List<Content> retrieve(Query query) {
        // Map LC4j Query → neocortex retrieval request
        RetrievalRequest request = RetrievalRequest.builder()
            .queryText(query.text())
            .maxResults(maxResults)
            .build();

        // Execute hybrid retrieval (SPLADE + dense + RRF + optional reranking)
        List<RetrievedContext> results = delegate.retrieve(request);

        // Map neocortex results → LC4j Content
        return results.stream()
            .map(rc -> Content.from(TextSegment.from(
                rc.text(),
                Metadata.from(rc.metadata()))))
            .toList();
    }
}
```

#### Configuration

```java
public class HybridSearchConfig {
    private int maxResults = 10;
    private boolean rerankingEnabled = true;
    private int rerankingTopN = 50;
    // Getters/setters or builder
}
```

### hybrid-search (Quarkus CDI)

```java
@ApplicationScoped
public class HybridSearchBeanProducer {

    @Inject CaseContextRetriever caseContextRetriever;

    @Produces @ApplicationScoped
    @Named("casehubHybridRetriever")
    public ContentRetriever hybridContentRetriever() {
        return new CasehubHybridContentRetriever(caseContextRetriever);
    }
}
```

**Important:** This is NOT `@Alternative` or `@DefaultBean`. It is
`@Named` so it coexists with any existing `ContentRetriever` the developer
has. The developer explicitly references it:

```java
@Inject @Named("casehubHybridRetriever") ContentRetriever retriever;
```

Or uses it in `@RegisterAiService`:

```java
@RegisterAiService(retriever = CasehubHybridContentRetriever.class)
```

This is a deliberate choice — hybrid search is a replacement, not a
decorator, so it should not silently override the developer's existing
retriever. They opt in explicitly.

### hybrid-search-spring

Same pattern with `@Bean @Named`.

### Dependencies

| Module | Compile | Test |
|---|---|---|
| hybrid-search-core | langchain4j-core, casehub-neocortex-rag-api | junit5, assertj |
| hybrid-search | hybrid-search-core, casehub-neocortex-rag, quarkus-arc | quarkus-junit5, casehub-neocortex-rag-testing |
| hybrid-search-spring | hybrid-search-core, casehub-neocortex-rag-spring, spring-boot-autoconfigure | spring-boot-test |

### Tests

- **Unit (hybrid-search-core):** Mock `CaseContextRetriever`. Verify:
  Query mapping, Content mapping, metadata preservation, max results
  configuration.
- **Integration (hybrid-search):** `@QuarkusTest` with in-memory neocortex
  stubs (from `casehub-neocortex-rag-testing`). Ingest documents, query,
  verify hybrid results returned.

---

## Cross-Cutting Concerns

### Version Compatibility

Pin to `langchain4j-core` version declared in `casehub-parent` BOM
(`version.dev.langchain4j`). Currently `1.14.1`. Test against the pinned
version only — do not support arbitrary LC4j versions.

When upstream LC4j releases a new version:
1. Check for SPI changes in `ChatModelListener`, `AgentListener`,
   `ChatMemoryStore`, `EmbeddingStore`, `ContentRetriever`
2. Update pin in parent BOM
3. Fix any breaking changes in this repo
4. Publish

### Maven Coordinates

- GroupId: `io.casehub.langchain4j`
- ArtifactId pattern: `casehub-langchain4j-<module>`
- Version: `${casehub.version}` (currently `0.2-SNAPSHOT`)

Examples:
- `io.casehub.langchain4j:casehub-langchain4j-audit-core`
- `io.casehub.langchain4j:casehub-langchain4j-audit`
- `io.casehub.langchain4j:casehub-langchain4j-audit-spring`

### BOM

`casehub-langchain4j-bom` imports all modules. Consumers can depend on the
BOM and pick modules:

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>io.casehub.langchain4j</groupId>
            <artifactId>casehub-langchain4j-bom</artifactId>
            <version>${casehub.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <dependency>
        <groupId>io.casehub.langchain4j</groupId>
        <artifactId>casehub-langchain4j-audit</artifactId>
    </dependency>
</dependencies>
```

### Examples

One example per module in `examples/`:

- `examples/audit-example/` — Quarkus app with Ollama ChatModel + audit.
  Shows: add dependency, run, check ledger entries.
- `examples/tenancy-example/` — Quarkus app with two tenants sharing one
  `InMemoryChatMemoryStore`. Shows: same memoryId, different tenants,
  no leakage.
- `examples/governance-example/` — Quarkus app with `@SupervisorAgent` +
  two sub-agents. Shows: lineage chain in ledger, oversight gate blocks
  until approval.
- `examples/hybrid-search-example/` — Quarkus app with corpus ingestion +
  hybrid retrieval. Shows: query returns SPLADE-expanded results.
