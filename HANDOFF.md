# Handoff — 2026-06-20

## What's Done

**Lifecycle alignment (#539, #540) + batch fixes (#536–#544)**

- engine#539 closed — OBSOLETE on PlanItemStatus + isTerminal()/isActive() as single source of truth; PlanItemFaultedEvent workerId→bindingName; PlanItemObsoleteEvent; 7 hardcoded checks eliminated; work-adapter bridge updated for FAULTED/OBSOLETE
- engine#540 closed — SUSPENDED on PlanItemStatus (active state); markSuspended()/markResumed() on PlanItem; WorkItemLifecycleAdapter handles suspension/resume
- engine#544 closed — CaseStartedEventHandler used currentPrincipal.tenancyId() instead of instance.tenancyId on Vert.x thread
- engine#541 closed — AgentCapability 9th arg fix (eidos published)
- engine#536 closed — already in codebase
- engine#537 closed — blocking=true on CaseContextChangedEventHandler; resume test rewritten
- engine#538 closed — OversightGateService SPI + ChainedReactive moved to api
- engine#519 closed — bindingName through WorkerRetriesExhaustedEvent
- engine#518 closed — auto-exhaust on failed tryProvision
- engine#542 closed — duplicate of #537
- engine#521 closed — no engine call sites
- engine#500 closed — resolved by #530's tenancyId
- PR #529 closed, PR #526 merged
- Protocol PP-20260620-ed1230 captured
- parent#289 filed

## Immediate Next Step

Pick up new work. CI is green.

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #543 | Migrate Worker primitives to casehub-worker-api | L | High | Major refactoring |
| #527 | Add optional baseUrl to OpenAiChatModelProvider | S | Low | |
| #525 | CaseDefinitionRegistry.findByIdentity() | S | Low | |
| #523 | Add findByStatus()/findAll() to CaseInstanceRepository | S | Low | |
