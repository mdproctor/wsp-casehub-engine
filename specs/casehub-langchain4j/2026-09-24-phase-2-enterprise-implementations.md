# Phase 2: Enterprise Implementations — Spec

**Date:** 2026-09-24
**Prerequisite:** Phase 1 modules shipped and stable
**Strategy:** [STRATEGY.md](STRATEGY.md)
**Decisions:** [decisions.md](decisions.md)

---

## Scope

Phase 2 adds SPI implementations that go beyond decorating existing LC4j
components. Each module provides capabilities langchain4j does not offer —
not competing implementations of things langchain4j already does well.

Every module in this phase requires a `## Why this module exists` section
in its README documenting which of the three justifications applies:
gap acknowledged, different category, or customer requested (see
STRATEGY.md §Expansion Policy).

---

## 1. CBR Memory — Case-Based Reasoning for Chat History

### Why This Module Exists

**Justification: Different category.**

LangChain4j's `ChatMemoryStore` is a persistence contract — store messages,
retrieve messages, delete messages. Community implementations (Redis,
PostgreSQL, Infinispan) fulfil this well.

CaseHub's CBR (case-based reasoning) memory is a different category. It
does not replace `ChatMemoryStore` — it sits alongside it, providing:

- **Outcome tracking** — records what happened after an agent conversation
  (did the recommendation succeed? was the diagnosis correct?)
- **Similarity-based retrieval** — finds past cases similar to the current
  situation using feature vectors, not just conversation ID
- **Trust-weighted retention** — agents with poor outcome history have
  their case records deprioritised or purged
- **Adaptation** — past solutions are adapted to new contexts, not
  replayed verbatim

A developer using `InMemoryChatMemoryStore` keeps it. CBR memory is
a separate concern: "what worked before in similar situations?"

### Module Structure

| Module | Purpose |
|---|---|
| `cbr-memory-core` | Pure Java — `CbrMemoryContentRetriever implements ContentRetriever`, `CbrChatMemoryAdvisor` |
| `cbr-memory` | Quarkus CDI wiring |
| `cbr-memory-spring` | Spring auto-configuration |

### Key Classes

#### CbrMemoryContentRetriever

```java
public class CbrMemoryContentRetriever implements ContentRetriever {

    private final CbrRecordStore cbrStore; // neocortex memory-api

    @Override
    public List<Content> retrieve(Query query) {
        // Build CbrQuery from LC4j Query
        CbrQuery cbrQuery = CbrQuery.builder()
            .queryText(query.text())
            .retrievalMode(RetrievalMode.SEMANTIC)
            .build();

        // Retrieve similar past cases
        List<CbrMatch> matches = cbrStore.retrieveSimilar(cbrQuery);

        // Map to LC4j Content
        return matches.stream()
            .map(m -> Content.from(TextSegment.from(
                m.record().text(),
                Metadata.from(Map.of(
                    "cbr_similarity", String.valueOf(m.score()),
                    "cbr_outcome", m.record().outcome().name(),
                    "cbr_record_id", m.record().id()
                )))))
            .toList();
    }
}
```

#### CbrChatMemoryAdvisor

An advisor (not a store) that enriches the agent's context with
similar past cases before each LLM call:

```java
public class CbrChatMemoryAdvisor {

    private final CbrRecordStore cbrStore;
    private final int maxPastCases;

    public List<ChatMessage> enrich(List<ChatMessage> history, String currentQuery) {
        List<CbrMatch> similar = cbrStore.retrieveSimilar(
            CbrQuery.builder().queryText(currentQuery).limit(maxPastCases).build());

        if (similar.isEmpty()) return history;

        // Prepend a SystemMessage with similar past cases
        String caseContext = similar.stream()
            .map(m -> "Past case (similarity: " + m.score() + "): " + m.record().text()
                + "\nOutcome: " + m.record().outcome())
            .collect(Collectors.joining("\n\n"));

        List<ChatMessage> enriched = new ArrayList<>();
        enriched.add(SystemMessage.from("Relevant past cases:\n" + caseContext));
        enriched.addAll(history);
        return enriched;
    }
}
```

### Dependencies

Compile: `langchain4j-core`, `casehub-neocortex-memory-api`
Optional: `casehub-neocortex-memory` (Qdrant backend), `casehub-neocortex-memory-cbr-jpa` (JPA backend)
Test: `casehub-neocortex-memory-cbr-inmem`

### Implementation Notes

- The CBR store is NOT a `ChatMemoryStore` implementation. It does not
  replace the developer's existing memory store. It provides a
  `ContentRetriever` and an advisor that enriches context.
- Outcome recording: the governance module's `afterAgentInvocation()`
  hook can feed outcomes into CBR automatically when both modules
  are on the classpath. This is optional — CBR works without governance.

---

## 2. SPLADE Embedding — Learned Sparse Vectors

### Why This Module Exists

**Justification: Gap acknowledged (no JVM implementation exists).**

LangChain4j provides `OnnxEmbeddingModel` for dense embeddings. No
langchain4j module or community implementation provides SPLADE sparse
embeddings on the JVM. Neocortex's `SparseEmbedder` is the only JVM
implementation. SPLADE adds 26–31% NDCG improvement when combined with
dense retrieval (see `neocortex/docs/architecture-justification.md` §1-2).

### Module Structure

| Module | Purpose |
|---|---|
| `splade-embedding-core` | Pure Java — `SpladeEmbeddingModel`, `SparseEmbeddingResult` |
| `splade-embedding` | Quarkus CDI wiring |
| `splade-embedding-spring` | Spring auto-configuration |

### Key Classes

#### SpladeEmbeddingModel

This is NOT an `EmbeddingModel` (LC4j's interface outputs dense `float[]`).
SPLADE produces sparse vectors (`Map<Integer, Float>`). LC4j has no sparse
embedding SPI — this is a standalone class that complements `EmbeddingModel`.

```java
public class SpladeEmbeddingModel {

    private final SparseEmbedder sparseEmbedder; // neocortex inference-splade

    public SparseEmbeddingResult embed(String text) {
        Map<Integer, Float> sparse = sparseEmbedder.embed(text);
        return new SparseEmbeddingResult(sparse);
    }

    public List<SparseEmbeddingResult> embedBatch(List<String> texts) {
        return sparseEmbedder.embedBatch(texts).stream()
            .map(SparseEmbeddingResult::new)
            .toList();
    }
}
```

#### SparseEmbeddingResult

```java
public record SparseEmbeddingResult(Map<Integer, Float> weights) {
    public int nonZeroCount() { return weights.size(); }
}
```

### Dependencies

Compile: `casehub-neocortex-inference-api`, `casehub-neocortex-inference-splade`
Test: `casehub-neocortex-inference-inmem` (no JNI in tests)

### Implementation Notes

- No LC4j SPI to implement — SPLADE is additive. Developers use it
  alongside their existing `EmbeddingModel` for hybrid retrieval.
- The hybrid-search module from Phase 1 uses SPLADE internally via
  neocortex. This module exposes SPLADE directly for developers who
  want standalone sparse embeddings without the full hybrid pipeline.
- ONNX model artifacts (model.onnx + tokenizer.json) are versioned
  dependencies, not bundled in the JAR.

---

## 3. Corpus Ingestion — Enterprise Document Lifecycle

### Why This Module Exists

**Justification: Different category.**

LangChain4j provides `DocumentLoader`, `DocumentSplitter`, and
`DocumentTransformer` for ingesting documents into embedding stores.
These are single-shot operations — load, split, embed, store.

CaseHub's corpus ingestion is a managed lifecycle:

- **Versioning** — track which version of a document is embedded.
  When the source document changes, re-ingest only the delta.
- **Provenance** — every chunk records its source document, page,
  section, and ingestion timestamp. Audit-trail grade.
- **Change tracking** — `ChangeSource` SPI detects when source
  documents change (filesystem watcher, S3 events, webhook).
  Automatic re-ingestion without developer code.
- **Tenant-scoped corpora** — each tenant's documents are isolated
  in separate corpora with independent versioning.

This is not a competing `DocumentLoader` — it's a document lifecycle
manager that uses LC4j's splitters internally.

### Module Structure

| Module | Purpose |
|---|---|
| `corpus-core` | Pure Java — `ManagedCorpus`, `CorpusDocument`, `IngestionPlan` |
| `corpus` | Quarkus CDI wiring |
| `corpus-spring` | Spring auto-configuration |

### Key Classes

#### ManagedCorpus

```java
public class ManagedCorpus {

    private final CorpusIngestionService ingestionService; // neocortex corpus-api
    private final EmbeddingStore<TextSegment> embeddingStore; // developer's store
    private final EmbeddingModel embeddingModel;              // developer's model
    private final DocumentSplitter splitter;                  // developer's splitter (or default)

    public IngestionResult ingest(Document document, CorpusRef corpusRef) {
        // 1. Check version — skip if already ingested at this version
        // 2. Split using LC4j splitter
        // 3. Embed using LC4j embedding model
        // 4. Store with provenance metadata (source, version, page, tenant)
        // 5. Record in corpus ledger
    }

    public IngestionResult reingest(CorpusRef corpusRef) {
        // 1. Find changed documents via ChangeSource
        // 2. Remove stale embeddings for changed docs
        // 3. Re-ingest changed docs only
    }

    public void purge(CorpusRef corpusRef) {
        // Remove all embeddings for this corpus
        // Record purge in corpus ledger
    }
}
```

### Dependencies

Compile: `langchain4j-core`, `casehub-neocortex-corpus-api`, `casehub-platform-api`
Optional: `casehub-neocortex-rag-core` (Qdrant-backed ingestion), `casehub-ledger-api` (provenance audit)
Test: in-memory stubs

### Implementation Notes

- Uses the developer's `EmbeddingModel` and `DocumentSplitter` — does
  not replace them.
- Uses the developer's `EmbeddingStore` — adds lifecycle management on
  top.
- The tenancy decorator from Phase 1 composes naturally: tenant-scoped
  embedding store + versioned corpus = tenant-isolated document lifecycle.

---

## Phase 2 Delivery Order

| Module | Dependencies on Phase 1 | Priority |
|---|---|---|
| CBR memory | Benefits from governance (outcome recording) but works without it | High — unique value |
| SPLADE embedding | Independent | Medium — niche use case |
| Corpus ingestion | Benefits from tenancy decorator | Medium — requires ChangeSource SPI |

Each module ships independently. No big-bang release.

---

## Open Questions (for implementer)

1. Should CBR memory auto-register as a `ContentRetriever` alongside the
   developer's existing retriever (via `@Named`), or require explicit
   wiring? Recommend `@Named` — same pattern as hybrid-search.
2. Should corpus ingestion depend on casehub-ledger for provenance, or
   provide its own lightweight provenance record? Recommend ledger
   dependency with graceful skip.
3. SPLADE model distribution — should the ONNX model be a Maven artifact
   or downloaded at first use? Recommend Maven artifact for
   reproducibility (matches neocortex pattern).
