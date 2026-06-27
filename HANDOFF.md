# Handoff — 2026-06-27

## What's Done

**engine#573 — Bounded recursive sub-case spawning (CLOSED)**

Replaced the hard circular self-reference guard in `SubCaseExecutionHandler` with a bounded depth check. `SubCase` gains `maxRecursionDepth` (int, default 0 = hard block). The handler walks the `parentCaseId` chain via `CaseInstanceCache`, counting all same-definition ancestors (total counting — prevents trampoline bypass). YAML schema caps at 20. 9 files, ~400 lines of implementation + tests. CLAUDE.md updated.

**engine#576 — Root-caused template-mode test failures (FILED)**

Root cause: `InMemoryWorkItemTemplateStore` (`@Alternative @Priority(100)`) is activated by CDI but tests write templates via Panache `persist()` (→ H2). Handler reads from in-memory store → template not found. Fix: exclude JPA template store, add in-memory to `selected-alternatives`, use `templateStore.put()` in tests. Same pattern as the existing `InMemoryWorkItemStore` fix.

## What's Left

- engine#576: fix work-adapter/inbound template-mode test failures · S · Low
- Consumer repo import migration (aml#69, clinical#92, devtown#96, life#44) · M · Low each

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #576 | Fix template-mode test failures (work-adapter + inbound) | S | Low | Root cause identified, fix documented in issue |
| aml#69 | Propagate worker-api imports to aml | S | Low | Mechanical import swap |
| clinical#92 | Propagate worker-api imports to clinical | S | Low | Mechanical import swap |
| devtown#96 | Propagate worker-api imports to devtown | S | Low | Mechanical import swap |
| life#44 | Propagate worker-api imports to life | M | Low | 15 files |
