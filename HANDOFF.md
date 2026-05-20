# Handoff — HITL YAML Binding + devtown Wiring
2026-05-20

## What changed this session

**engine#293 closed.** Added `humanTask` as a first-class YAML binding target type in the schema (`CaseDefinition.yaml` + jsonschema2pojo codegen) and `CaseDefinitionYamlMapper`. Wired `casehub-engine-work-adapter` in devtown — `pr-review.yaml` now uses `humanTask:` binding instead of `capability: "human-decision:pr-approval"`. devtown#30 (e2e HITL test) is unblocked.

ADR-0001 written (humanTask as first-class type — type safety over convention). Two protocols captured in casehub/parent: `PP-20260520-b2a932` (yaml-humantask-binding-type) and `PP-20260520-5d0b91` (hitl-runtime-assembly).

CLAUDE.md updated with `## Document Locations` table using `proj/wksp` relative paths — no absolute paths. casehubio/parent#34 filed to adopt this convention platform-wide.

## Immediate Next Step

Wait for treblereel to review and merge `casehubio/engine#296`. Once merged, casehubio/engine publishes to GitHub Packages and devtown#30 can add the e2e test.

## Cross-Module

**We're unblocking:**
- `devtown` — devtown#30 (e2e HITL test) needs engine#296 merged and published · M · Low

## What's Left

- `engine#297` — CaseDefinitionYamlMapper: improve error handling for malformed humanTask bindings · S · Low
- `casehubio/parent#33` — Update PLATFORM.md: add casehub-engine-work-adapter + blackboard to devtown dependency table · XS · Low
- `casehubio/parent#34` — Adopt proj/wksp relative-path convention platform-wide · L · Med
- Two modified blog files (2026-05-12-mdp02, 2026-05-13-mdp01) — pre-existing uncommitted edits · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #278 | SelectionContext arity mismatch — 2-line fix | XS | Low | Quick win |
| #281 | FailingWorkItemStore test isolation leak | S | Low | |
| #280 | Missing JpaReactivePlanItemStore contract test | S | Low | |
| #279 | JpaReactivePlanItemStore.updateStatus flush fix | S | Low | |
| devtown#30 | E2e HITL integration test | M | Low | Blocked on engine#296 merge |
| #254 | Java 21 platform migration | L | Med | |
| parent#34 | proj/wksp relative-path convention in skills | L | Med | |

## Key references

- PR: `casehubio/engine#296`
- Blog: `blog/2026-05-20-mdp01-giving-yaml-a-human-concept.md`
- ADR: `proj/docs/adr/0001-humantask-yaml-binding-target.md`
- Protocols: `PP-20260520-b2a932`, `PP-20260520-5d0b91` (casehub/parent)
- devtown branch merged to `mdproctor/devtown` main

## Unchanged

*Background, project context — retrieve with: `git show 280292d:HANDOFF.md`*
