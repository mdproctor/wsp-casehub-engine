# Handoff — 2026-06-25

## What's Done

**engine#572 — Fix test compilation from upstream dependency changes (CLOSED)**

Two upstream breaks on main: qhorus-api `MessageReceivedEvent` gained `Instant occurredAt` (7th param), and `casehub-work-testing` was renamed to `casehub-work-persistence-memory` (`io.casehub.work.memory`). Fixed 12 files across runtime, work-adapter, casehub-engine-inbound. CLAUDE.md updated. Build green, pushed to origin+upstream.

Also: engine artifacts now installed to local Maven repo — consumer migration can proceed.

## What's Left

- Consumer repo import migration (aml#69, clinical#92, devtown#96, life#44, claudony#157) · M · Low each

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #570 | Output schema evaluation uses ExpressionEngineRegistry | S | Low | Surfaced during #567 design |
| aml#69 | Propagate worker-api imports to aml | S | Low | Mechanical import swap |
| clinical#92 | Propagate worker-api imports to clinical | S | Low | Mechanical import swap |
| devtown#96 | Propagate worker-api imports to devtown | S | Low | Mechanical import swap |
| life#44 | Propagate worker-api imports to life | M | Low | 15 files |
| claudony#157 | Propagate worker-api imports to claudony | M | Low | 19 files |
