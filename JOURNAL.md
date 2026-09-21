# Design Journal — issue-1131-evolution-readiness

## 2026-09-21 — Evolution readiness methodology implemented

### Key design decisions

**@DefaultBean rejected for multi-instance SPIs.** The spec review caught this: Quarkus Arc suppresses ALL default beans of a type when any non-default exists. For CapabilityArea (10 coexisting implementations), @DefaultBean would silently drop 9 areas when a consumer provides 1 custom one. Override happens via `CapabilityAreaRegistry.register()` with higher `@Priority` on the startup observer. Captured as protocol PP-20260921-b7c277.

**ABSENT exclusion from weighted average.** Without this, 7 neutral areas at 0.5 drag 3 healthy areas scoring 0.9 down to ~0.66 — barely above the circuit breaker's 0.6 threshold. The composite would lie about project health. Areas with no data are excluded entirely; only areas producing real assessments contribute. Captured as protocol PP-20260921-001bc9.

**Per-area compliance, not global.** The original design had a single global ComplianceLevel. The decision review revised this: each area can be at a different level, and the project level is `min(non-L0 areas)`. L0 areas don't constrain the project. Without this, an unconfigured cognitive-reasoning area (L0 until Epics 2-3) would permanently block the project from reaching L2.

**MetricSource SPI withdrawn.** The brainstorming proposed a MetricSource intermediary between data and CapabilityArea. The decision review found that #1115 explicitly rejected parallel metrics infrastructure (MetricsSnapshot was removed). Each area directly queries EventLog via `findByCaseAndTypes()` — simpler, no abstraction without value.

**tenancyId threaded early.** Every EventLog query requires tenancyId. The SPI had no external consumers yet, so adding the parameter now was a low-cost, high-value change. Threading it through 6 source files and 9 test files in one pass avoided what would have been a painful migration later.

### What was built

10 concrete CapabilityArea implementations (stability through cognitive-memory), ComplianceLevel enum (L0-L3), ComplianceChecklist with per-area requirements, ReadinessValidator producing ReadinessReport, CDI bootstrap observer, HealthPolicy expanded 8→10 areas. 89 tests, 0 failures.
