---
layout: post
title: "The Batch That Paid for Itself"
date: 2026-06-13
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-engine]
tags: [cdi, spi, composition, risk-classifier]
---

When the issue backlog accumulates fifteen XS/S items, the temptation is to knock them out one by one across separate branches. I tried the opposite — one branch, all fifteen, sequential execution.

## Triage before work

Three categories emerged from scanning the full issue list. Design questions (#465 panel event model, #464 naming review, #466 compile scope review) that needed analysis but no code. Already-resolved issues (#409 binary incompatibility, #479 domainContentBytes overrides) where the fix had landed in a previous session but the issue was never closed. And actual code changes — SPI signatures, schema wiring, composition refactors, CDI event promotions.

The design questions and already-resolved issues closed with comments. No commits, no tests, no risk. Five issues gone before touching a source file.

## The inheritance that wasn't visible

The composition refactor on `CaseLedgerEntryRepository` was supposed to be textbook. Remove `extends JpaLedgerEntryRepository`, inject `LedgerEntryRepository` via CDI, keep the case-specific query methods. Build succeeded. All 78 tests failed.

The problem: when a `@DefaultBean` extends an `@Alternative`, CDI treats the child as satisfying all parent-type injection points — without the parent needing explicit activation via `selected-alternatives`. The child is a stealth activation of the parent. Remove the inheritance, and those injection points vanish. No compile error. No CDI warning. Pure runtime failure.

The fix was one line in `application.properties` — explicitly activating the `@Alternative` parent. The lesson is that CDI type resolution through inheritance is implicit and invisible. Any composition refactor on a bean hierarchy involving `@DefaultBean` and `@Alternative` needs to audit which types the child was silently providing.

## The fail-safe that wasn't safe

Code review caught a compliance gap in `ChainedReactiveActionRiskClassifier`. The chain recovers from Uni-level failures (a classifier's reactive pipeline throws asynchronously), but a classifier that throws synchronously — during argument validation before the Uni is even constructed — escapes the fail-safe entirely. The stream materialization in `.map(c -> c.classify(action))` runs eagerly. A `NullPointerException` there bypasses `.onFailure().recoverWithItem()` because the Uni was never created.

For AML and clinical deployments, the fail-safe contract is that any classifier failure results in `GateRequired` — the action is gated for manual review. A synchronous throw violating that contract means a consequential action could proceed without human oversight. The fix was a try/catch around the stream materialization, matching the blocking path's existing pattern.

## What the batch revealed

The SPI signature changes (adding `tenancyId` to `terminate()`, extending the risk classifier chain for reactive consumers) each required updating every implementation — interfaces, no-ops, contract tests, integration test stubs — in the same commit. This is the protocol working as intended. Breaking changes cost nothing externally; the migration is mechanical and the breakage forces every caller to be explicit.

The batch approach works for issues that share no dependencies between them. Each commit is atomic, each issue closes independently, and the branch reads as a coherent changelog rather than fifteen separate PRs with one commit each.
