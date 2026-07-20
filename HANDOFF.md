# Handoff — 2026-07-20

## What's Done

- **engine#742**: ActionGate resolutionType threading — GateRequired → PendingActionGate → events → handler validation
- **engine#692**: Connector boundary wiring — InboundSignalBridge, InboundSignalMapping, CaseCorrelationResolver, signal auto-activation, YAML parsing. casehub-connectors wired to engine.
- **engine#740**: DataRef<T> linked data references — DataRefResolver SPI, DataRefRegistry, BridgeResolver deferred resolution
- Design-reviewed (3 rounds, 15 issues, $14), code-reviewed, squashed, pushed to upstream
- **CI green** — fixed 5 pre-existing failures: CaseChannelProvider contract tests (missing target param), case_instance_label Flyway migration, InboundSignalBridge Instance<> for optional deps, CommitmentStore.findOpenByChannelId test stub, SubjectViewOrchestrator.deleteView return type drift

## Immediate Next Step

- Pick up #741 follow-on issues (#754-757) for HumanTask CBR routing
- Or engine#764 (update architecture spec §5 Connectors to reflect inboundMappings)

## What's Left

- engine#764: update architecture spec §5 Connectors — follow-on from design review · S · Low
- Work repo DataRef support — follow-on from #740 (not yet filed) · M · Med
- 2 unrecovered specs on closed workspace branches (hygiene scan finding) · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #754 | HumanTask CBR routing implementation | M | Med | Follow-on from #741 |
| #755 | HumanTask routing constraint impl | M | Med | Follow-on from #741 |
| #756 | Work repo consumption of HumanTask routing | M | Med | Follow-on from #741 |
| #757 | Group scoring for HumanTask routing | S | Med | Follow-on from #741 |
| #764 | Update architecture spec §5 Connectors | S | Low | Superseded by inboundMappings |
