# Handoff — 2026-06-05

**Head commit (engine):** 9b29427 — fix(ci): use push trigger for publish
**Head commit (workspace):** a88cc58 — docs(issue-206-flowworker-bridge): mark closed
**Both repos on:** main
**casehubio/engine:** ✅ green (11m38s, push-triggered build+publish)

## What Changed This Session

**engine#206 — FlowWorker ↔ WorkOrchestrator bridge: implemented, merged, CI green.**

New `casehub-engine-flow` module: `FlowWorkerExecutor`, `CasehubCallableTaskBuilder` (call: casehub:dispatch YAML), `CasehubDispatch`, `CasehubFlow`. Non-blocking workflow path in `QuartzWorkerExecutionJob` — Quartz thread returns immediately, async `whenComplete` handles result/failure via event bus + `WorkerRetryContext`.

**CI fixes (24-hour debugging marathon — three root causes found):**

1. **actor-state module lacked `quarkus-maven-plugin`** — without it, Quarkus augmentation of the full engine+work+qhorus+ledger CDI graph ran inside Surefire's forked JVM at test time, hanging indefinitely on 2-core CI. Fix: add `generate-code`/`generate-code-tests` goals. Actor-state was never tested in CI before (missing from Jun 3 reactor). The CI log's visible hang point (persistence-hibernate) was misleading — Hibernate SQL traces filled the stdout pipe buffer, silencing all subsequent output. Surefire reports artifact proved persistence-hibernate passed every time.

2. **CI workflow lacked `push` trigger** — fork PRs (`mdproctor/engine` → `casehubio/engine`) get downgraded `GITHUB_TOKEN` (packages: read, not write). Previous PRs were same-repo branches. Fix: added `push: branches: [main]` trigger; removed `closed` from `pull_request` types; simplified all step conditions.

3. **No concurrency group or timeout** — stale PR runs piled up (3 concurrent), default 6h timeout. Fix: `concurrency` group with `cancel-in-progress`, `timeout-minutes: 45`.

## Immediate Next Step

Run `/work` to pick next issue. engine#404 (registry lifecycle) is the largest open item.

## Cross-Module

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## What's Left

- engine#404 — registry lifecycle: eviction + stateless-on-rest + Quartz restart · L · High

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | AI Fusion typed fact space | XL | High | New module — own session |
| engine#404 | Registry lifecycle design | L | High | Design-only |
| engine#383 | Oversight response loop | M | Med | Unblocked |
| engine#384 | PlanItem escalation state | M | Med | Unblocked |
| engine#387 | humanTask dynamic candidateGroups | M | Med | — |

## Key References

- Spec: `docs/specs/2026-06-03-flowworker-bridge-design.md`
- Blog: `blog/2026-06-04-mdp01-flow-worker-bridge.md`
- CI note: push-triggered builds now handle publish+downstream. Fork PRs only build+test.
