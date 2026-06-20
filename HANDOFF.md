# Handoff — 2026-06-20

**Branch:** `issue-539-lifecycle-alignment-faulted-obsolete` — CLOSED

## What's Done

- engine#539 closed — lifecycle alignment: OBSOLETE added to PlanItemStatus with isTerminal()/isActive() as single source of truth; PlanItemFaultedEvent workerId→bindingName; PlanItemObsoleteEvent added; 7 hardcoded checks migrated; work-adapter bridge updated for FAULTED/OBSOLETE
- engine#536 closed — BlackboardEventCodecRegistrar idempotency (already in codebase)
- engine#537 closed — CaseContextChangedEventHandler blocking=true; resume test rewritten (signal-while-suspended race fixed)
- engine#538 closed — OversightGateService SPI + move ChainedReactive to api/classification
- engine#519 closed — bindingName threaded through WorkerRetriesExhaustedEvent
- engine#518 closed — auto-exhaust PlanItem when tryProvision fails
- engine#542 closed — duplicate of #537
- PR #529 closed (stale fork), PR #526 merged (treblereel's tenancyId threading)
- CI fix: resume test race condition root-caused and fixed
- Protocol PP-20260620-ed1230 captured: lifecycle enum classification on enum
- parent#289 filed: PLATFORM.md + lifecycle coherence protocol

## Immediate Next Step

Pick up new work. CI should be green after 60e47a64.

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #541 | AgentCapability test arity fix | XS | Low | Blocked — eidos not yet published with excludedDomains |
| #527 | Add optional baseUrl to OpenAiChatModelProvider | S | Low | |
| #525 | CaseDefinitionRegistry.findByIdentity() | S | Low | |
| #523 | Add findByStatus()/findAll() to CaseInstanceRepository | S | Low | |
| #521 | Update CaseRetriever call sites for PayloadFilter | XS | Low | Mechanical |
| #500 | Add triggerTenancyId to ProvisionContext | XS | Low | |
| #540 | PlanItemStatus SUSPENDED gap | M | Med | OBSOLETE note added |
