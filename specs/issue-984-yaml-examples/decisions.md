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

## D4: Three-pathway pairing

**Choice:** Every example exists in all three forms: YAML + Java DSL (`CaseHub` subclass) + Java Annotations (`@Case` interface). Where annotations can't express a feature natively, `@Customize` drops into the DSL — showing the blend honestly.
**Alternatives:**
- YAML + best-fit Java — one Java file using whichever pathway fits. Less work but doesn't show all three side-by-side.
- YAML + DSL only — skip annotations. Misses the annotation pathway which is the primary Java entry point.
- YAML first, Java later — faster to ship but parity isn't proven until all three exist.
**Rationale:** The "Rosetta Stone" approach — same case definition in all three forms. Shows users exactly where each pathway is self-sufficient and where it needs `@Customize` escape hatches. Directly proves YAML/Java parity. Existing annotated examples (3 already exist) provide a head start.
**Trade-offs:** ~24 files for 8 examples (8 YAML + 8 DSL + 8 annotations). Some annotation versions will need `@Customize` for features annotations can't express natively (A2A, MCP, SubCase mappings). That's honest documentation, not a weakness.
**Depends on:** D2 (scope), D3 (domain)
**Sources:** `examples/` (9 existing annotated Java examples), `casehub-engine-annotations` module, `api/model/converter/YamlCaseDefinitionConverter.java`
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

## D4: Three-pathway pairing

**Choice:** Every example exists in all three forms: YAML + Java DSL (`CaseHub` subclass) + Java Annotations (`@Case` interface). Where annotations can't express a feature natively, `@Customize` drops into the DSL — showing the blend honestly.
**Alternatives:**
- YAML + best-fit Java — one Java file using whichever pathway fits. Less work but doesn't show all three side-by-side.
- YAML + DSL only — skip annotations. Misses the annotation pathway which is the primary Java entry point.
- YAML first, Java later — faster to ship but parity isn't proven until all three exist.
**Rationale:** The "Rosetta Stone" approach — same case definition in all three forms. Shows users exactly where each pathway is self-sufficient and where it needs `@Customize` escape hatches. Directly proves YAML/Java parity. Existing annotated examples (3 already exist) provide a head start.
**Trade-offs:** ~24 files for 8 examples (8 YAML + 8 DSL + 8 annotations). Some annotation versions will need `@Customize` for features annotations can't express natively (A2A, MCP, SubCase mappings). That's honest documentation, not a weakness.
**Depends on:** D2 (scope), D3 (domain)
**Sources:** `examples/` (9 existing annotated Java examples), `casehub-engine-annotations` module, `api/model/converter/YamlCaseDefinitionConverter.java`
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

## D4: Three-pathway pairing

**Choice:** Every example exists in all three forms: YAML + Java DSL (`CaseHub` subclass) + Java Annotations (`@Case` interface). Where annotations can't express a feature natively, `@Customize` drops into the DSL — showing the blend honestly.
**Alternatives:**
- YAML + best-fit Java — one Java file using whichever pathway fits. Less work but doesn't show all three side-by-side.
- YAML + DSL only — skip annotations. Misses the annotation pathway which is the primary Java entry point.
- YAML first, Java later — faster to ship but parity isn't proven until all three exist.
**Rationale:** The "Rosetta Stone" approach — same case definition in all three forms. Shows users exactly where each pathway is self-sufficient and where it needs `@Customize` escape hatches. Directly proves YAML/Java parity. Existing annotated examples (3 already exist) provide a head start.
**Trade-offs:** ~24 files for 8 examples (8 YAML + 8 DSL + 8 annotations). Some annotation versions will need `@Customize` for features annotations can't express natively (A2A, MCP, SubCase mappings). That's honest documentation, not a weakness.
**Depends on:** D2 (scope), D3 (domain)
**Sources:** `examples/` (9 existing annotated Java examples), `casehub-engine-annotations` module, `api/model/converter/YamlCaseDefinitionConverter.java`
**Exploration:** quick
**Status:** captured
