## D1: Existing example handling

**Choice:** Move + extend — relocate relevant schema examples to `examples/yaml/`, update/extend them, write remaining from scratch
**Alternatives:**
- Start fresh — write all examples independently, risk duplication with schema examples
- Symlink — reference schema examples from examples/yaml/, messy dual-location
**Rationale:** Reuses proven YAML (document-processing.yaml is already a solid choreography example). Avoids maintaining two overlapping sets. Schema module keeps test fixtures that aren't full examples.
**Trade-offs:** Schema validation test may need updating if fixture paths change. But the test can reference examples/yaml/ just as easily.
**Sources:** `schema/src/main/resources/examples/` (6 existing files), `examples/` (9 Java examples)
**Exploration:** quick
**Status:** captured

## D2: Scope — first batch

**Choice:** Core 8 examples for this branch: Choreography, Sequential, GOAP, LLM decomposition, HumanTask, SubCase, A2A, MCP
**Alternatives:**
- All 23 — thorough but too large for one branch
- Core 5 (orchestration only) — narrower, misses worker integration patterns (A2A, MCP, HumanTask)
**Rationale:** Covers the primary authoring patterns — both orchestration models and integration targets. Proves YAML parity for the most common use cases. Remaining 15 examples are follow-on work.
**Trade-offs:** 15 examples deferred. But the 8 selected cover the highest-value execution models.
**Depends on:** D1 (existing examples)
**Sources:** engine#978 Part 2 table (23 examples), engine#984 issue body
**Exploration:** quick
**Status:** captured

## D3: Domain flavor

**Choice:** Different realistic domains per example — each uses a domain that naturally fits its execution model
**Alternatives:**
- Shared domain (all document processing variations) — easier side-by-side comparison but less realistic
- Mixed — some shared, some unique
**Rationale:** More realistic examples are better for tutorials and blocks-ui fixtures. Each domain shows a natural use case for its execution model rather than forcing a contrived fit.
**Trade-offs:** Harder to compare execution models side-by-side. But the Three Pathways Guide (#986) is the right place for comparison, not the examples themselves.
**Depends on:** D2 (scope)
**Sources:** engine#978 Part 3 (blocks-ui fixtures)
**Exploration:** quick
**Status:** captured
