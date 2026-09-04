## D1: Document location

**Choice:** Standalone `docs/guides/three-pathways.md` with consumer-guide link
**Alternatives:**
- Consumer guide section — keeps everything in one place but bloats the reference
- Both (standalone + summary in consumer guide) — more maintenance surface
**Rationale:** Consumer guide is dense reference; pathways content is tutorial/onboarding for a different audience. Standalone gives room for code examples. Future platform-wide docs revamp may relocate it.
**Trade-offs:** Two files to maintain instead of one; newcomers might not find it without the consumer-guide link
**Sources:** docs/guides/consumer-guide.md, issue #986
**Exploration:** quick
**Status:** captured

## D2: Document structure

**Choice:** Five sections: opening, decision table, side-by-side walkthrough (choreography-onboarding), mixing pathways (hybrid), DSL quick reference
**Alternatives:**
- Tutorial-first (Getting Started walkthrough then reference) — buries the decision guidance
- Reference-only (field tables, no narrative) — doesn't help newcomers choose
**Rationale:** Decision table upfront answers "which pathway?" before the reader commits. Side-by-side uses the same scenario across all three so differences are visible. Quick reference covers the YAML schema systematically.
**Trade-offs:** Code-heavy doc (~400-600 lines); needs updating when schema changes
**Sources:** examples/yaml/choreography-onboarding.yaml, examples/choreography-annotated/, examples/choreography-dsl/
**Exploration:** quick
**Status:** captured
