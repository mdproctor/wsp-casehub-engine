# Engine HANDOFF — Slot 214

## Open Branches

### issue-1245-react-platform-migration
- **Commits this session:**
  - `d8a745b75` feat(#1252): remove quarkiverse-langchain4j from flow module — strips @RegisterAiService annotations, adds StubAgents @ApplicationScoped implementations, removes quarkus-langchain4j-ollama dependency
  - `d044ccab2` feat(#644): bump LangChain4j 1.14.1 → 1.19.0 — also fixes VocabularyRegistry VocabularyMetadata → VocabularyInfo rename in test fixture
- **Status:** pushed, no PR yet. 5 commits total (3 prior + 2 this session)
- **Issues covered:** engine#1245, engine#1252, platform#644
- **Pre-existing test failures:** 3 errors in runtime module (NoSuchMethodError on WorkerAgentInvoker constructor — SNAPSHOT binary compat, not caused by this branch)
