## D1: Model ID sourcing strategy

**Choice:** Store modelId on Agent, look up at retain time via CaseDefinition
**Alternatives:**
- Thread via protocolMetadata → EventLog → retain — captures runtime model but adds EventLog dependency and N queries per retain, overkill for Scale S
**Rationale:** Observer already has CaseDefinition. YAML modelName is the stable identity for CBR correlation. Simple 4-file change.
**Trade-offs:** Only works for AgentWorkerFunction workers (not A2A, MCP, ReAct). Reflects declared model, not dynamically selected.
**Sources:** CbrCaseRetainObserver.java:335-344, Agent.java, AgentConverter.java, ModelPreferenceSignalProvider.java
**Exploration:** quick
**Status:** captured
