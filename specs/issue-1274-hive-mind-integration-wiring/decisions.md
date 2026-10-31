## D1: Dependency model for eidos/qhorus integration

**Choice:** Direct injection with optional beans (`Instance<T>`)
**Alternatives:**
- Engine-defined SPI adapters — adds a wrapping layer that mirrors eidos/qhorus APIs with no design benefit
- New integration module — premature for a few injections and method calls
**Rationale:** engine-api already depends on eidos-api and qhorus-api; the types are on the classpath. Pre-release platform — no backward-compat concern. eidos-routing precedent exists only because routing involves complex domain translation; swarm integration is simpler.
**Trade-offs:** Engine gains a runtime dependency on eidos/qhorus API types (already present via engine-api). If eidos/qhorus APIs change, engine code changes too — acceptable for a pre-release platform that bumps versions together.
**Sources:** engine/api/pom.xml (existing eidos-api/qhorus-api deps), eidos-routing module (precedent), AgentGraphQuery.java, RuntimeCollaborationQuery.java, MessageDispatcher.java
**Exploration:** quick
**Status:** captured
