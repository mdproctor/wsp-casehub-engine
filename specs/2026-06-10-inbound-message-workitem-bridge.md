# Inbound Message → WorkItem Bridge

**Date:** 2026-06-10
**Origin:** casehubio/work#234
**Repos:** casehub-engine (implementation), casehub-work (issue update)

---

## Problem

External messages (Slack DMs, SMS, email) arrive via `casehub-connectors` and are routed to Qhorus channels by `ConnectorChannelBackend`. No path exists to create a WorkItem from these messages. The "customer emails support → WorkItem appears in someone's inbox" flow is missing.

## Architectural Decision

casehub-work and casehub-qhorus are Foundation-tier peers — neither depends on the other. The bridge between them belongs in casehub-engine, which already aggregates both dependencies.

The bridge lives in a **new thin module** in the engine repo: `casehub-engine-inbound`. It depends only on `casehub-qhorus-api` and `casehub-work-core` — not the full engine runtime. Deployments add it by classpath presence (optional module pattern).

This treats the engine repo as an aggregator for Foundation-peer bridge modules, alongside the existing `work-adapter` pattern.

## Event Flow

```
External message (Slack DM, SMS, email)
  → casehub-connectors: Event<InboundMessage>.fireAsync()
    → casehub-qhorus/connector-backend: ConnectorChannelBackend
      → MessageService.dispatch()
        → MessageReceivedEvent (CDI event, in casehub-qhorus-api)
          → casehub-engine-inbound: InboundWorkItemBridge
            → WorkItemService.create(WorkItemCreateRequest)
              → WorkItem created, linked to source channel
```

## Module: `casehub-engine-inbound`

### Location

`casehub-engine/inbound/` — Maven artifact `casehub-engine-inbound`.

### Dependencies

| Dependency | Scope | What it provides |
|---|---|---|
| `casehub-qhorus-api` | compile | `MessageReceivedEvent`, `MessageType` |
| `casehub-work-core` | compile | `WorkBroker` (not used directly — see below) |
| `casehub-work` (runtime) | compile | `WorkItemService`, `WorkItemCreateRequest` |
| `casehub-platform-api` | compile | `TenancyConstants` |

**Note:** The bridge calls `WorkItemService.create()` (in runtime) rather than `WorkBroker` (in core). `WorkBroker` handles worker selection for an existing WorkItem; `WorkItemService.create()` is the full creation path including tenant stamping, audit, timers, and CDI event firing. This matches how `WorkItemTemplateService.instantiate()` and the REST endpoint create WorkItems.

### Components

#### `InboundWorkItemBridge`

`@ApplicationScoped` CDI bean. Observes `@ObservesAsync MessageReceivedEvent`.

Responsibilities:
1. **Filter:** Only act on messages that should create WorkItems (delegate to `InboundWorkItemPolicy`)
2. **Tenant context:** Use `TenantContextRunner` to establish tenant scope from `MessageReceivedEvent.tenancyId()` — the event fires via `@ObservesAsync` which has no request context
3. **Create:** Build a `WorkItemCreateRequest` from the event and call `WorkItemService.create()`
4. **Link:** Set `callerRef` on the WorkItem to `"qhorus:channel:{channelId}"` so the WorkItem can be correlated back to the conversation

```java
@ApplicationScoped
public class InboundWorkItemBridge {

    @Inject WorkItemService workItemService;
    @Inject TenantContextRunner tenantContextRunner;
    @Inject Instance<InboundWorkItemPolicy> policies;

    void onMessage(@ObservesAsync MessageReceivedEvent event) {
        if (event.messageType() == MessageType.EVENT) return;

        InboundWorkItemPolicy policy = policies.stream().findFirst().orElse(null);
        if (policy == null) return;

        Optional<WorkItemCreateRequest> request = policy.evaluate(event);
        if (request.isEmpty()) return;

        tenantContextRunner.runInTenantContext(event.tenancyId(), () ->
            workItemService.create(request.get()));
    }
}
```

#### `InboundWorkItemPolicy` (SPI)

`@FunctionalInterface` in the bridge module. Deployments provide an `@ApplicationScoped` implementation to control which messages create WorkItems and how.

```java
@FunctionalInterface
public interface InboundWorkItemPolicy {
    Optional<WorkItemCreateRequest> evaluate(MessageReceivedEvent event);
}
```

**No `@DefaultBean` implementation.** Without a policy on the classpath, no WorkItems are created — the bridge is inert. This is deliberate: the routing decision (which messages create WorkItems, which template, which queue) is deployment-specific domain logic.

The policy receives the full `MessageReceivedEvent` and returns either:
- `Optional.empty()` — skip, don't create a WorkItem
- `Optional.of(request)` — create this WorkItem

Classification logic (is this a new request, a reply, automated noise?) lives inside the policy implementation, not in the bridge.

### What the bridge does NOT do

- **Speech-act classification** — that's the policy's job
- **Reply detection** — the policy can check `correlationId` to detect replies
- **Template resolution** — the policy builds the `WorkItemCreateRequest`, choosing the template
- **Duplicate prevention** — the policy can check for existing WorkItems via `callerRef` pattern

## Testing

### Unit tests (in `casehub-engine/inbound/`)

TDD against `InboundWorkItemBridge`:

1. No policy bean → no WorkItem created
2. Policy returns empty → no WorkItem created
3. Policy returns request → WorkItem created with correct fields
4. EVENT message type → skipped (telemetry, not actionable)
5. Tenant context established from event's tenancyId
6. callerRef set to `qhorus:channel:{channelId}`

Mock `WorkItemService` and provide a test `InboundWorkItemPolicy`.

### E2E integration test (in `casehub-engine/integration-tests/` or dedicated)

Full CDI chain test with connectors + qhorus + bridge on classpath:

1. Fire `InboundMessage` with `connectorId = "slack-inbound"` via `Event.fireAsync()`
2. Verify `ConnectorChannelBackend` routes to Qhorus channel
3. Verify `MessageReceivedEvent` fires
4. Verify `InboundWorkItemBridge` creates a WorkItem
5. Assert WorkItem has correct title, content, callerRef, tenancyId

Requires: `casehub-connectors-core`, `casehub-qhorus` (runtime), `casehub-work` (runtime), and the bridge module on the test classpath. H2 in-memory database.

## Issues to File

| Repo | Issue | Scope |
|---|---|---|
| `casehubio/engine` | feat: `casehub-engine-inbound` — bridge InboundMessage to WorkItem via MessageReceivedEvent | New module, SPI, unit tests |
| `casehubio/engine` | test: E2E integration test — Slack InboundMessage → Qhorus → WorkItem | Cross-module CDI chain verification |
| `casehubio/work` | Update #234 — design moved to casehub-engine | Comment explaining the architectural decision; re-scope or close |

## Protocols

The following existing protocols apply:

- **async-event-tenant-context-propagation** — `@ObservesAsync` handler must use `TenantContextRunner`
- **store-tenancy-stamping-on-insert** — WorkItem creation via `WorkItemService.create()` handles this
- **optional-module-pattern** — bridge activates by classpath presence; no code changes in consuming repos

## Out of Scope

- Outbound reply path (WorkItem completed → message sent back to Slack/SMS) — separate concern, tracked separately
- Policy implementations for specific domains (devtown, clinical, aml) — each application repo provides its own
- Qhorus channel auto-creation — already handled by `ConnectorChannelBackend`