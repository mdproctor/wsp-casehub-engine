# TS Event Decorator Framework — Design Spec

**Issue:** casehubio/examples#216
**Date:** 2026-11-26
**Branch:** issue-216-ts-event-decorator-framework

## Overview

TypeScript decorator framework mirroring the YAML binding paradigm. Provides `@Observes`, `@Publish`, and `@ApplicationScoped` decorators wired to push topics via `EventContainer`. Standalone classes (no UI framework dependency) managed as singletons — the CDI model brought to the browser.

## Package

**Name:** `@casehubio/pages-event-decorators`
**Location:** `packages/pages-event-decorators/`
**Dependencies:** `@casehubio/pages-data` (for `EventStream`, `EventStreamPool`, `defaultPool`)

No dependency on Lit, `reflect-metadata`, or any polyfill. Uses TC39 legacy decorators (`experimentalDecorators: true`, already in `pages-tsconfig`).

## Decorators

### @ApplicationScoped

Class decorator. Marks a class for singleton lifecycle management by `EventContainer`.

```typescript
function ApplicationScoped(constructor: Function): void
```

Stores a marker on the constructor via a well-known Symbol. `EventContainer` reads this to confirm a class is container-managed before instantiating it.

A class without `@ApplicationScoped` passed to `container.register()` throws at registration time — fail fast, not at runtime.

### @Observes(topic)

Method decorator. Registers the method as a handler for push events on the given topic pattern.

```typescript
function Observes(topic: string): MethodDecorator
```

- `topic` supports the standard push topic syntax: exact (`helpdesk:tickets`), single-wildcard (`helpdesk:*`), multi-wildcard (`helpdesk:**`)
- Multiple `@Observes` on the same method are allowed — registers the method for each topic independently
- The method receives the event payload (typed `unknown`, consumer casts)
- Topic validation at registration time using `isValidTopicOrPattern` from `pages-data`

### @Publish(topic)

Method decorator. Wraps the method so its return value is published to a push topic via REST.

```typescript
function Publish(topic: string): MethodDecorator
```

- The original method becomes a payload builder — its return value is POSTed to `{publishBaseUrl}/api/push/publish/{topic}` via `fetch`
- Returns `Promise<{ seq: number }>` (the response from PushPublishResource)
- If the original method returns `undefined` or `null`, the publish is skipped (no-op)
- Topic validation at decoration time

## EventContainer

The runtime that instantiates decorated classes and wires push subscriptions.

```typescript
interface EventContainerConfig {
  wsUrl: string;         // WebSocket endpoint, e.g. 'ws://localhost:8190/push'
  publishBaseUrl: string; // REST base URL, e.g. 'http://localhost:8190'
  auth?: { token: string };  // optional auth token for publish requests
}

class EventContainer {
  constructor(config: EventContainerConfig);

  register(...classes: Function[]): void;  // register @ApplicationScoped classes
  start(): void;                           // instantiate, subscribe, wire
  stop(): void;                            // unsubscribe, disconnect, release

  getInstance<T>(clazz: new (...args: any[]) => T): T;  // retrieve singleton
}
```

### Lifecycle

1. **`register()`** — reads decorator metadata from each class. Validates `@ApplicationScoped` marker. Collects `@Observes` topics and `@Publish` targets. Throws if a class lacks `@ApplicationScoped`.

2. **`start()`** — for each registered class:
   - Instantiates the class (zero-arg constructor)
   - Collects all `@Observes` topics across registered classes
   - Creates `EventStream` instances via `defaultPool` from `pages-data`, subscribing to all collected topics
   - Routes incoming events to the matching `@Observes` methods (using `matchesTopic` from `pages-data`)
   - Wraps `@Publish` methods with `fetch`-based publishing to the REST endpoint

3. **`stop()`** — disconnects all `EventStream` instances, releases pool handles, clears singleton references.

### Metadata Storage

Decorators write to well-known Symbols on the constructor/prototype:

```typescript
const SCOPED = Symbol.for('casehub:application-scoped');
const OBSERVERS = Symbol.for('casehub:observers');
const PUBLISHERS = Symbol.for('casehub:publishers');
```

- `constructor[SCOPED] = true`
- `prototype[OBSERVERS]` — `Map<string, { topic: string; methodName: string }[]>`
- `prototype[PUBLISHERS]` — `Map<string, { topic: string; methodName: string }[]>`

Using `Symbol.for()` ensures stability across module boundaries in the monorepo.

## Example: helpdesk-reactions.ts

Located in `examples/helpdesk-yaml/src/main/webui/` (or similar frontend directory):

```typescript
import { ApplicationScoped, Observes, Publish } from '@casehubio/pages-event-decorators';

interface TicketEvent {
  type: 'CREATED' | 'CLASSIFIED' | 'ASSIGNED' | 'RESOLVED';
  subject: string;
  category: string;
  priority: string;
  status: string;
}

interface NotificationEvent {
  to: string;
  message: string;
}

@ApplicationScoped
class HelpdeskReactions {

  @Observes('helpdesk:tickets')
  onTicketClassified(event: TicketEvent) {
    if (event.type === 'CLASSIFIED') {
      console.log(`Ticket classified: ${event.category} / ${event.priority}`);
    }
  }

  @Observes('helpdesk:tickets')
  onTicketResolved(event: TicketEvent) {
    if (event.type === 'RESOLVED') {
      console.log(`Ticket resolved: ${event.subject}`);
    }
  }

  @Observes('helpdesk:notifications')
  onNotification(event: NotificationEvent) {
    console.log(`Notification sent to ${event.to}: ${event.message}`);
  }
}

// --- Bootstrap ---
import { EventContainer } from '@casehubio/pages-event-decorators';

const container = new EventContainer({
  wsUrl: `${location.protocol === 'https:' ? 'wss:' : 'ws:'}//${location.host}/push`,
  publishBaseUrl: location.origin,
});
container.register(HelpdeskReactions);
container.start();
```

## YAML / TS Equivalence

| YAML Binding | TS Decorator |
|---|---|
| `on: contextChange: filter:` | `@Observes('topic')` + method body filter |
| `capability:` → worker → agent | `@Observes('topic')` → method body logic |
| push publish (via REST) | `@Publish('topic')` |
| case definition scope | `@ApplicationScoped` class |

The equivalence is conceptual, not syntactic. YAML bindings are reactive rules evaluated by the engine; TS decorators are event handlers wired by `EventContainer`. Same paradigm (declarative, event-driven), different execution model (server-side engine vs. browser-side container).

## Testing Strategy

### Unit tests (Vitest)

1. **Decorator metadata** — verify `@ApplicationScoped`, `@Observes`, `@Publish` store correct metadata on Symbol properties
2. **EventContainer.register()** — validates `@ApplicationScoped` marker, collects topics, rejects undecorated classes
3. **EventContainer.start()/stop()** — mock `EventStream` and `fetch` to verify:
   - Singletons instantiated
   - Topics subscribed via `EventStream`
   - Events routed to correct `@Observes` methods
   - `@Publish` methods call `fetch` with correct URL and payload
   - `stop()` disconnects streams and clears state
4. **Topic matching** — delegates to `matchesTopic` from `pages-data` (already tested there)
5. **Edge cases** — multiple `@Observes` on one method, `@Publish` returning null (skipped), class without `@ApplicationScoped` rejected

### Integration test (helpdesk-yaml)

The `helpdesk-reactions.ts` example serves as a manual integration test — demonstrates the decorators working against the live push infrastructure in dev mode.

## File Structure

```
packages/pages-event-decorators/
  package.json
  tsconfig.json           # extends @casehubio/pages-tsconfig
  tsconfig.build.json
  vitest.config.ts
  src/
    index.ts              # barrel: decorators + EventContainer
    decorators.ts         # @ApplicationScoped, @Observes, @Publish
    symbols.ts            # well-known Symbol constants
    metadata.ts           # metadata reading utilities
    event-container.ts    # EventContainer class
  dist/                   # build output
```

## References

- `pages-data/src/event-stream/event-stream.ts` — EventStream API used by EventContainer
- `pages-data/src/event-stream/event-stream-pool.ts` — defaultPool for connection sharing
- `pages-data/src/dataset/external/sources/topic-matching.ts` — matchesTopic for event routing
- `pages-component/src/controller/event-stream-controller.ts` — existing Lit reactive controller (complementary, not replaced)
- `pages/backend/push-runtime/.../PushPublishResource.java` — REST publish endpoint
- `helpdesk-yaml/src/main/resources/casehub/helpdesk-ticket.yaml` — YAML binding paradigm being mirrored
- `pages-tsconfig/tsconfig.json` — experimentalDecorators: true, no emitDecoratorMetadata
