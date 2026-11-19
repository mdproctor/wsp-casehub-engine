# Decisions — #1331 Auto-Discovery

## D1: Registration mechanism — enhance existing vs new class

**Choice:** Enhance `CaseYamlRegistrar` in-place
**Alternatives:**
- New standalone `CaseHubAutoDiscoverer` class — clean separation but duplicates classpath scanning
- Shared `CaseYamlLoader` utility — DRY but bigger refactor scope, couples YamlCaseHub internals
**Rationale:** CaseYamlRegistrar already scans `casehub/` and `cases/` at priority 200 via the platform bootstrap SPI. It's the natural place to add full pipeline loading. No new classes, no scanning duplication.
**Trade-offs:** register() receives an InputStream but overlay support requires classpath lookup — minor complexity addition.
**Sources:** CaseYamlRegistrar.java:30, DefaultCaseDefinitionRegistry.java:103, YamlCaseHub.java:68
**Exploration:** quick
**Status:** captured

## D2: No actual CaseHub CDI beans needed

**Choice:** Register fully-wired CaseDefinition objects via CaseDefinitionRegistry (no CDI CaseHub beans)
**Alternatives:**
- Quarkus build extension for synthetic YamlCaseHub beans — first-class CDI but heavy infrastructure, new module required
**Rationale:** The runtime operates on CaseDefinition objects via CaseDefinitionRegistry. REST/MCP endpoints use the registry, not injected CaseHub beans. For zero-Java apps, the programmatic startCase() API is irrelevant — cases are started via endpoints.
**Trade-offs:** No injectable CaseHub bean for auto-discovered cases. If a consumer later needs programmatic access, they write a one-line YamlCaseHub subclass (the existing pattern).
**Sources:** DefaultCaseDefinitionRegistry.java:111 (iterates Instance<CaseHub> for definitions only), CaseHub.java:28 (startCase delegates to CaseHubRuntime)
**Exploration:** quick
**Status:** captured

## D3: Overlay support via convention path

**Choice:** Support `-overrides` convention overlays for auto-discovered YAML
**Alternatives:**
- Base YAML only — simpler but inconsistent with YamlCaseHub behavior
**Rationale:** Mirrors YamlCaseHub's overlay composition. Consumers can customize auto-discovered cases without Java code — consistent zero-Java experience.
**Trade-offs:** Slightly more complex register() method. Need to handle the stream-to-JsonNode conversion for merging.
**Sources:** YamlCaseHub.java:109 (resolveOverlay), YamlCaseHub.java:143 (deriveConventionPath)
**Exploration:** quick
**Status:** captured
