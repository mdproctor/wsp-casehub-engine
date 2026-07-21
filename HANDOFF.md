# Handoff — 2026-07-21

## What's Done

- **engine#762**: Extracted `casehub-engine-rest` module — 5 JAX-RS resources, 9 DTOs,
  CaseService, exception mappers. Virtual threads, SPI pagination, no Panache, no ACL.
  PR #766 merged upstream. Design spec adversarially reviewed (4 rounds, 24 issues).

## Immediate Next Step

**scaffold#35** — replace scaffold's inline REST with `casehub-engine-rest` dependency.
Delete 19 source files, add `ContainerRequestFilter` for ACL, update test imports.
https://github.com/casehubio/scaffold/issues/35

## Cross-Module

**Enabled:**
- `scaffold` — `casehub-engine-rest` jar published; scaffold#35 replaces inline REST · M · Med

## What's Left

- engine#764: update architecture spec §5 Connectors · S · Low
- Work repo DataRef support — follow-on from #740 (not yet filed) · M · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| scaffold#35 | Replace scaffold inline REST with engine-rest | M | Med | engine-rest jar available |
| #754 | HumanTask CBR routing implementation | M | Med | Follow-on from #741 |
| #755 | HumanTask routing constraint impl | M | Med | Follow-on from #741 |
| #764 | Update architecture spec §5 Connectors | S | Low | |
