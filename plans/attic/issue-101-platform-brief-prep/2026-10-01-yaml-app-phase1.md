# YAML App Phase 1 — Hello Case Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** casehubio/casehub-ops#101 — design: 100% YAML CaseHub applications
**Issue group:** casehubio/casehub-ops#101

**Goal:** Deliver the design spec document and create the implementation issue structure for Phase 1 (scaffold-runtime extraction, casehub-app-parent POM, hello-case example).

**Architecture:** The design spec is the primary deliverable for this issue. Phase 1 implementation (separate issues under master epic) requires: extracting a `scaffold-runtime` module from `scaffold-backend`, creating a `casehub-app-parent` POM that pre-configures all dependencies, and building a `hello-case` example that demonstrates a YAML-only CaseHub app with zero Java.

**Tech Stack:** Maven POM configuration, Quarkus, YAML, Java (scaffold internals only)

## Global Constraints

- Quarkus version: `3.32.2` (`version.quarkus.platform`)
- CaseHub version: `0.2-SNAPSHOT` (`version.io.casehub`)
- All YAML fields: camelCase per DSL conventions
- Case definitions use `spec:` wrapper, `inputSchema`/`outputSchema` field names
- `dsl: "0.1"` is the current DSL version
- No Flyway migrations in engine (schema managed by Hibernate `drop-and-create`)
- GitHub Packages for dependency resolution (`https://maven.pkg.github.com/casehubio/*`)

---

## Batch 1: Finalize design spec and create implementation issues

### Task 1: Finalize and deliver the design spec

**Files:**
- Modify: `wsp-casehub-engine/specs/issue-101-platform-brief-prep/2026-10-01-yaml-app-design.md` — change status from Draft to Final
- Modify: `wsp-casehub-engine/specs/issue-101-platform-brief-prep/pipeline.state` — set state to complete

**Interfaces:**
- Consumes: Nothing — this is the entry point
- Produces: Finalized spec document referenced by all subsequent implementation issues

- [ ] **Step 1: Update spec status to Final**

Change `**Status:** Draft` to `**Status:** Final` in the spec header.

- [ ] **Step 2: Create implementation epic on GitHub**

Create a Phase 1 epic issue on casehubio/casehub-ops:

```bash
gh issue create --repo casehubio/casehub-ops \
  --title "epic: YAML app Phase 1 — scaffold-runtime + parent POM + hello-case" \
  --body "$(cat <<'BODY'
## Context

Implementation of Phase 1 from the YAML app design spec (casehubio/casehub-ops#101).
Tracked under master epic casehubio/casehub-engine#1017.

## Scope

1. Extract `scaffold-runtime` module from `scaffold-backend` (scaffold repo)
2. Create `casehub-app-parent` POM with managed dependencies (scaffold repo)
3. Configure default dev-profile `application.properties` (scaffold-runtime)
4. Create `getting-started/hello-case/` example (examples repo)
5. Verify: `mvn quarkus:dev` runs hello-case with zero Java sources

## Deliverable

Demo 1 ("Hello Case") running end-to-end: single YAML case definition, auto-discovered, Quarkus dev mode, GraphQL endpoint for starting and querying cases.

**Scale:** L | **Complexity:** Med
BODY
)"
```

- [ ] **Step 3: Create child issues for each task**

Create child issues under the Phase 1 epic:

Issue 1: "Extract `scaffold-runtime` module from `scaffold-backend`"
- Scale: M, Complexity: Med
- Repo: casehubio/scaffold
- Move reusable runtime infrastructure (YamlCaseDefinitionLoader, CaseHubClassPathLoader, health checks, ACL filter, exception mapper, GraphQL resolver, module registry) into a new `scaffold-runtime` module. scaffold-backend becomes a thin shell depending on scaffold-runtime.

Issue 2: "Create `casehub-app-parent` POM"
- Scale: S, Complexity: Low
- Repo: casehubio/scaffold
- Parent POM with managed dependencies, Quarkus uber-jar config, default dev-profile application.properties. Published from scaffold repo.

Issue 3: "Create `getting-started/hello-case` example"
- Scale: S, Complexity: Low
- Repo: casehubio/examples
- Minimal YAML-only CaseHub app: pom.xml + one case definition YAML. Zero Java sources. Verifiable via `mvn quarkus:dev`.

- [ ] **Step 4: Close issue #101**

The design spec deliverable is complete. Close casehubio/casehub-ops#101 with a reference to the spec document and the Phase 1 implementation epic.

```bash
gh issue close 101 --repo casehubio/casehub-ops \
  --comment "$(cat <<'COMMENT'
Design spec complete: `wsp-casehub-engine/specs/issue-101-platform-brief-prep/2026-10-01-yaml-app-design.md`

10 design decisions captured. Three-phase critical path defined.
Phase 1 implementation tracked under [epic link].
COMMENT
)"
```

- [ ] **Step 5: Commit and wrap**

```bash
git -C "$WORKSPACE" add specs/ plans/
git -C "$WORKSPACE" commit -m "feat: finalize YAML app design spec Closes casehubio/casehub-ops#101"
```

---

## References

- [2026-10-01-yaml-app-design.md] — design spec this plan implements
- [decisions.md] — 10 captured design decisions
- scaffold/scaffold-backend/src/main/java/ — classes to extract into scaffold-runtime
- scaffold/scaffold-backend/src/main/resources/application.properties — index-dependency config
- scaffold/demo/definitions/ — existing demo YAML case definitions
- examples/CLAUDE.md — examples repo build conventions
- casehubio/casehub-ops#101 — tracking issue
- casehubio/casehub-engine#1017 — master epic
