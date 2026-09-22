# Handoff — Evolution Conductor Production Readiness

**Branch:** `main` (previous branch `issue-1131-evolution-readiness` closed and landed)
**Epic:** casehubio/engine#1149 (Production Readiness, Generic Extraction, UI)
**Slot:** 197
**Date:** 2026-09-22

## What Happened This Session

Completed #1132 Batches 2-4 (6 commits, 176 tests green), ran a thorough completeness audit, filed follow-up issues, and closed the branch.

### #1132 — Command Centre Conductor (COMPLETE — all 4 batches landed)

| Batch | What was built | Tests |
|-------|---------------|-------|
| Batch 2: Control | Dynamic deny list on ImprovementBudgetEnforcer (two-layer: static + dynamic), DenyPatternView, ConductorInboxManager with gate lifecycle, ConductorInboxEntry/ConductorDecision/GateResolutionPayload model types, EscalationTrigger, WatchPattern | 25 new |
| Batch 3: Steer | GatePolicy (Map-based per-stage modes), EscalationPolicy/CategoryEscalationRules, EscalationContext/EscalationResult, EscalationProvider SPI, DefaultEscalationProvider (3-layer: category rules + watch patterns + confidence), ImprovementConfig extended, GateCheckpoint/ResearchPipelineResult sealed interfaces, ResearchPipelineOrchestrator checkpoint at RESEARCH_SCOPE, ImprovementCoordinator | 22 new |
| Batch 4: API | SummaryScope, ArtifactEntry/ArtifactManifest, EvolutionSummary, SummarizationProvider SPI, DefaultSummarizationProvider (skeleton), EvolutionStateSnapshot, DefaultEngineEvolutionApi (9 methods) | 11 new |

### Completeness Audit

Thorough audit identified:
- **4 CDI event wiring gaps** — events never fired (broadcaster is dead observer)
- **10 missing API methods** on DefaultEngineEvolutionApi
- **No persistence** — all state stores are bare ConcurrentHashMap, lost on restart
- **Pipeline resume** not implemented (hypothesis gate + resume method)
- **Summarization** is a skeleton

### Architecture Decision: Persistence

Rejected "add case context writes" as a shim. The right architecture:
- Domain beans become stateless logic — inject repository SPIs
- State behind SPI boundary (same pattern as CaseInstanceRepository)
- In-memory impls for tests, case-context-backed for production
- EventLog is audit trail only (write-only, never the read path)

### Domain-Agnostic Extraction Analysis

Audited all 13 components for domain-specificity. Result: **12 of 13 already generic**. Only 3 items need extraction:
- `ImprovementRequest.targetRepo/targetPaths` → generalise target model
- `ConflictDetector` → extract to ConflictStrategy SPI
- `STRUCTURAL_DENIED_PATTERNS` → extract to DenyPatternProvider SPI

fsitrading (trading strategy evolution) is the first non-engine consumer. Full use case documented in #1148.

### Issues Filed

| # | Title | Scale | Complexity |
|---|-------|-------|------------|
| #1149 | Epic: Production readiness, generic extraction, UI | XL | High |
| #1140 | Conductor state persistence — repository SPIs | L | High |
| #1141 | CDI event wiring | S | Low |
| #1142 | API surface completion — 10 missing methods | M | Med |
| #1143 | Pipeline checkpoint completion | S | Med |
| #1144 | Summarization — EventLog-backed | M | Med |
| #1145 | YAML codegen entries | XS | Low |
| #1146 | MCP annotation adapter | S | Low |
| #1148 | Generic extraction + domain specialisation + blog | L | High |

## Queue

| # | Issue | Status |
|---|-------|--------|
| 1 | #1140 — Conductor state persistence | Next (active in .plan) |
| 2 | #1141 — CDI event wiring | Queued |
| 3 | #1145 — YAML codegen | Queued |
| 4 | #1142 — API surface completion | Queued (depends on #1140) |
| 5 | #1143 — Pipeline checkpoint | Queued (depends on #1140) |
| 6 | #1144 — Summarization | Queued |
| 7 | #1146 — MCP adapter | Queued (depends on #1142) |
| 8 | #1148 — Generic extraction | Queued (depends on #1140) |

## What's Next

| Priority | Item | Scale | Complexity | Notes |
|----------|------|-------|------------|-------|
| 1 | #1140 Persistence SPIs | L | High | Foundational — extract ConductorInboxRepository, DenyPatternStore, ImprovementBlockStore. Refactor domain beans to inject SPIs. Tenancy-aware signatures. |
| 2 | #1141 CDI event wiring | S | Low | Independent — inject Event<T> into 4 source beans, add fireAsync() calls |
| 3 | #1145 YAML codegen | XS | Low | Independent — add GatePolicy, EscalationPolicy, CategoryEscalationRules to yaml-record-mappings.yaml |

## UI Planning (Phase 4, not yet filed as issues)

blocks-ui components needed:
- Reuse: approval-gate, notification-inbox, kpi-metric-row, compliance-summary, event-trail, audit-trail-viewer, trust-score-panel, work-item-detail/row
- New: deny-pattern-editor, watch-pattern-editor, gate-policy-editor
- Workbench: evolution-workbench composing all above
- Sample page for domain extension

## Repos in Slot

| Repo | Path | Branch | Role |
|------|------|--------|------|
| engine | `slots/197/engine` | `main` | Primary |
| blocks | `slots/197/blocks` | `main` | Cognitive stack source |
| eidos | `slots/197/eidos` | `main` | Synced |
| qhorus | `slots/197/qhorus` | `main` | Synced |

## Artifacts

| Artifact | Path |
|----------|------|
| Design spec (#1131) | `wsp/specs/issue-1131-evolution-readiness/2026-09-21-evolution-readiness-methodology-design.md` |
| Design spec (#1132) | `wsp/specs/issue-1131-evolution-readiness/2026-09-21-command-centre-conductor-design.md` |
| Decisions | `wsp/specs/issue-1131-evolution-readiness/decisions.md` (D1-D27) |
| Plan (#1131) | `wsp/plans/2026-09-21-evolution-readiness-methodology.md` |
| Plan (#1132) | `wsp/plans/2026-09-21-command-centre-conductor.md` |
| Blog | `proj/docs/blog/2026-09-21-mdp01-the-hive-that-knows-its-health.md` |
| Queue | `wsp/.plan` (8 issues for #1149) |
