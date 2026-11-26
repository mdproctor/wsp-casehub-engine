# TS Event Decorator Framework Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development). Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** casehubio/examples#216 — TS event decorator framework
**Issue group:** casehubio/examples#212 (parent epic)

**Goal:** Create `@casehubio/pages-event-decorators` package with `@Observes`, `@Publish`, `@ApplicationScoped` decorators and `EventContainer`, plus a helpdesk example.

**Architecture:** Three decorators store metadata on well-known Symbols. `EventContainer` reads metadata at registration, instantiates singletons, creates `EventStream` instances from `@casehubio/pages-data`'s pool for `@Observes` subscriptions, and wraps `@Publish` methods with `fetch` to the REST publish endpoint.

**Tech Stack:** TypeScript (ES2022), Vitest, `@casehubio/pages-data` (EventStream, matchesTopic, isValidTopicOrPattern)

## Global Constraints

- `experimentalDecorators: true` (from pages-tsconfig), no `emitDecoratorMetadata`
- ESM modules (`"type": "module"`) with `.js` import extensions in source
- No `reflect-metadata` polyfill
- Reuse `defaultPool` from `@casehubio/pages-data` for WebSocket connection sharing
- Publish via REST `POST /api/push/publish/{topic}`, not WebSocket

---

## Batch 1: Package scaffold + decorators + unit tests

### Task 1: Package scaffold and decorator metadata

**Files:**
- Create: `packages/pages-event-decorators/package.json`
- Create: `packages/pages-event-decorators/tsconfig.json`
- Create: `packages/pages-event-decorators/tsconfig.build.json`
- Create: `packages/pages-event-decorators/vitest.config.ts`
- Create: `packages/pages-event-decorators/src/symbols.ts`
- Create: `packages/pages-event-decorators/src/decorators.ts`
- Create: `packages/pages-event-decorators/src/metadata.ts`
- Create: `packages/pages-event-decorators/src/index.ts`
- Test: `packages/pages-event-decorators/src/decorators.test.ts`

**Interfaces:**
- Produces: `SCOPED`, `OBSERVERS`, `PUBLISHERS` Symbols; `ApplicationScoped`, `Observes(topic)`, `Publish(topic)` decorators; `getObservers(proto)`, `getPublishers(proto)`, `isApplicationScoped(ctor)` metadata readers

- [ ] **Step 1: Create package.json**

```json
{
  "name": "@casehubio/pages-event-decorators",
  "version": "0.1.0",
  "description": "CDI-inspired decorators for push topic event handling",
  "repository": {
    "type": "git",
    "url": "https://github.com/casehubio/casehub-pages.git"
  },
  "publishConfig": {
    "registry": "https://npm.pkg.github.com"
  },
  "type": "module",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "exports": {
    ".": "./dist/index.js"
  },
  "scripts": {
    "build": "tsc -p tsconfig.build.json",
    "typecheck": "tsc --noEmit",
    "test": "vitest run",
    "test:watch": "vitest",
    "clean": "rimraf dist .typecheck"
  },
  "dependencies": {
    "@casehubio/pages-data": "workspace:*"
  },
  "devDependencies": {
    "@casehubio/pages-tsconfig": "workspace:*",
    "vitest": "^3.2.1"
  },
  "files": ["dist", "package.json"]
}
```

- [ ] **Step 2: Create tsconfig.json**

```json
{
  "extends": "@casehubio/pages-tsconfig/tsconfig.json",
  "compilerOptions": {
    "rootDir": "src",
    "outDir": ".typecheck",
    "emitDeclarationOnly": true
  },
  "include": ["src"],
  "references": []
}
```

- [ ] **Step 3: Create tsconfig.build.json**

```json
{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "outDir": "dist",
    "emitDeclarationOnly": false,
    "composite": false
  },
  "exclude": ["**/*.test.ts"]
}
```

- [ ] **Step 4: Create vitest.config.ts**

```typescript
import { defineConfig } from "vitest/config";

export default defineConfig({
  test: {
    globals: true,
    environment: "node",
    include: ["src/**/*.test.ts"],
  },
});
```

- [ ] **Step 5: Create src/symbols.ts**

```typescript
export const SCOPED = Symbol.for("casehub:application-scoped");
export const OBSERVERS = Symbol.for("casehub:observers");
export const PUBLISHERS = Symbol.for("casehub:publishers");
```

- [ ] **Step 6: Write the failing decorator tests**

Create `src/decorators.test.ts`:

```typescript
import { describe, it, expect } from "vitest";
import { ApplicationScoped, Observes, Publish } from "./decorators.js";
import { isApplicationScoped, getObservers, getPublishers } from "./metadata.js";

describe("@ApplicationScoped", () => {
  it("marks a class as container-managed", () => {
    @ApplicationScoped
    class MyBean {}

    expect(isApplicationScoped(MyBean)).toBe(true);
  });

  it("undecorated class is not scoped", () => {
    class Plain {}

    expect(isApplicationScoped(Plain)).toBe(false);
  });
});

describe("@Observes", () => {
  it("registers a method for a topic", () => {
    class Bean {
      @Observes("orders:new")
      onOrder(_event: unknown) {}
    }

    const observers = getObservers(Bean.prototype);
    expect(observers).toEqual([
      { topic: "orders:new", methodName: "onOrder" },
    ]);
  });

  it("supports multiple @Observes on the same method", () => {
    class Bean {
      @Observes("a:topic")
      @Observes("b:topic")
      onBoth(_event: unknown) {}
    }

    const observers = getObservers(Bean.prototype);
    expect(observers).toHaveLength(2);
    expect(observers).toContainEqual({ topic: "a:topic", methodName: "onBoth" });
    expect(observers).toContainEqual({ topic: "b:topic", methodName: "onBoth" });
  });

  it("collects observers from multiple methods", () => {
    class Bean {
      @Observes("a:topic")
      onA(_event: unknown) {}

      @Observes("b:topic")
      onB(_event: unknown) {}
    }

    const observers = getObservers(Bean.prototype);
    expect(observers).toHaveLength(2);
  });
});

describe("@Publish", () => {
  it("registers a method as a publisher", () => {
    class Bean {
      @Publish("alerts:fire")
      sendAlert() {
        return { level: "critical" };
      }
    }

    const publishers = getPublishers(Bean.prototype);
    expect(publishers).toEqual([
      { topic: "alerts:fire", methodName: "sendAlert" },
    ]);
  });
});
```

- [ ] **Step 7: Run test to verify it fails**

Run: `cd packages/pages-event-decorators && npx vitest run`
Expected: FAIL — modules not found

- [ ] **Step 8: Create src/decorators.ts**

```typescript
import { SCOPED, OBSERVERS, PUBLISHERS } from "./symbols.js";

export interface ObserverEntry {
  topic: string;
  methodName: string;
}

export interface PublisherEntry {
  topic: string;
  methodName: string;
}

export function ApplicationScoped<T extends new (...args: any[]) => any>(
  constructor: T,
): T {
  (constructor as any)[SCOPED] = true;
  return constructor;
}

export function Observes(topic: string): MethodDecorator {
  return function (
    target: Object,
    propertyKey: string | symbol,
    _descriptor: PropertyDescriptor,
  ) {
    const entries: ObserverEntry[] = (target as any)[OBSERVERS] ?? [];
    entries.push({ topic, methodName: String(propertyKey) });
    (target as any)[OBSERVERS] = entries;
  };
}

export function Publish(topic: string): MethodDecorator {
  return function (
    target: Object,
    propertyKey: string | symbol,
    _descriptor: PropertyDescriptor,
  ) {
    const entries: PublisherEntry[] = (target as any)[PUBLISHERS] ?? [];
    entries.push({ topic, methodName: String(propertyKey) });
    (target as any)[PUBLISHERS] = entries;
  };
}
```

- [ ] **Step 9: Create src/metadata.ts**

```typescript
import { SCOPED, OBSERVERS, PUBLISHERS } from "./symbols.js";
import type { ObserverEntry, PublisherEntry } from "./decorators.js";

export function isApplicationScoped(constructor: Function): boolean {
  return (constructor as any)[SCOPED] === true;
}

export function getObservers(prototype: Object): readonly ObserverEntry[] {
  return (prototype as any)[OBSERVERS] ?? [];
}

export function getPublishers(prototype: Object): readonly PublisherEntry[] {
  return (prototype as any)[PUBLISHERS] ?? [];
}
```

- [ ] **Step 10: Create src/index.ts**

```typescript
export { ApplicationScoped, Observes, Publish } from "./decorators.js";
export type { ObserverEntry, PublisherEntry } from "./decorators.js";
export { isApplicationScoped, getObservers, getPublishers } from "./metadata.js";
export { SCOPED, OBSERVERS, PUBLISHERS } from "./symbols.js";
```

(EventContainer will be added in Task 2 and exported here.)

- [ ] **Step 11: Run tests to verify they pass**

Run: `cd packages/pages-event-decorators && npx vitest run`
Expected: PASS — all 6 tests

- [ ] **Step 12: Run typecheck**

Run: `cd packages/pages-event-decorators && npx tsc --noEmit`
Expected: no errors

- [ ] **Step 13: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/231/pages add packages/pages-event-decorators/
git -C /Users/mdproctor/claude/casehub/slots/231/pages commit -m "feat(#216): add pages-event-decorators package — decorators + metadata

@ApplicationScoped, @Observes, @Publish decorators storing metadata on
well-known Symbols. Metadata readers for EventContainer consumption.

Refs casehubio/examples#216"
```

---

### Task 2: EventContainer — registration, start, stop, event routing

**Files:**
- Create: `packages/pages-event-decorators/src/event-container.ts`
- Modify: `packages/pages-event-decorators/src/index.ts` (add EventContainer export)
- Test: `packages/pages-event-decorators/src/event-container.test.ts`

**Interfaces:**
- Consumes: `isApplicationScoped(ctor)`, `getObservers(proto)`, `getPublishers(proto)` from Task 1
- Consumes: `EventStream`, `matchesTopic`, `isValidTopicOrPattern` from `@casehubio/pages-data`
- Produces: `EventContainer` class with `register()`, `start()`, `stop()`, `getInstance()`

- [ ] **Step 1: Write failing EventContainer tests**

Create `src/event-container.test.ts`:

```typescript
import { describe, it, expect, vi, beforeEach } from "vitest";
import { ApplicationScoped, Observes, Publish } from "./decorators.js";
import { EventContainer } from "./event-container.js";

vi.mock("@casehubio/pages-data", () => {
  const instances: any[] = [];
  return {
    EventStream: vi.fn().mockImplementation(function (
      this: any,
      _url: string,
      topics: string | string[],
      options?: any,
    ) {
      this.topics = Array.isArray(topics) ? topics : [topics];
      this.onChange = options?.onChange;
      this._latest = undefined;
      this.connect = vi.fn();
      this.disconnect = vi.fn();
      Object.defineProperty(this, "latest", {
        get: () => this._latest,
      });
      instances.push(this);
    }),
    matchesTopic: vi.fn(
      (pattern: string, topic: string) => pattern === topic,
    ),
    isValidTopicOrPattern: vi.fn(() => true),
    __instances: instances,
  };
});

function getStreamInstances(): any[] {
  // eslint-disable-next-line @typescript-eslint/no-require-imports
  return (vi.mocked(await import("@casehubio/pages-data")) as any)
    .__instances;
}

describe("EventContainer", () => {
  beforeEach(() => {
    vi.clearAllMocks();
  });

  it("rejects undecorated classes", () => {
    class Plain {}

    const container = new EventContainer({
      wsUrl: "ws://test/push",
      publishBaseUrl: "http://test",
    });
    expect(() => container.register(Plain)).toThrow(
      /not decorated with @ApplicationScoped/,
    );
  });

  it("accepts @ApplicationScoped classes", () => {
    @ApplicationScoped
    class Bean {}

    const container = new EventContainer({
      wsUrl: "ws://test/push",
      publishBaseUrl: "http://test",
    });
    expect(() => container.register(Bean)).not.toThrow();
  });

  it("instantiates singletons on start", () => {
    @ApplicationScoped
    class Bean {
      started = true;
    }

    const container = new EventContainer({
      wsUrl: "ws://test/push",
      publishBaseUrl: "http://test",
    });
    container.register(Bean);
    container.start();

    const instance = container.getInstance(Bean);
    expect(instance).toBeInstanceOf(Bean);
    expect(instance.started).toBe(true);
  });

  it("returns same singleton on repeated getInstance", () => {
    @ApplicationScoped
    class Bean {}

    const container = new EventContainer({
      wsUrl: "ws://test/push",
      publishBaseUrl: "http://test",
    });
    container.register(Bean);
    container.start();

    expect(container.getInstance(Bean)).toBe(container.getInstance(Bean));
  });

  it("routes events to @Observes methods", async () => {
    const received: unknown[] = [];

    @ApplicationScoped
    class Handler {
      @Observes("orders:new")
      onOrder(event: unknown) {
        received.push(event);
      }
    }

    const container = new EventContainer({
      wsUrl: "ws://test/push",
      publishBaseUrl: "http://test",
    });
    container.register(Handler);
    container.start();

    // Simulate event delivery via the EventStream onChange callback
    const { EventStream } = await import("@casehubio/pages-data");
    const stream = vi.mocked(EventStream).mock.instances[0] as any;
    stream._latest = { item: "widget" };
    stream.onChange?.();

    expect(received).toEqual([{ item: "widget" }]);
  });

  it("stop disconnects streams and clears singletons", () => {
    @ApplicationScoped
    class Bean {}

    const container = new EventContainer({
      wsUrl: "ws://test/push",
      publishBaseUrl: "http://test",
    });
    container.register(Bean);
    container.start();
    container.stop();

    expect(() => container.getInstance(Bean)).toThrow();
  });
});

describe("@Publish wiring", () => {
  it("publishes return value via fetch", async () => {
    const fetchSpy = vi.fn().mockResolvedValue({
      ok: true,
      json: () => Promise.resolve({ seq: 1 }),
    });
    vi.stubGlobal("fetch", fetchSpy);

    @ApplicationScoped
    class Publisher {
      @Publish("alerts:fire")
      sendAlert() {
        return { level: "critical" };
      }
    }

    const container = new EventContainer({
      wsUrl: "ws://test/push",
      publishBaseUrl: "http://test",
    });
    container.register(Publisher);
    container.start();

    const instance = container.getInstance(Publisher);
    const result = await (instance.sendAlert as any)();

    expect(fetchSpy).toHaveBeenCalledWith(
      "http://test/api/push/publish/alerts:fire",
      expect.objectContaining({
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ level: "critical" }),
      }),
    );
    expect(result).toEqual({ seq: 1 });

    vi.unstubAllGlobals();
  });

  it("skips publish when method returns null", async () => {
    const fetchSpy = vi.fn();
    vi.stubGlobal("fetch", fetchSpy);

    @ApplicationScoped
    class Publisher {
      @Publish("alerts:fire")
      sendNothing() {
        return null;
      }
    }

    const container = new EventContainer({
      wsUrl: "ws://test/push",
      publishBaseUrl: "http://test",
    });
    container.register(Publisher);
    container.start();

    const instance = container.getInstance(Publisher);
    await (instance.sendNothing as any)();

    expect(fetchSpy).not.toHaveBeenCalled();

    vi.unstubAllGlobals();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd packages/pages-event-decorators && npx vitest run`
Expected: FAIL — `event-container.js` not found

- [ ] **Step 3: Implement EventContainer**

Create `src/event-container.ts`:

```typescript
import { EventStream, matchesTopic, isValidTopicOrPattern } from "@casehubio/pages-data";
import { isApplicationScoped, getObservers, getPublishers } from "./metadata.js";
import type { ObserverEntry, PublisherEntry } from "./decorators.js";

export interface EventContainerConfig {
  wsUrl: string;
  publishBaseUrl: string;
  auth?: { token: string };
}

interface Registration {
  constructor: new (...args: any[]) => any;
  observers: readonly ObserverEntry[];
  publishers: readonly PublisherEntry[];
}

export class EventContainer {
  private readonly config: EventContainerConfig;
  private readonly registrations: Registration[] = [];
  private readonly instances = new Map<Function, any>();
  private readonly streams: EventStream[] = [];
  private started = false;

  constructor(config: EventContainerConfig) {
    this.config = config;
  }

  register(...classes: Function[]): void {
    for (const clazz of classes) {
      if (!isApplicationScoped(clazz)) {
        throw new Error(
          `${clazz.name} is not decorated with @ApplicationScoped`,
        );
      }
      const proto = clazz.prototype;
      const observers = getObservers(proto);
      const publishers = getPublishers(proto);

      for (const o of observers) {
        if (!isValidTopicOrPattern(o.topic)) {
          throw new Error(
            `Invalid topic pattern '${o.topic}' on ${clazz.name}.${o.methodName}`,
          );
        }
      }
      for (const p of publishers) {
        if (!isValidTopicOrPattern(p.topic)) {
          throw new Error(
            `Invalid topic pattern '${p.topic}' on ${clazz.name}.${p.methodName}`,
          );
        }
      }

      this.registrations.push({
        constructor: clazz as new (...args: any[]) => any,
        observers,
        publishers,
      });
    }
  }

  start(): void {
    if (this.started) return;
    this.started = true;

    for (const reg of this.registrations) {
      const instance = new reg.constructor();
      this.instances.set(reg.constructor, instance);
      this.wirePublishers(instance, reg.publishers);
    }

    const allTopics = this.registrations.flatMap((r) =>
      r.observers.map((o) => o.topic),
    );
    const uniqueTopics = [...new Set(allTopics)];

    if (uniqueTopics.length > 0) {
      const stream = new EventStream(this.config.wsUrl, uniqueTopics, {
        onChange: () => this.dispatchLatest(stream),
      });
      this.streams.push(stream);
      stream.connect();
    }
  }

  stop(): void {
    for (const stream of this.streams) {
      stream.disconnect();
    }
    this.streams.length = 0;
    this.instances.clear();
    this.started = false;
  }

  getInstance<T>(clazz: new (...args: any[]) => T): T {
    const instance = this.instances.get(clazz);
    if (!instance) {
      throw new Error(
        `No instance for ${clazz.name} — not registered or container not started`,
      );
    }
    return instance as T;
  }

  private dispatchLatest(stream: EventStream): void {
    const event = stream.latest;
    if (!event) return;

    const detail = event as { topic?: string; payload?: unknown };
    const topic = detail.topic;
    const payload = detail.payload ?? event;

    for (const reg of this.registrations) {
      const instance = this.instances.get(reg.constructor);
      if (!instance) continue;

      for (const obs of reg.observers) {
        if (topic && matchesTopic(obs.topic, topic)) {
          instance[obs.methodName](payload);
        } else if (!topic) {
          instance[obs.methodName](event);
        }
      }
    }
  }

  private wirePublishers(
    instance: any,
    publishers: readonly PublisherEntry[],
  ): void {
    const { publishBaseUrl, auth } = this.config;

    for (const pub of publishers) {
      const original = instance[pub.methodName].bind(instance);
      instance[pub.methodName] = async (
        ...args: any[]
      ): Promise<{ seq: number } | undefined> => {
        const payload = original(...args);
        if (payload == null) return undefined;

        const headers: Record<string, string> = {
          "Content-Type": "application/json",
        };
        if (auth?.token) {
          headers["Authorization"] = `Bearer ${auth.token}`;
        }

        const response = await fetch(
          `${publishBaseUrl}/api/push/publish/${pub.topic}`,
          {
            method: "POST",
            headers,
            body: JSON.stringify(payload),
          },
        );
        return response.json() as Promise<{ seq: number }>;
      };
    }
  }
}
```

- [ ] **Step 4: Add EventContainer export to index.ts**

Append to `src/index.ts`:

```typescript
export { EventContainer } from "./event-container.js";
export type { EventContainerConfig } from "./event-container.js";
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `cd packages/pages-event-decorators && npx vitest run`
Expected: PASS — all tests

- [ ] **Step 6: Run typecheck**

Run: `cd packages/pages-event-decorators && npx tsc --noEmit`
Expected: no errors

- [ ] **Step 7: Build**

Run: `cd packages/pages-event-decorators && npx tsc -p tsconfig.build.json`
Expected: `dist/` populated with `.js` and `.d.ts` files

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/231/pages add packages/pages-event-decorators/
git -C /Users/mdproctor/claude/casehub/slots/231/pages commit -m "feat(#216): add EventContainer — singleton lifecycle + event routing

EventContainer registers @ApplicationScoped classes, instantiates
singletons, subscribes @Observes topics via EventStream pool, and
wires @Publish methods to REST endpoint via fetch.

Refs casehubio/examples#216"
```

---

## Batch 2: Helpdesk example

### Task 3: helpdesk-reactions.ts example

**Files:**
- Create: `examples/helpdesk-yaml/src/main/webui/helpdesk-reactions.ts`

**Interfaces:**
- Consumes: `ApplicationScoped`, `Observes`, `EventContainer` from Task 1 & 2

- [ ] **Step 1: Create helpdesk-reactions.ts**

```typescript
import {
  ApplicationScoped,
  Observes,
  EventContainer,
} from "@casehubio/pages-event-decorators";

interface TicketEvent {
  type: "CREATED" | "CLASSIFIED" | "ASSIGNED" | "RESOLVED";
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
  @Observes("helpdesk:tickets")
  onTicketClassified(event: TicketEvent) {
    if (event.type === "CLASSIFIED") {
      console.log(`Ticket classified: ${event.category} / ${event.priority}`);
    }
  }

  @Observes("helpdesk:tickets")
  onTicketResolved(event: TicketEvent) {
    if (event.type === "RESOLVED") {
      console.log(`Ticket resolved: ${event.subject}`);
    }
  }

  @Observes("helpdesk:notifications")
  onNotification(event: NotificationEvent) {
    console.log(`Notification sent to ${event.to}: ${event.message}`);
  }
}

const wsProtocol = location.protocol === "https:" ? "wss:" : "ws:";
const container = new EventContainer({
  wsUrl: `${wsProtocol}//${location.host}/push`,
  publishBaseUrl: location.origin,
});
container.register(HelpdeskReactions);
container.start();

export { container, HelpdeskReactions };
```

- [ ] **Step 2: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/231/examples add helpdesk-yaml/src/main/webui/
git -C /Users/mdproctor/claude/casehub/slots/231/examples commit -m "feat(#216): add helpdesk-reactions.ts — YAML/TS equivalence example

Demonstrates @Observes decorators listening to helpdesk push topics,
mirroring the YAML binding paradigm with TypeScript syntax.

Refs casehubio/examples#216"
```

## References

- [2026-11-26-ts-event-decorator-framework-design.md] — design spec this plan implements
- [pages-data/src/event-stream/event-stream.ts] — EventStream API consumed by EventContainer
- [pages-data/src/event-stream/event-stream-pool.ts] — defaultPool for connection sharing
- [pages-data/src/dataset/external/sources/topic-matching.ts] — matchesTopic, isValidTopicOrPattern
- [pages-data/package.json] — reference package structure pattern
- [pages-data/vitest.config.ts] — reference vitest config pattern
- [GitHub casehubio/examples#216] — focal issue
