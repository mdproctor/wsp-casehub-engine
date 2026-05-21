# Handoff — PR Decomposition + Upstream Stabilisation
2026-05-21

## What changed this session

*Previous handover (engine#300 + upstream rebase): `git show HEAD~1:HANDOFF.md`*

**Additional work this session:**

**6 focused PRs opened against casehubio/engine** (#305–#310) — all on hold, do not push/merge yet:
- #305: SubCase extraction (engine#252)
- #306: Work adapter fixes (engine#278–280)
- #307: Test quality (engine#282, #290–292)
- #308: humanTask YAML binding (engine#293)
- #309: Deadline propagation (engine#300 + parent#6)
- #310: Test name cleanup (engine#304)

Old batch PR #296 closed (superseded). `mdproctor/engine` PR #1 closed + branch deleted (content already in main as `da3c41a`).

**Decision: do NOT push to casehubio upstream yet.** Cross-repo dependency on work/ledger not resolved. Plan: stabilise all fork mains first, then push upstream together.

**Garden:** GE-20260521-c89fd1 — cherry-pick conflict resolution via `git checkout <branch> -- <file>`.

## Immediate Next Step

Stabilise commits across remaining repos (work, ledger, qhorus, etc.) before any casehubio upstream pushes.

## What's Next

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## Cross-Module

**PRs against casehubio/engine (#305–#310)** depend on casehubio/work and casehubio/ledger having compatible APIs. Do not request review until those repos are also pushed to upstream. Recommended merge order when ready: #305 → #306 → #307 → #308 → #309 → #310.
