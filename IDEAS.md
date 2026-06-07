# Idea Log

Undecided possibilities — things worth remembering but not yet decided.
Promote to an ADR when ready to decide; discard when no longer relevant.

---

## 2026-06-07 — Pluggable candidateGroup routing strategies for humanTask

**Priority:** high
**Status:** promoted

Rather than sealing humanTask candidate group resolution to JQ expressions and
static lists, expose a named-strategy SPI. Each strategy is a CDI bean; harness
authors select the strategy by name in YAML. The JQ evaluator becomes the default
strategy, not the only one. A Drools strategy could fire rules against the case
context; an ML strategy could call a model. Generalising to a strategy pattern
would make `ListEvaluator` an implementation detail of the JQ strategy, not a
top-level API. The same extensibility question applies to all routing decision
points across the platform (worker routing, SLA escalation, sub-case routing).

**Context:** engine#387 shipped JQ expressions for `candidateGroups` (sealed
`ListEvaluator` hierarchy). Post-merge discussion raised case-based routing via
Drools and the need for a universal routing architecture — consistent SPI pattern
documented in PLATFORM.md and protocols so all repos use the same approach. Also
affects engine#439 (dynamic title/scope/expiresIn).

**Promoted to:** engine#442
