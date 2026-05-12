---
layout: post
title: "Optional Belongs on a Short Leash"
date: 2026-05-12
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-engine]
tags: [java, api-design, migration]
---

The question of whether `MapCaseFile`'s `get()` should return `Optional<T>` or nullable `T` seemed like a design detail. It opened into something more useful.

Claude pulled up Brian Goetz's own words: "I think routinely using it as a return value for getters would definitely be over-use." The design intent was narrow — a mechanism for library method return types where null would overwhelmingly cause errors. `Stream.findFirst()`, `findById()`, `Config.get(key)`. Not map accessors, not entity getters, not method parameters.

JavaParser gets cited in academic research as the cautionary case — an API that returns `Optional` from everything. I'd always found it uncomfortable to use. Blanket null-replacement reads as noise, not safety.

We codified the result as a platform rule: use `Optional` only when finding or computing the value is the method's entire purpose, absence is a normal expected outcome, and null would overwhelmingly cause the caller to NPE. Everything else — map accessors, entity getters, fields, parameters, collection returns — uses null or `getOrDefault()`. One practical corollary: prefer `orElseGet(() -> expr)` over `orElse(expr)`, since the latter evaluates its argument even when a value is already present.

So: `get(key, Class<T>)` returns nullable `T`.

## What the poc workers actually needed

Before building the shim, I wanted to check the scope assumption. The poc's `CaseFile` was wide: identity, lifecycle, graph relationships, change listeners. Was anything being silently dropped?

We read the actual worker code. In `execute()` methods, 36 of 40 `CaseFile` calls are `get` or `put`. Identity and lifecycle methods — `getId()`, `getStatus()`, `complete()` — appear almost entirely in application-level orchestration. That code's responsibility has moved into the engine. Nothing is lost; the access path changed.

`MapCaseFile` itself is thin:

```java
public void put(String key, Object value) { set(key, value); }
public <T> T get(String key, Class<T> type) { return getAs(key, type); }
public Set<String> keys() { return getKeys(); }
```

`contains`, `putIfAbsent`, `snapshot`, locking, and versioning are inherited unchanged from `CaseContextImpl`.

## The null trap and the snapshot type loss

`set(key, null)` on an absent key is a no-op. The guard in `CaseContextImpl.set()` is `!Objects.equals(prev, value)` — when both `prev` and `value` are null, the equality fires true, the write is skipped, and the key is never inserted. Standard `Map.put(key, null)` inserts it. Poc code that writes null and then checks `contains()` will silently break during migration.

The second came from a code review. `CaseContextImpl.snapshot()` hardcodes `new CaseContextImpl(...)` — never the subclass type. Without an override, `MapCaseFile.snapshot()` returns a `CaseContextImpl`, dropping `put`, `get`, and `keys` on the copy. Claude caught it before it shipped. The fix:

```java
@Override
public CaseContext snapshot() {
    final CaseContext base = super.snapshot();
    return new MapCaseFile(base.getData());
}
```

`super.snapshot()` handles the deep copy. `getData()` on the result is already a defensive copy — no duplication needed.
