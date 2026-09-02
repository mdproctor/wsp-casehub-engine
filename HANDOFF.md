# HANDOFF — casehub-engine (#984)

## Session Summary (2026-09-02)

All 8 Rosetta Stone example sets complete for #984. Each execution model exists in three pathways (YAML + DSL + Annotations):

| # | Model | Domain | YAML | DSL | Annotated |
|---|---|---|---|---|---|
| 1 | Choreography | Banking — customer onboarding | done (prev session) | done (prev session) | done (prev session) |
| 2 | Sequential | HR — employee onboarding | done (prev session) | done (prev session) | done (prev session) |
| 3 | GOAP | Legal — contract review | done | done | renamed from goap-case-annotated |
| 4 | LLM decomposition | Research — analysis pipeline | done | done | done |
| 5 | HumanTask | Finance — loan approval | done | done | done |
| 6 | SubCase | Insurance — claims processing | done | done | done |
| 7 | A2A | Intelligence — market research | done | done | done |
| 8 | MCP | DevOps — code analysis | done | done | done |

Schema validation test updated to scan `examples/yaml/` — all 8 YAML files validate.

Fix: `JsonNodeForEachAdapter.getForEach()` updated to return `ForEachDirective` (upstream `casehub-platform-yaml-core` API change).

## Key Decisions

1. Three-pathway pairing: every example exists in YAML + DSL + Annotations. `@Customize` escape hatches show where annotations reach their limits.
2. Core 8 examples for this branch (choreography, sequential, GOAP, LLM, HumanTask, SubCase, A2A, MCP). Remaining 15 from #978 are follow-on.
3. Different realistic domains per example (banking, HR, legal, research, finance, insurance, intelligence, devops).
4. DSL and annotated A2A/MCP examples use `noFunction()` workers — the `a2a:` and `mcp:` blocks are YAML-specific deployment configuration.
5. JudgmentTarget used instead of deprecated HumanTaskTarget in DSL and annotated examples.

## Known Issues

- `@Case` annotation Jandex index error: all annotated example `@QuarkusTest`s fail with "Index did not contain annotation definition: io.casehub.engine.annotations.Case". Pre-existing — not caused by this branch. Annotated modules compile correctly; only the test runner fails.
- Pre-existing `HumanTaskTargetTest.isBindingTarget` failure (deprecated test from #1015).
- Pre-existing compilation errors in `runtime` module (neocortex `MemoryInput`/`Outcome` gained new constructor parameters) and `casehub-engine-inbound` (casehub-work-api package not found). Not caused by this branch.

## References

| Artifact | Path |
|----------|------|
| Design spec | `specs/issue-984-yaml-examples/2026-09-01-yaml-examples-design.md` |
| Decisions | `specs/issue-984-yaml-examples/decisions.md` |
| Plan | `plans/2026-09-01-yaml-examples.md` |
| YAML examples (8) | `proj/examples/yaml/` |
| DSL examples (8) | `proj/examples/*-dsl/` |
| Annotated examples (8) | `proj/examples/*-annotated/` |
