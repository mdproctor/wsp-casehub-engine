# Decisions — TS event decorator framework (#216)

## D1: Primary consumer — standalone classes

**Choice:** Plain TS classes decorated with `@ApplicationScoped` — `EventContainer` manages their lifecycle independently of any UI framework. Lit components can consume them but aren't required.

**Alternatives:**
- Lit components — decorators work directly on LitElement subclasses. Couples the framework to Lit's lifecycle and limits reuse in non-UI contexts.
- Both equally — framework-agnostic core with Lit integration layer. Over-engineered for v1 when standalone classes cover both use cases.

**Rationale:** The framework mirrors CDI's model: singletons managed by a container, not by the DOM. Lit components already have `EventStreamController` for reactive push; this fills the gap for application-layer logic outside components.

**Trade-offs:** No automatic DOM lifecycle integration — `EventContainer.start()`/`stop()` must be called explicitly.

**Sources:**
- `pages-component/src/controller/event-stream-controller.ts` — existing Lit reactive controller
- `helpdesk-yaml case YAML bindings` — the YAML paradigm being mirrored

**Exploration:** quick
**Status:** captured

## D2: Publish path — REST endpoint

**Choice:** `@Publish` methods POST to `/api/push/publish/{topic}` via `fetch`. Events go through the server, get persisted in `EventStore`, and are broadcast to all subscribers.

**Alternatives:**
- WebSocket direct — lower latency but requires the publisher to maintain a WS connection and bypasses server-side EventStore persistence.
- Configurable — default REST, allow overriding to WebSocket. Unnecessary complexity for v1.

**Rationale:** The REST endpoint already exists (`PushPublishResource`), handles auth, validates JSON, and returns a sequence number. Publishing via REST is simpler (no WebSocket state management) and works even when the publisher isn't subscribed to anything.

**Trade-offs:** Higher latency than direct WebSocket (HTTP round-trip). Acceptable for notification-style events; not suitable for high-frequency streaming.

**Sources:**
- `pages/backend/push-runtime/.../PushPublishResource.java` — POST `/api/push/publish/{topic}`
- casehub-pages#751 — push publish endpoint (landed this session)

**Exploration:** quick
**Status:** captured

## D3: Filtering — topic pattern only

**Choice:** `@Observes` filters at the topic subscription level (e.g., `@Observes('helpdesk:tickets:*')`). Content filtering is the method's responsibility.

**Alternatives:**
- Topic + content filter — `@Observes('helpdesk:tickets', { filter: '.status == "OPEN"' })`. More declarative but adds a JQ evaluator dependency to the browser bundle. The YAML parity is conceptual, not syntactic.

**Rationale:** Matches how `EventStream` already works (topic-based subscription). Content filtering in the method body is idiomatic TypeScript and doesn't require shipping a JQ runtime to the browser.

**Trade-offs:** Less declarative than the YAML equivalent. Developers write `if (event.status === 'OPEN')` instead of a filter expression. This is natural in TS.

**Sources:**
- `pages-data/src/event-stream/event-stream.ts` — topic-based EventStream API
- `pages-data/src/dataset/external/sources/topic-matching.ts` — wildcard topic matching

**Exploration:** quick
**Status:** captured

## D4: Transport — reuse EventStream pool

**Choice:** `EventContainer` creates `EventStream` instances via `defaultPool` from `pages-data`. Shares WebSocket connections with Lit components using `EventStreamController`.

**Alternatives:**
- Own connection — `EventContainer` manages its own `EventConnection`. Full control but duplicates infrastructure and wastes connections if Lit components are also subscribed to the same URL.

**Rationale:** One WebSocket connection per URL is the design intent of the pool. The decorator framework is an additional consumer of the same push infrastructure, not a replacement.

**Trade-offs:** Coupled to `pages-data`'s pool implementation. Acceptable since `pages-data` is the platform's push transport layer.

**Sources:**
- `pages-data/src/event-stream/event-stream-pool.ts` — reference-counted pool with per-topic subscription counts
- `pages-data/src/dataset/external/sources/push-pool.ts` — connection dedup by URL

**Exploration:** quick
**Status:** captured

## D5: Architecture — decorator metadata via symbol properties

**Choice:** Decorators store metadata on constructor/prototype via well-known Symbol properties. No `reflect-metadata` polyfill. `EventContainer.register()` reads these symbols to discover `@Observes` topics and `@Publish` targets. Uses TC39 stage-3 legacy decorators (`experimentalDecorators: true`, already set in pages-tsconfig).

**Alternatives:**
- reflect-metadata — standard CDI-like approach using `Reflect.defineMetadata`. Requires a polyfill (`reflect-metadata` or `core-js`), adds bundle weight, and pages doesn't use it anywhere.
- WeakMap registry — decorators write to a module-scoped WeakMap keyed by constructor/method. Works but requires the decorator module to be a singleton (broken by duplicate package installs in monorepos).

**Rationale:** Symbol properties are the simplest metadata mechanism. They survive prototype chains, don't need polyfills, and are the established pattern when `emitDecoratorMetadata` is not enabled (as in pages-tsconfig).

**Trade-offs:** Less discoverable than `Reflect.getMetadata()` — consumers must know the symbol names. Acceptable since only `EventContainer` reads the metadata.

**Sources:**
- `pages-tsconfig/tsconfig.json` — `experimentalDecorators: true`, no `emitDecoratorMetadata`
- Lit decorators in pages — precedent for decorator usage without reflect-metadata

**Exploration:** quick
**Status:** captured
