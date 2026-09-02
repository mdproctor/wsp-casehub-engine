# HANDOFF — casehub-engine (#984)

## Session Summary (2026-09-02)

Two issues completed this session. First: #1018 (schema-driven YAML record generation) — built `CasehubRecordCodegen` in the codegen module, wrote `yaml-record-mappings.yaml` covering all 44 record types, wired exec-maven-plugin into the api build, deleted hand-written records. 1379/1380 api tests pass (1 pre-existing HumanTaskTargetTest failure). Merged to main.

Then started #984 (standalone YAML examples). Batch 1 complete: choreography and sequential examples, each as a three-pathway Rosetta Stone (YAML + DSL + Annotations). `simple-case-annotated` renamed to `choreography-annotated`. 6 examples remain across 3 batches.

## Key Decisions

1. Three-pathway pairing: every example exists in YAML + DSL + Annotations. `@Customize` escape hatches show where annotations reach their limits.
2. Core 8 examples for this branch (choreography, sequential, GOAP, LLM, HumanTask, SubCase, A2A, MCP). Remaining 15 from #978 are follow-on.
3. Different realistic domains per example (banking, HR, legal, etc.).

## Known Issues

- `@Case` annotation Jandex index error: all annotated example `@QuarkusTest`s fail with "Index did not contain annotation definition: io.casehub.engine.annotations.Case". Pre-existing — not caused by this branch. Annotated modules compile correctly; only the test runner fails.
- Pre-existing `HumanTaskTargetTest.isBindingTarget` failure (deprecated test from #1015).

## References

| Artifact | Path |
|----------|------|
| Design spec | `specs/issue-984-yaml-examples/2026-09-01-yaml-examples-design.md` |
| Decisions | `specs/issue-984-yaml-examples/decisions.md` |
| Plan | `plans/2026-09-01-yaml-examples.md` |
| YAML examples | `proj/examples/yaml/` |
| DSL examples | `proj/examples/choreography-dsl/`, `proj/examples/sequential-dsl/` |
| Annotated examples | `proj/examples/choreography-annotated/`, `proj/examples/sequential-annotated/` |
