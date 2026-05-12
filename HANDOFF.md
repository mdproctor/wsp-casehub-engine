# Handoff — 2026-05-12

## What's done

- **#239 closed** — `MapCaseFile` implemented and merged to `casehubio/engine` main (`cb78922`).
  Three poc-compatible aliases (`put/get/keys`) on `CaseContextImpl`. `snapshot()` overridden
  to preserve subclass type. 14 unit tests, 505 passing overall.

- **Platform protocol PP-20260512-5f055d** written and committed to `casehub/parent`
  (`777d2f2`) — Java Optional usage rules. In `docs/protocols/java-optional-usage.md`
  and indexed.

- **CLAUDE.md** — Writing Style Guide section added to engine project CLAUDE.md (`7304354`).

- **3 garden entries** submitted: `CaseContextImpl.set(null)` no-op gotcha, snapshot
  subclass type loss gotcha, type-preserving snapshot technique (all in `jvm/`).

## What's next

- **#238** `JavaBeanCaseFile<T>` — typed POJO-backed CaseContext. Half-day minimum;
  module home TBD (needs design decision upfront). Start with brainstorming.

## Key references

- Blog: `blog/2026-05-12-mdp02-optional-short-leash.md`
- Protocol: `casehub/parent/docs/protocols/java-optional-usage.md`
- Spec: `specs/2026-05-12-map-case-file-design.md`
- Plan: `plans/2026-05-12-map-case-file.md`
