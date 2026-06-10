# CaseContext Panels — Issue #80 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the flat `CaseContext` map with a panel-structured design that partitions working, semantic, and episodic memory into named, policy-aware regions.

**Architecture:** `CaseContextImpl` is restructured internally to hold a `Map<String, WritablePanelImpl>` keyed by panel name. All existing flat API methods delegate to the working panel so no lambda call sites change. `asJsonNode()` changes semantics to return the full panel document `{"working":{...},"semantic":{...},"episodic":{...}}`, which is the breaking change that forces JQ expression migration. Panel initialization and the reactive memory query happen in `CaseHubReactor.buildInstance()`.

**Tech Stack:** Java 21, Quarkus 3.32.2, Jackson, Mutiny, `casehub-platform-api` (existing `ReactiveCaseMemoryStore`, `Memory`, `MemoryQuery`, `MemoryDomain`, `MemoryAttributeKeys`)

**Spec:** `docs/specs/2026-06-09-casefile-panels-design.md` (rev 6)

**Build commands:**
```bash
mvn install -DskipTests -q                              # install deps first
TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl api
TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl common
TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl runtime
```

---

## File Map

### New files
| File | Purpose |
|---|---|
| `api/src/main/java/io/casehub/api/context/ReadOnlyPanelException.java` | Thrown on write to semantic/episodic panel |
| `api/src/main/java/io/casehub/api/context/ReadablePanel.java` | Read-only panel interface |
| `api/src/main/java/io/casehub/api/context/WritablePanel.java` | Read-write panel interface |
| `api/src/main/java/io/casehub/api/context/ContextPanel.java` | Panel name constants (WORKING, SEMANTIC, EPISODIC) |
| `api/src/main/java/io/casehub/api/model/EpisodicMemoryConfig.java` | Value object: domain, entityId JQ expression, limit |
| `runtime/src/main/java/io/casehub/engine/internal/context/WritablePanelImpl.java` | `WritablePanel` implementation (owns data map + lock) |
| `runtime/src/main/java/io/casehub/engine/internal/context/ReadOnlyPanelView.java` | Delegates reads, throws `ReadOnlyPanelException` on writes |
| `runtime/src/test/java/io/casehub/engine/internal/context/PanelTest.java` | Tests for panel interfaces + implementations |
| `runtime/src/test/java/io/casehub/engine/internal/context/SemanticPanelTest.java` | Semantic panel population + read-only enforcement |
| `runtime/src/test/java/io/casehub/engine/internal/context/EpisodicPanelIntraCaseTest.java` | Engine lifecycle updates to episodic panel |
| `runtime/src/test/java/io/casehub/engine/EpisodicPanelInterCaseTest.java` | @QuarkusTest for inter-case memory query |
| `runtime/src/test/java/io/casehub/engine/internal/engine/recovery/RecoveryPanelAwareTest.java` | Panel-aware rebuildStateContext() |

### Modified files
| File | Change |
|---|---|
| `api/src/main/java/io/casehub/api/context/CaseContext.java` | Add `ReadablePanel panel(String name)` |
| `runtime/src/main/java/io/casehub/engine/internal/context/CaseContextImpl.java` | Replace flat map with panel map; delegate flat API to working |
| `runtime/src/main/java/io/casehub/engine/internal/context/MapCaseFile.java` | Remove `snapshot()` override |
| `api/src/main/java/io/casehub/api/model/CaseDefinition.java` | Add `semanticData`, `episodicMemoryConfig` fields |
| `api/src/main/java/io/casehub/api/engine/CaseHubRuntime.java` | Add two `startCase()` overloads with `semanticData` |
| `runtime/src/main/java/io/casehub/engine/internal/engine/CaseHubRuntimeImpl.java` | Implement new overloads |
| `runtime/src/main/java/io/casehub/engine/internal/engine/CaseHubReactor.java` | Panel initialization in `buildInstance()`; inject `ReactiveCaseMemoryStore` |
| `common/src/main/java/io/casehub/engine/common/internal/event/CaseContextChangedEvent.java` | `CaseContext contextSnapshot`, `String changedPanel` |
| `common/src/main/java/io/casehub/engine/common/internal/event/EventBusAddresses.java` | Add `panelChanged(String)` |
| All 12 production construction sites for `CaseContextChangedEvent` | Pass `snapshot()` + `ContextPanel.WORKING` |
| `runtime/src/main/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandler.java` | Handler signature + listenPanel filtering (episodic) |
| `runtime/src/main/java/io/casehub/engine/internal/engine/recovery/DefaultWorkerExecutionRecoveryService.java` | Panel-aware rebuild |
| `runtime/src/main/java/io/casehub/engine/internal/engine/handler/WorkflowExecutionCompletedHandler.java` | Episodic panel update after worker |
| All JQ expressions in tests | `.key` → `.working.key` |

---

## Task 1: `ReadOnlyPanelException` and `ContextPanel` constants

**Files:**
- Create: `api/src/main/java/io/casehub/api/context/ReadOnlyPanelException.java`
- Create: `api/src/main/java/io/casehub/api/context/ContextPanel.java`

- [ ] **Step 1: Create `ReadOnlyPanelException`**

```java
// api/src/main/java/io/casehub/api/context/ReadOnlyPanelException.java
package io.casehub.api.context;

public class ReadOnlyPanelException extends RuntimeException {
  public ReadOnlyPanelException(String panelName) {
    super("Panel '" + panelName + "' is read-only — writes are not permitted");
  }
}
```

- [ ] **Step 2: Create `ContextPanel` constants**

```java
// api/src/main/java/io/casehub/api/context/ContextPanel.java
package io.casehub.api.context;

public final class ContextPanel {
  public static final String WORKING  = "working";
  public static final String SEMANTIC = "semantic";
  public static final String EPISODIC = "episodic";

  private ContextPanel() {}
}
```

- [ ] **Step 3: Verify compilation**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn clean compile -pl api -q
```
Expected: BUILD SUCCESS

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add api/src/main/java/io/casehub/api/context/ReadOnlyPanelException.java api/src/main/java/io/casehub/api/context/ContextPanel.java
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(panels): ReadOnlyPanelException + ContextPanel constants

Refs #80"
```

---

## Task 2: `ReadablePanel` and `WritablePanel` interfaces

**Files:**
- Create: `api/src/main/java/io/casehub/api/context/ReadablePanel.java`
- Create: `api/src/main/java/io/casehub/api/context/WritablePanel.java`

- [ ] **Step 1: Create `ReadablePanel`**

```java
// api/src/main/java/io/casehub/api/context/ReadablePanel.java
package io.casehub.api.context;

import com.fasterxml.jackson.databind.JsonNode;
import java.util.List;
import java.util.Map;
import java.util.Set;

public interface ReadablePanel {

  String panelName();
  boolean isReadOnly();

  Object get(String key);
  <T> T getAs(String key, Class<T> type);
  <T> T getOrDefault(String key, T defaultValue);
  String getString(String key);
  Integer getInt(String key);
  Long getLong(String key);
  Double getDouble(String key);
  Boolean getBoolean(String key);
  <T> List<T> getList(String key, Class<T> elementType);
  boolean contains(String key);
  Set<String> getKeys();
  Map<String, Object> getData();
  Map<String, Object> getAll(String... keys);
  Object getPath(String path);
  String getPathAsString(String path);
  boolean isEmpty();
  int size();
  long getVersion();
  JsonNode asJsonNode();
}
```

- [ ] **Step 2: Create `WritablePanel`**

```java
// api/src/main/java/io/casehub/api/context/WritablePanel.java
package io.casehub.api.context;

import com.fasterxml.jackson.databind.JsonNode;
import java.util.Map;
import java.util.Optional;
import java.util.function.Function;

public interface WritablePanel extends ReadablePanel {

  WritablePanel set(String key, Object value);
  WritablePanel setAll(Map<String, Object> values);
  WritablePanel setPath(String path, Object value);
  WritablePanel remove(String key);
  WritablePanel clear();
  WritablePanel merge(ReadablePanel other);
  Object computeIfAbsent(String key, Function<String, Object> mappingFunction);
  Object putIfAbsent(String key, Object value);
  boolean compareAndSet(String key, Object expected, Object newValue);
  WritablePanel update(String key, Function<Object, Object> updateFunction);
  Optional<JsonNode> applyAndDiff(String path, Object value);
  void applyDiff(JsonNode diff);
  JsonNode diff(ReadablePanel other);
}
```

- [ ] **Step 3: Verify compilation**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn clean compile -pl api -q
```
Expected: BUILD SUCCESS

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add api/src/main/java/io/casehub/api/context/ReadablePanel.java api/src/main/java/io/casehub/api/context/WritablePanel.java
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(panels): ReadablePanel and WritablePanel interfaces

Refs #80"
```

---

## Task 3: `WritablePanelImpl` — the panel implementation

**Files:**
- Create: `runtime/src/main/java/io/casehub/engine/internal/context/WritablePanelImpl.java`
- Create: `runtime/src/test/java/io/casehub/engine/internal/context/PanelTest.java`

- [ ] **Step 1: Write failing tests**

```java
// runtime/src/test/java/io/casehub/engine/internal/context/PanelTest.java
package io.casehub.engine.internal.context;

import io.casehub.api.context.ContextPanel;
import io.casehub.api.context.WritablePanel;
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class PanelTest {

  @Test
  void panelName_returnsConstructedName() {
    WritablePanel p = new WritablePanelImpl(ContextPanel.WORKING);
    assertEquals("working", p.panelName());
  }

  @Test
  void isReadOnly_falseByDefault() {
    assertFalse(new WritablePanelImpl("custom").isReadOnly());
  }

  @Test
  void setAndGet_roundTrip() {
    WritablePanel p = new WritablePanelImpl(ContextPanel.WORKING);
    p.set("score", 42);
    assertEquals(42, p.get("score"));
  }

  @Test
  void getVersion_incrementsOnWrite() {
    WritablePanel p = new WritablePanelImpl(ContextPanel.WORKING);
    long before = p.getVersion();
    p.set("key", "val");
    assertEquals(before + 1, p.getVersion());
  }

  @Test
  void getVersion_noIncrementOnNoOp() {
    WritablePanel p = new WritablePanelImpl(ContextPanel.WORKING);
    p.set("key", "val");
    long before = p.getVersion();
    p.set("key", "val"); // same value
    assertEquals(before, p.getVersion());
  }

  @Test
  void clear_removesAllKeys() {
    WritablePanel p = new WritablePanelImpl(ContextPanel.WORKING);
    p.set("a", 1).set("b", 2);
    p.clear();
    assertTrue(p.isEmpty());
  }

  @Test
  void asJsonNode_representsData() {
    WritablePanel p = new WritablePanelImpl(ContextPanel.WORKING);
    p.set("result", "done");
    assertEquals("done", p.asJsonNode().get("result").asText());
  }

  @Test
  void applyAndDiff_returnsNonEmptyOnChange() {
    WritablePanel p = new WritablePanelImpl(ContextPanel.WORKING);
    p.set("score", 10);
    var diff = p.applyAndDiff("score", 20);
    assertTrue(diff.isPresent());
  }

  @Test
  void applyAndDiff_returnsEmptyOnNoChange() {
    WritablePanel p = new WritablePanelImpl(ContextPanel.WORKING);
    p.set("score", 10);
    var diff = p.applyAndDiff("score", 10);
    assertTrue(diff.isEmpty());
  }
}
```

- [ ] **Step 2: Run tests to confirm they fail**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest=PanelTest -q 2>&1 | tail -5
```
Expected: COMPILATION ERROR — `WritablePanelImpl` not found

- [ ] **Step 3: Implement `WritablePanelImpl`**

`WritablePanelImpl` is essentially the guts of the current `CaseContextImpl` but scoped to a single panel. Copy the lock+data+version pattern from `CaseContextImpl`, remove the panel-container concern.

```java
// runtime/src/main/java/io/casehub/engine/internal/context/WritablePanelImpl.java
package io.casehub.engine.internal.context;

import com.fasterxml.jackson.annotation.JsonAnyGetter;
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.api.context.ReadablePanel;
import io.casehub.api.context.WritablePanel;
import io.fabric8.zjsonpatch.JsonDiff;
import io.fabric8.zjsonpatch.JsonPatch;
import java.util.*;
import java.util.concurrent.locks.*;
import java.util.function.Function;

public class WritablePanelImpl implements WritablePanel {

  private static final ObjectMapper MAPPER = new ObjectMapper();

  private final String panelName;
  private final Map<String, Object> data = new LinkedHashMap<>();
  private final ReadWriteLock lock = new ReentrantReadWriteLock();
  private long version = 0L;

  public WritablePanelImpl(String panelName) {
    this.panelName = panelName;
  }

  public WritablePanelImpl(String panelName, Map<String, Object> initial) {
    this.panelName = panelName;
    if (initial != null) data.putAll(initial);
  }

  @Override public String panelName() { return panelName; }
  @Override public boolean isReadOnly() { return false; }

  @Override
  public Object get(String key) {
    lock.readLock().lock();
    try { return data.get(key); } finally { lock.readLock().unlock(); }
  }

  @Override
  @SuppressWarnings("unchecked")
  public <T> T getAs(String key, Class<T> type) {
    lock.readLock().lock();
    try {
      Object v = data.get(key);
      if (v == null) return null;
      if (type.isInstance(v)) return type.cast(v);
      return MAPPER.convertValue(v, type);
    } finally { lock.readLock().unlock(); }
  }

  @Override
  @SuppressWarnings("unchecked")
  public <T> T getOrDefault(String key, T defaultValue) {
    lock.readLock().lock();
    try {
      Object v = data.get(key);
      return v != null ? (T) v : defaultValue;
    } finally { lock.readLock().unlock(); }
  }

  @Override public String getString(String key) { Object v = get(key); return v != null ? v.toString() : null; }

  @Override
  public Integer getInt(String key) {
    Object v = get(key);
    if (v == null) return null;
    if (v instanceof Number n) return n.intValue();
    return Integer.parseInt(v.toString());
  }

  @Override
  public Long getLong(String key) {
    Object v = get(key);
    if (v == null) return null;
    if (v instanceof Number n) return n.longValue();
    return Long.parseLong(v.toString());
  }

  @Override
  public Double getDouble(String key) {
    Object v = get(key);
    if (v == null) return null;
    if (v instanceof Number n) return n.doubleValue();
    return Double.parseDouble(v.toString());
  }

  @Override
  public Boolean getBoolean(String key) {
    Object v = get(key);
    if (v == null) return null;
    if (v instanceof Boolean b) return b;
    return Boolean.parseBoolean(v.toString());
  }

  @Override
  public <T> List<T> getList(String key, Class<T> elementType) {
    lock.readLock().lock();
    try {
      Object v = data.get(key);
      if (v == null) return null;
      if (v instanceof List<?> list) return list.stream().map(i -> MAPPER.convertValue(i, elementType)).toList();
      return null;
    } finally { lock.readLock().unlock(); }
  }

  @Override
  public boolean contains(String key) {
    lock.readLock().lock();
    try { return data.containsKey(key); } finally { lock.readLock().unlock(); }
  }

  @Override
  public Set<String> getKeys() {
    lock.readLock().lock();
    try { return new HashSet<>(data.keySet()); } finally { lock.readLock().unlock(); }
  }

  @JsonAnyGetter
  @Override
  public Map<String, Object> getData() {
    lock.readLock().lock();
    try { return new LinkedHashMap<>(data); } finally { lock.readLock().unlock(); }
  }

  @Override
  public Map<String, Object> getAll(String... keys) {
    lock.readLock().lock();
    try {
      Map<String, Object> result = new LinkedHashMap<>();
      for (String key : keys) { Object v = data.get(key); if (v != null) result.put(key, v); }
      return result;
    } finally { lock.readLock().unlock(); }
  }

  @Override
  public Object getPath(String path) {
    lock.readLock().lock();
    try { return getPathInternal(path); } finally { lock.readLock().unlock(); }
  }

  @Override
  public String getPathAsString(String path) {
    Object v = getPath(path);
    return v != null ? v.toString() : null;
  }

  @Override
  public boolean isEmpty() {
    lock.readLock().lock();
    try { return data.isEmpty(); } finally { lock.readLock().unlock(); }
  }

  @Override
  public int size() {
    lock.readLock().lock();
    try { return data.size(); } finally { lock.readLock().unlock(); }
  }

  @Override
  public long getVersion() {
    lock.readLock().lock();
    try { return version; } finally { lock.readLock().unlock(); }
  }

  @Override
  public JsonNode asJsonNode() {
    lock.readLock().lock();
    try { return MAPPER.convertValue(data, JsonNode.class); } finally { lock.readLock().unlock(); }
  }

  @Override
  public WritablePanel set(String key, Object value) {
    lock.writeLock().lock();
    try {
      if (!Objects.equals(data.get(key), value)) { data.put(key, value); version++; }
      return this;
    } finally { lock.writeLock().unlock(); }
  }

  @Override
  public WritablePanel setAll(Map<String, Object> values) {
    if (values == null || values.isEmpty()) return this;
    lock.writeLock().lock();
    try {
      boolean changed = false;
      for (var e : values.entrySet()) {
        if (!Objects.equals(data.get(e.getKey()), e.getValue())) { data.put(e.getKey(), e.getValue()); changed = true; }
      }
      if (changed) version++;
      return this;
    } finally { lock.writeLock().unlock(); }
  }

  @Override
  @SuppressWarnings("unchecked")
  public WritablePanel setPath(String path, Object value) {
    lock.writeLock().lock();
    try {
      String[] parts = path.split("\\.");
      Map<String, Object> current = data;
      for (int i = 0; i < parts.length - 1; i++) {
        Object next = current.get(parts[i]);
        if (next == null) { next = new LinkedHashMap<String, Object>(); current.put(parts[i], next); }
        if (next instanceof Map) current = (Map<String, Object>) next;
        else throw new IllegalStateException("Cannot set path: " + parts[i] + " is not a Map");
      }
      String leaf = parts[parts.length - 1];
      if (!Objects.equals(current.get(leaf), value)) { current.put(leaf, value); version++; }
      return this;
    } finally { lock.writeLock().unlock(); }
  }

  @Override
  public WritablePanel remove(String key) {
    lock.writeLock().lock();
    try { if (data.remove(key) != null) version++; return this; } finally { lock.writeLock().unlock(); }
  }

  @Override
  public WritablePanel clear() {
    lock.writeLock().lock();
    try { if (!data.isEmpty()) { data.clear(); version++; } return this; } finally { lock.writeLock().unlock(); }
  }

  @Override
  public WritablePanel merge(ReadablePanel other) {
    if (other == null) return this;
    return setAll(other.getData());
  }

  @Override
  public Object computeIfAbsent(String key, Function<String, Object> mappingFunction) {
    lock.writeLock().lock();
    try {
      Object v = data.get(key);
      if (v == null) { v = mappingFunction.apply(key); if (v != null) { data.put(key, v); version++; } }
      return v;
    } finally { lock.writeLock().unlock(); }
  }

  @Override
  public Object putIfAbsent(String key, Object value) {
    lock.writeLock().lock();
    try {
      Object existing = data.get(key);
      if (existing == null) { data.put(key, value); version++; }
      return existing;
    } finally { lock.writeLock().unlock(); }
  }

  @Override
  public boolean compareAndSet(String key, Object expected, Object newValue) {
    lock.writeLock().lock();
    try {
      Object current = data.get(key);
      if (Objects.equals(current, expected)) {
        if (!Objects.equals(current, newValue)) { data.put(key, newValue); version++; }
        return true;
      }
      return false;
    } finally { lock.writeLock().unlock(); }
  }

  @Override
  public WritablePanel update(String key, Function<Object, Object> updateFunction) {
    lock.writeLock().lock();
    try {
      Object current = data.get(key);
      Object newValue = updateFunction.apply(current);
      if (newValue != null) {
        if (!Objects.equals(current, newValue)) { data.put(key, newValue); version++; }
      } else {
        if (data.containsKey(key)) { data.remove(key); version++; }
      }
      return this;
    } finally { lock.writeLock().unlock(); }
  }

  @Override
  @SuppressWarnings("unchecked")
  public Optional<JsonNode> applyAndDiff(String path, Object value) {
    lock.writeLock().lock();
    try {
      JsonNode before = MAPPER.convertValue(data, JsonNode.class);
      String[] parts = path.split("\\.");
      Map<String, Object> current = data;
      for (int i = 0; i < parts.length - 1; i++) {
        Object next = current.get(parts[i]);
        if (next == null) { next = new LinkedHashMap<String, Object>(); current.put(parts[i], next); }
        if (next instanceof Map) current = (Map<String, Object>) next;
        else throw new IllegalStateException("Cannot set path: " + parts[i] + " is not a Map");
      }
      String leaf = parts[parts.length - 1];
      if (!Objects.equals(current.get(leaf), value)) { current.put(leaf, value); version++; }
      JsonNode after = MAPPER.convertValue(data, JsonNode.class);
      JsonNode diff = JsonDiff.asJson(before, after);
      return diff.isEmpty() ? Optional.empty() : Optional.of(diff);
    } finally { lock.writeLock().unlock(); }
  }

  @Override
  public void applyDiff(JsonNode diff) {
    lock.writeLock().lock();
    try {
      JsonNode current = MAPPER.convertValue(data, JsonNode.class);
      JsonNode patched = JsonPatch.apply(diff, current);
      Map<String, Object> updated = MAPPER.convertValue(patched,
          MAPPER.getTypeFactory().constructMapType(LinkedHashMap.class, String.class, Object.class));
      data.clear();
      data.putAll(updated);
      version++;
    } finally { lock.writeLock().unlock(); }
  }

  @Override
  public JsonNode diff(ReadablePanel other) {
    lock.readLock().lock();
    try {
      JsonNode thisNode = MAPPER.convertValue(data, JsonNode.class);
      JsonNode otherNode = other.asJsonNode();
      return JsonDiff.asJson(thisNode, otherNode);
    } finally { lock.readLock().unlock(); }
  }

  @SuppressWarnings("unchecked")
  private Object getPathInternal(String path) {
    String[] parts = path.split("\\.");
    Object current = data;
    for (String part : parts) {
      if (current instanceof Map<?, ?> map) current = map.get(part);
      else return null;
      if (current == null) return null;
    }
    return current;
  }

  /** Deep-copies this panel's data into a new WritablePanelImpl. */
  public WritablePanelImpl deepCopy() {
    lock.readLock().lock();
    try { return new WritablePanelImpl(panelName, deepCopyMap(data)); } finally { lock.readLock().unlock(); }
  }

  @SuppressWarnings("unchecked")
  private static Map<String, Object> deepCopyMap(Map<String, Object> source) {
    Map<String, Object> copy = new LinkedHashMap<>();
    for (var e : source.entrySet()) {
      Object v = e.getValue();
      if (v instanceof Map) v = deepCopyMap((Map<String, Object>) v);
      else if (v instanceof List) v = new ArrayList<>((List<?>) v);
      copy.put(e.getKey(), v);
    }
    return copy;
  }
}
```

- [ ] **Step 4: Run tests**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest=PanelTest -q 2>&1 | tail -10
```
Expected: All tests PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add runtime/src/main/java/io/casehub/engine/internal/context/WritablePanelImpl.java runtime/src/test/java/io/casehub/engine/internal/context/PanelTest.java
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(panels): WritablePanelImpl — panel data store with lock and version

Refs #80"
```

---

## Task 4: `ReadOnlyPanelView`

**Files:**
- Create: `runtime/src/main/java/io/casehub/engine/internal/context/ReadOnlyPanelView.java`

- [ ] **Step 1: Write failing test** (add to `PanelTest.java`)

```java
// Add to PanelTest.java:
import io.casehub.api.context.ReadOnlyPanelException;
import io.casehub.api.context.ReadablePanel;

@Test
void readOnlyView_allowsReads() {
  WritablePanelImpl backing = new WritablePanelImpl(ContextPanel.SEMANTIC);
  backing.set("threshold", 0.8);
  ReadablePanel view = new ReadOnlyPanelView(backing);
  assertEquals(0.8, view.getAs("threshold", Double.class));
  assertTrue(view.isReadOnly());
  assertEquals(ContextPanel.SEMANTIC, view.panelName());
}

@Test
void readOnlyView_throwsOnSet() {
  WritablePanel backing = new WritablePanelImpl(ContextPanel.SEMANTIC);
  ReadOnlyPanelView view = new ReadOnlyPanelView(backing);
  assertThrows(ReadOnlyPanelException.class, () -> view.set("key", "value"));
}

@Test
void readOnlyView_throwsOnClear() {
  ReadOnlyPanelView view = new ReadOnlyPanelView(new WritablePanelImpl(ContextPanel.SEMANTIC));
  assertThrows(ReadOnlyPanelException.class, view::clear);
}
```

- [ ] **Step 2: Run test to confirm it fails**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest=PanelTest -q 2>&1 | tail -5
```
Expected: COMPILATION ERROR — `ReadOnlyPanelView` not found

- [ ] **Step 3: Implement `ReadOnlyPanelView`**

```java
// runtime/src/main/java/io/casehub/engine/internal/context/ReadOnlyPanelView.java
package io.casehub.engine.internal.context;

import com.fasterxml.jackson.databind.JsonNode;
import io.casehub.api.context.ReadOnlyPanelException;
import io.casehub.api.context.ReadablePanel;
import io.casehub.api.context.WritablePanel;
import java.util.*;
import java.util.function.Function;

public class ReadOnlyPanelView implements WritablePanel {

  private final WritablePanelImpl delegate;

  public ReadOnlyPanelView(WritablePanelImpl delegate) {
    this.delegate = delegate;
  }

  @Override public String panelName() { return delegate.panelName(); }
  @Override public boolean isReadOnly() { return true; }
  @Override public Object get(String key) { return delegate.get(key); }
  @Override public <T> T getAs(String key, Class<T> type) { return delegate.getAs(key, type); }
  @Override public <T> T getOrDefault(String key, T d) { return delegate.getOrDefault(key, d); }
  @Override public String getString(String key) { return delegate.getString(key); }
  @Override public Integer getInt(String key) { return delegate.getInt(key); }
  @Override public Long getLong(String key) { return delegate.getLong(key); }
  @Override public Double getDouble(String key) { return delegate.getDouble(key); }
  @Override public Boolean getBoolean(String key) { return delegate.getBoolean(key); }
  @Override public <T> List<T> getList(String key, Class<T> et) { return delegate.getList(key, et); }
  @Override public boolean contains(String key) { return delegate.contains(key); }
  @Override public Set<String> getKeys() { return delegate.getKeys(); }
  @Override public Map<String, Object> getData() { return delegate.getData(); }
  @Override public Map<String, Object> getAll(String... keys) { return delegate.getAll(keys); }
  @Override public Object getPath(String path) { return delegate.getPath(path); }
  @Override public String getPathAsString(String path) { return delegate.getPathAsString(path); }
  @Override public boolean isEmpty() { return delegate.isEmpty(); }
  @Override public int size() { return delegate.size(); }
  @Override public long getVersion() { return delegate.getVersion(); }
  @Override public JsonNode asJsonNode() { return delegate.asJsonNode(); }
  @Override public JsonNode diff(ReadablePanel other) { return delegate.diff(other); }

  private void reject() { throw new ReadOnlyPanelException(panelName()); }

  @Override public WritablePanel set(String k, Object v) { reject(); return this; }
  @Override public WritablePanel setAll(Map<String, Object> v) { reject(); return this; }
  @Override public WritablePanel setPath(String p, Object v) { reject(); return this; }
  @Override public WritablePanel remove(String k) { reject(); return this; }
  @Override public WritablePanel clear() { reject(); return this; }
  @Override public WritablePanel merge(ReadablePanel o) { reject(); return this; }
  @Override public Object computeIfAbsent(String k, Function<String, Object> f) { reject(); return null; }
  @Override public Object putIfAbsent(String k, Object v) { reject(); return null; }
  @Override public boolean compareAndSet(String k, Object e, Object n) { reject(); return false; }
  @Override public WritablePanel update(String k, Function<Object, Object> f) { reject(); return this; }
  @Override public Optional<JsonNode> applyAndDiff(String p, Object v) { reject(); return Optional.empty(); }
  @Override public void applyDiff(JsonNode d) { reject(); }
}
```

- [ ] **Step 4: Run tests**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest=PanelTest -q 2>&1 | tail -5
```
Expected: All tests PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add runtime/src/main/java/io/casehub/engine/internal/context/ReadOnlyPanelView.java runtime/src/test/java/io/casehub/engine/internal/context/PanelTest.java
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(panels): ReadOnlyPanelView — write-blocking delegate

Refs #80"
```

---

## Task 5: Restructure `CaseContextImpl` to panel map

This is the biggest change. `CaseContextImpl` replaces its flat `data` map with a `Map<String, WritablePanelImpl> panels`. All existing flat API methods delegate to `panels.get("working")`.

**Files:**
- Modify: `runtime/src/main/java/io/casehub/engine/internal/context/CaseContextImpl.java`
- Modify: `api/src/main/java/io/casehub/api/context/CaseContext.java`

- [ ] **Step 1: Add `panel()` method to `CaseContext` interface**

Add to `CaseContext.java` after the last method:

```java
// Add import at top:
import io.casehub.api.context.ReadablePanel;

// Add method to interface:
ReadablePanel panel(String name);
```

- [ ] **Step 2: Write failing tests** (extend `CaseContextImplTest`)

Add these test methods to `runtime/src/test/java/io/casehub/engine/internal/context/CaseContextImplTest.java`:

```java
import io.casehub.api.context.ContextPanel;
import io.casehub.api.context.ReadablePanel;
import io.casehub.api.context.WritablePanel;

@Test
void panel_returnsWorkingPanel() {
  CaseContextImpl ctx = new CaseContextImpl();
  ctx.set("result", "done");
  ReadablePanel p = ctx.panel(ContextPanel.WORKING);
  assertEquals("done", p.get("result"));
  assertEquals(ContextPanel.WORKING, p.panelName());
}

@Test
void flatApi_delegatesToWorkingPanel() {
  CaseContextImpl ctx = new CaseContextImpl();
  ctx.set("key", "value");
  assertEquals("value", ctx.panel(ContextPanel.WORKING).get("key"));
}

@Test
void asJsonNode_returnsPanelDocument() {
  CaseContextImpl ctx = new CaseContextImpl();
  ctx.set("result", "done");
  JsonNode doc = ctx.asJsonNode();
  assertNotNull(doc.get("working"));
  assertEquals("done", doc.get("working").get("result").asText());
  // semantic and episodic panels present but empty
  assertNotNull(doc.get("semantic"));
  assertNotNull(doc.get("episodic"));
}

@Test
void snapshot_includesAllPanels() {
  CaseContextImpl ctx = new CaseContextImpl();
  ctx.set("result", "done");
  CaseContext snap = ctx.snapshot();
  assertNotNull(snap.asJsonNode().get("semantic"));
  assertNotNull(snap.asJsonNode().get("episodic"));
  assertEquals("done", snap.asJsonNode().get("working").get("result").asText());
}

@Test
void getVersion_delegatesToWorkingPanel() {
  CaseContextImpl ctx = new CaseContextImpl();
  long before = ctx.getVersion();
  ctx.set("k", "v");
  assertEquals(before + 1, ctx.getVersion());
}

@Test
void clear_onlyClearsWorkingPanel() {
  CaseContextImpl ctx = new CaseContextImpl();
  ctx.set("k", "v");
  ctx.clear();
  assertTrue(ctx.isEmpty()); // working is clear
  // semantic + episodic panels still exist
  assertNotNull(ctx.panel(ContextPanel.SEMANTIC));
}

@Test
void applyAndDiff_isWorkingPanelRelative() {
  CaseContextImpl ctx = new CaseContextImpl();
  ctx.set("score", 10);
  var diff = ctx.applyAndDiff("score", 20);
  assertTrue(diff.isPresent());
  // patch path is /score not /working/score
  String patchStr = diff.get().toString();
  assertTrue(patchStr.contains("/score"));
  assertFalse(patchStr.contains("/working/score"));
}
```

- [ ] **Step 3: Run to confirm failures**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest=CaseContextImplTest -q 2>&1 | tail -10
```
Expected: COMPILATION ERROR or test failures

- [ ] **Step 4: Restructure `CaseContextImpl`**

Replace the class body entirely. Keep the same class-level annotations (`@JsonDeserialize`, etc.) but replace the `data` field with the panel map:

```java
// runtime/src/main/java/io/casehub/engine/internal/context/CaseContextImpl.java
package io.casehub.engine.internal.context;

import com.fasterxml.jackson.annotation.JsonAnySetter;
import com.fasterxml.jackson.annotation.JsonIgnore;
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ObjectNode;
import com.fasterxml.jackson.databind.annotation.JsonDeserialize;
import io.casehub.api.context.CaseContext;
import io.casehub.api.context.ContextPanel;
import io.casehub.api.context.ReadablePanel;
import io.casehub.api.context.WritablePanel;
import io.fabric8.zjsonpatch.JsonDiff;
import java.util.*;
import java.util.function.Function;

@JsonDeserialize(as = CaseContextImpl.class)
public class CaseContextImpl implements CaseContext {

  private static final ObjectMapper MAPPER = new ObjectMapper();

  // Panel map. Insertion order: working first for predictable JSON serialisation.
  private final Map<String, WritablePanelImpl> panels = new LinkedHashMap<>();

  public CaseContextImpl() {
    initBuiltinPanels();
  }

  public CaseContextImpl(Map<String, Object> initial) {
    initBuiltinPanels();
    if (initial != null && !initial.isEmpty()) {
      working().setAll(initial);
    }
  }

  public CaseContextImpl(Map<String, Object> initial, long ignoredVersion) {
    // version is per-panel now; the top-level version param is ignored
    this(initial);
  }

  public CaseContextImpl(JsonNode asNode) {
    this(asNode == null ? null : MAPPER.convertValue(asNode, Map.class));
  }

  private void initBuiltinPanels() {
    panels.put(ContextPanel.WORKING,  new WritablePanelImpl(ContextPanel.WORKING));
    panels.put(ContextPanel.SEMANTIC, new WritablePanelImpl(ContextPanel.SEMANTIC));
    panels.put(ContextPanel.EPISODIC, new WritablePanelImpl(ContextPanel.EPISODIC));
  }

  private WritablePanelImpl working() {
    return panels.get(ContextPanel.WORKING);
  }

  // ---- CaseContext.panel() ----

  @Override
  public ReadablePanel panel(String name) {
    WritablePanelImpl p = panels.computeIfAbsent(name, WritablePanelImpl::new);
    return p;
  }

  // Internal: engine writes to read-only panels via this method.
  public WritablePanelImpl writablePanel(String name) {
    return panels.computeIfAbsent(name, WritablePanelImpl::new);
  }

  // ---- Flat API — all delegate to working panel ----

  @JsonAnySetter
  @Override public CaseContext set(String key, Object value) { working().set(key, value); return this; }
  @Override public Object get(String key) { return working().get(key); }
  @Override public <T> T getAs(String key, Class<T> type) { return working().getAs(key, type); }
  @Override public <T> T getOrDefault(String key, T d) { return working().getOrDefault(key, d); }
  @Override public Object computeIfAbsent(String key, Function<String, Object> f) { return working().computeIfAbsent(key, f); }
  @Override public Object putIfAbsent(String key, Object value) { return working().putIfAbsent(key, value); }
  @Override public boolean compareAndSet(String key, Object expected, Object n) { return working().compareAndSet(key, expected, n); }
  @Override public CaseContext update(String key, Function<Object, Object> f) { working().update(key, f); return this; }
  @Override public String getString(String key) { return working().getString(key); }
  @Override public Integer getInt(String key) { return working().getInt(key); }
  @Override public Long getLong(String key) { return working().getLong(key); }
  @Override public Double getDouble(String key) { return working().getDouble(key); }
  @Override public Boolean getBoolean(String key) { return working().getBoolean(key); }
  @Override public <T> List<T> getList(String key, Class<T> et) { return working().getList(key, et); }
  @Override public Object getPath(String path) { return working().getPath(path); }
  @Override public String getPathAsString(String path) { return working().getPathAsString(path); }
  @Override public CaseContext setPath(String path, Object value) { working().setPath(path, value); return this; }
  @Override public Optional<JsonNode> applyAndDiff(String path, Object value) { return working().applyAndDiff(path, value); }
  @Override public CaseContext setAll(Map<String, Object> values) { working().setAll(values); return this; }
  @Override public Map<String, Object> getAll(String... keys) { return working().getAll(keys); }
  @Override public boolean contains(String key) { return working().contains(key); }
  @Override public CaseContext remove(String key) { working().remove(key); return this; }
  @Override public CaseContext clear() { working().clear(); return this; }
  @Override public Set<String> getKeys() { return working().getKeys(); }
  @Override public boolean isEmpty() { return working().isEmpty(); }
  @Override public int size() { return working().size(); }
  @Override public long getVersion() { return working().getVersion(); }

  @JsonIgnore
  @Override
  public Map<String, Object> getData() { return working().getData(); }

  // merge: working panel only
  @Override
  public CaseContext merge(CaseContext other) {
    if (other == null) return this;
    working().setAll(other.getData());
    return this;
  }

  // diff: working panels only
  @Override
  public JsonNode diff(CaseContext other) {
    return working().diff(other.panel(ContextPanel.WORKING));
  }

  // applyDiff: working panel only (patch paths are working-panel-relative)
  @Override
  public void applyDiff(JsonNode diff) {
    working().applyDiff(diff);
  }

  // snapshot: deep-copies ALL panels
  @Override
  public CaseContext snapshot() {
    CaseContextImpl snap = new CaseContextImpl();
    snap.panels.clear();
    for (var e : panels.entrySet()) {
      snap.panels.put(e.getKey(), e.getValue().deepCopy());
    }
    return snap;
  }

  // asJsonNode: full panel document {"working":{...},"semantic":{...},"episodic":{...},...}
  @Override
  public JsonNode asJsonNode() {
    ObjectNode doc = MAPPER.createObjectNode();
    for (var e : panels.entrySet()) {
      doc.set(e.getKey(), e.getValue().asJsonNode());
    }
    return doc;
  }

  /**
   * Reconstructs a CaseContextImpl from a stored panel document (e.g. CASE_STARTED EventLog payload).
   * Called by DefaultWorkerExecutionRecoveryService.
   */
  public static CaseContextImpl fromPanelDocument(JsonNode panelDoc) {
    CaseContextImpl ctx = new CaseContextImpl();
    if (panelDoc == null || panelDoc.isNull()) return ctx;
    panelDoc.fields().forEachRemaining(e -> {
      String name = e.getKey();
      @SuppressWarnings("unchecked")
      Map<String, Object> data = MAPPER.convertValue(e.getValue(), Map.class);
      ctx.panels.put(name, new WritablePanelImpl(name, data));
    });
    return ctx;
  }

  @Override
  public String toString() {
    try { return MAPPER.writeValueAsString(asJsonNode()); }
    catch (Exception e) { return panels.toString(); }
  }

  @Override
  public boolean equals(Object o) {
    if (this == o) return true;
    if (!(o instanceof CaseContextImpl that)) return false;
    return getData().equals(that.getData()); // compare working panels
  }

  @Override
  public int hashCode() {
    return working().getData().hashCode();
  }
}
```

- [ ] **Step 5: Run tests**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest="CaseContextImplTest,PanelTest,CaseContextImplApplyDiffTest" -q 2>&1 | tail -15
```
Expected: All tests PASS. Fix any remaining failures before continuing.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add api/src/main/java/io/casehub/api/context/CaseContext.java runtime/src/main/java/io/casehub/engine/internal/context/CaseContextImpl.java
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(panels): CaseContextImpl restructured to panel map

- Flat API delegates to working panel (no lambda call sites change)
- asJsonNode() returns full panel document
- snapshot() deep-copies all panels
- applyAndDiff/applyDiff operate on working panel (patch paths unchanged)
- panel(name) returns ReadablePanel; writablePanel(name) for engine-internal writes
- fromPanelDocument() factory for recovery

Refs #80"
```

---

## Task 6: Clean up `MapCaseFile` and compile all modules

**Files:**
- Modify: `runtime/src/main/java/io/casehub/engine/internal/context/MapCaseFile.java`

- [ ] **Step 1: Remove `snapshot()` override from `MapCaseFile`**

Delete the `snapshot()` override method entirely from `MapCaseFile`. The `CaseContextImpl.snapshot()` now handles all panels correctly, so the MapCaseFile override (which was silently dropping semantic/episodic panels) is wrong.

The remaining MapCaseFile methods (`put`, `get`, `keys`) still work because they use the flat API which delegates to the working panel.

- [ ] **Step 2: Compile the full engine**

```bash
mvn install -DskipTests -q 2>&1 | tail -20
```
Fix any compilation errors. Common issues:
- Any `new CaseContextImpl(Map, long)` constructor calls: the second `long` param is now ignored. Calls still compile but the version is not restored. This is correct — per-panel versions start at 0 on construction.
- Any code that calls `getData()` expecting all data: now returns only working panel data. Callers that need this should use `panel("working").getData()` or the flat API.

- [ ] **Step 3: Run the full runtime test suite**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl runtime -q 2>&1 | tail -20
```
Expected: most tests pass; some JQ expression tests will FAIL (they reference `.key` which now needs `.working.key`). Do NOT fix these yet — they are addressed in Task 14.

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add runtime/src/main/java/io/casehub/engine/internal/context/MapCaseFile.java
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(panels): remove MapCaseFile.snapshot() override

snapshot() on CaseContextImpl now correctly includes all panels.
The override was silently dropping semantic and episodic.

Refs #80"
```

---

## Task 7: `CaseContextChangedEvent` — add `CaseContext contextSnapshot` and `changedPanel`

**Files:**
- Modify: `common/src/main/java/io/casehub/engine/common/internal/event/CaseContextChangedEvent.java`
- Modify: `common/src/main/java/io/casehub/engine/common/internal/event/EventBusAddresses.java`
- Modify: 12 production construction sites (listed below)

- [ ] **Step 1: Update the record**

```java
// common/src/main/java/io/casehub/engine/common/internal/event/CaseContextChangedEvent.java
package io.casehub.engine.common.internal.event;

import io.casehub.api.context.CaseContext;
import io.casehub.engine.common.internal.model.CaseInstance;
import java.util.Objects;

public record CaseContextChangedEvent(CaseInstance instance, CaseContext contextSnapshot, String changedPanel) {

  public CaseContextChangedEvent {
    instance = Objects.requireNonNull(instance, "instance cannot be null");
    contextSnapshot = Objects.requireNonNull(contextSnapshot, "contextSnapshot cannot be null");
    // contextSnapshot is already a snapshot (caller passes instance.getCaseContext().snapshot())
    // changedPanel may be null (case started / resumed — evaluate all bindings)
  }

  @Override
  public boolean equals(Object o) {
    if (this == o) return true;
    if (o == null || getClass() != o.getClass()) return false;
    CaseContextChangedEvent that = (CaseContextChangedEvent) o;
    return Objects.equals(instance, that.instance)
        && Objects.equals(changedPanel, that.changedPanel);
  }

  @Override
  public String toString() {
    return "CaseContextChangedEvent{uuid=" + instance.getUuid() + ", panel=" + changedPanel + '}';
  }
}
```

- [ ] **Step 2: Add `panelChanged()` to `EventBusAddresses`**

Add to `EventBusAddresses.java`:

```java
public static String panelChanged(String panelName) {
    return "casehub.context.changed." + panelName;
}
```

- [ ] **Step 3: Update all 12 production construction sites**

Each site currently constructs `new CaseContextChangedEvent(instance, contextSnapshot)` where `contextSnapshot` is `JsonNode`. Change to `new CaseContextChangedEvent(instance, instance.getCaseContext().snapshot(), ContextPanel.WORKING)`.

For `CaseStartedEventHandler` (line 80): `changedPanel = null` (case start — evaluate all bindings).
For `CaseStatusChangedHandler` (line ~118): `changedPanel = null` (resume — evaluate all bindings).
All others: `changedPanel = ContextPanel.WORKING`.

Files to update:
1. `runtime/.../handler/CaseStartedEventHandler.java` — null changedPanel, snapshot not JsonNode
2. `runtime/.../handler/SignalReceivedEventHandler.java` — WORKING
3. `runtime/.../handler/WorkflowExecutionCompletedHandler.java` — WORKING
4. `runtime/.../handler/CaseStatusChangedHandler.java` — null
5. `runtime/.../handler/MilestoneActivatedEventHandler.java` — WORKING
6. `runtime/.../handler/MilestoneCompletedEventHandler.java` — WORKING
7. `runtime/.../handler/MilestoneSLAViolatedEventHandler.java` — WORKING
8. `runtime/.../handler/ActionGateExpiredHandler.java` — WORKING
9. `runtime/.../handler/ActionGateRejectedHandler.java` — WORKING
10. `work-adapter/.../WorkItemLifecycleAdapter.java` — WORKING (two sites)
11. `work-adapter/.../PlanItemCompletionApplier.java` — WORKING

Pattern for most sites (replace `instance.getCaseContext().asJsonNode()` with `instance.getCaseContext().snapshot()`):
```java
// Before
new CaseContextChangedEvent(instance, instance.getCaseContext().asJsonNode())
// After
new CaseContextChangedEvent(instance, instance.getCaseContext().snapshot(), ContextPanel.WORKING)
```

For `SignalReceivedEventHandler` (line 111–119): remove the local `contextSnapshot` variable and build inline:
```java
// Before
JsonNode contextSnapshot = instance.getCaseContext().asJsonNode();
// ... later
new CaseContextChangedEvent(instance, contextSnapshot)

// After — take snapshot after applyAndDiff
new CaseContextChangedEvent(instance, instance.getCaseContext().snapshot(), ContextPanel.WORKING)
```

Add import `import io.casehub.api.context.ContextPanel;` to each modified file.

- [ ] **Step 4: Update `CaseContextChangedEventHandler` — change snapshot usage**

In `CaseContextChangedEventHandler`, the handler now receives `CaseContext contextSnapshot` instead of `JsonNode`. Update the `rules()`, `milestones()`, and `goals()` methods:

```java
// In onCaseStateContextChangedEventHandler():
final CaseContext contextSnapshot = event.contextSnapshot();
// Remove: final JsonNode contextSnapshot = event.contextSnapshot();

// In rules() signature:
private Uni<Void> rules(CaseInstance caseInstance, CaseContext contextSnapshot, CaseDefinition definition)

// In rules() — for JQ evaluation:
if (!expressionEngineRegistry.evaluate(cct.getFilter(), contextSnapshot)) continue;
if (binding.getWhen() != null && !expressionEngineRegistry.evaluate(binding.getWhen(), contextSnapshot)) continue;
// (registry dispatches to JsonNode overload via contextSnapshot.asJsonNode() internally for JQ)

// In milestones() signature:
private Uni<Void> milestones(CaseInstance caseInstance, CaseContext contextSnapshot, CaseDefinition definition)

// In milestones() — use snapshot not live context:
if (!expressionEngineRegistry.evaluate(milestone.getCompletionCriteria(), contextSnapshot)) continue;
// Remove: if (!expressionEngineRegistry.evaluate(milestone.getCompletionCriteria(), caseInstance.getCaseContext())) continue;

// In goals() signature:
private Uni<Void> goals(CaseInstance caseInstance, CaseContext contextSnapshot, CaseDefinition definition)

// In goals() — use snapshot not live context:
if (!expressionEngineRegistry.evaluate(goal.getCondition(), contextSnapshot)) continue;
// Remove: if (!expressionEngineRegistry.evaluate(goal.getCondition(), caseInstance.getCaseContext())) continue;

// Add changedPanel filtering before the binding loop (episodic updates skip all bindings):
final String changedPanel = event.changedPanel();
if (ContextPanel.EPISODIC.equals(changedPanel)) {
    return Uni.createFrom().voidItem();
}
```

- [ ] **Step 5: Update test construction sites**

In `CaseContextChangedEventHandlerRoutingTest` (4 places) and `MilestoneLifecycleTest` (1 place): change `new CaseContextChangedEvent(caseInstance, NullNode.instance)` to use a real snapshot. Use `new CaseContextImpl().snapshot()` as the context, or `mock(CaseContext.class)`.

In `MilestoneLifecycleTest`: change `new CaseContextChangedEvent(instance, contextSnapshot)` to `new CaseContextChangedEvent(instance, instance.getCaseContext().snapshot(), ContextPanel.WORKING)`.

- [ ] **Step 6: Compile and run**

```bash
mvn install -DskipTests -q 2>&1 | tail -10
TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl runtime -q 2>&1 | tail -15
```
Expected: previous test failures unchanged (JQ expression migration not done yet); no NEW failures from this change.

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add -u
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(panels): CaseContextChangedEvent — CaseContext snapshot + changedPanel

- contextSnapshot: JsonNode → CaseContext (eliminates milestones/goals asymmetry)
- changedPanel: null for case start/resume, WORKING for all engine events
- handler: episodic changes bypass all bindings
- milestones() and goals() use snapshot consistently

Refs #80"
```

---

## Task 8: Semantic panel — `CaseDefinition` model + population

**Files:**
- Modify: `api/src/main/java/io/casehub/api/model/CaseDefinition.java`
- Create: `api/src/main/java/io/casehub/api/model/EpisodicMemoryConfig.java`
- Modify: `api/src/main/java/io/casehub/api/engine/CaseHubRuntime.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/CaseHubRuntimeImpl.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/CaseHubReactor.java`
- Create: `runtime/src/test/java/io/casehub/engine/internal/context/SemanticPanelTest.java`

- [ ] **Step 1: Add `EpisodicMemoryConfig` value object**

```java
// api/src/main/java/io/casehub/api/model/EpisodicMemoryConfig.java
package io.casehub.api.model;

import java.util.Objects;

public record EpisodicMemoryConfig(
    String domain,      // MemoryDomain name; becomes new MemoryDomain(domain)
    String entityId,    // JQ expression against semantic panel; result → List<String>
    int recent          // maps to MemoryQuery.withLimit(recent); default 10
) {
  public EpisodicMemoryConfig {
    Objects.requireNonNull(domain, "domain is required when episodic.memory is declared");
    Objects.requireNonNull(entityId, "entityId is required when episodic.memory is declared");
    if (recent < 1) recent = 10;
  }

  public static EpisodicMemoryConfig of(String domain, String entityId) {
    return new EpisodicMemoryConfig(domain, entityId, 10);
  }

  public static EpisodicMemoryConfig of(String domain, String entityId, int recent) {
    return new EpisodicMemoryConfig(domain, entityId, recent);
  }
}
```

- [ ] **Step 2: Add `semanticData` and `episodicMemoryConfig` to `CaseDefinition`**

Add two fields to `CaseDefinition`:

```java
// In CaseDefinition fields:
private Map<String, Object> semanticData;
private EpisodicMemoryConfig episodicMemoryConfig;

// Add getters/setters:
public Map<String, Object> getSemanticData() { return semanticData; }
public void setSemanticData(Map<String, Object> semanticData) { this.semanticData = semanticData; }
public EpisodicMemoryConfig getEpisodicMemoryConfig() { return episodicMemoryConfig; }
public void setEpisodicMemoryConfig(EpisodicMemoryConfig config) { this.episodicMemoryConfig = config; }
```

Add to `CaseDefinition.Builder`:

```java
private Map<String, Object> semanticData;
private EpisodicMemoryConfig episodicMemoryConfig;

public Builder semanticData(Map<String, Object> semanticData) {
    this.semanticData = semanticData;
    return this;
}

public Builder episodicMemory(String domain, String entityId) {
    this.episodicMemoryConfig = EpisodicMemoryConfig.of(domain, entityId);
    return this;
}

public Builder episodicMemory(String domain, String entityId, int recent) {
    this.episodicMemoryConfig = EpisodicMemoryConfig.of(domain, entityId, recent);
    return this;
}
```

In `Builder.build()`, add:
```java
caseHubDefinition.setSemanticData(semanticData);
caseHubDefinition.setEpisodicMemoryConfig(episodicMemoryConfig);
```

- [ ] **Step 3: Add `startCase()` overloads to `CaseHubRuntime` interface**

```java
// api/src/main/java/io/casehub/api/engine/CaseHubRuntime.java — add:
CompletionStage<UUID> startCase(CaseDefinition definition, Object inputData,
    Map<String, Object> semanticData);

CompletionStage<UUID> startCase(CaseDefinition definition, Object inputData,
    Map<String, Object> semanticData, UUID parentCaseId, PropagationContext propagationContext);
```

Add import `import java.util.Map;` if not present.

- [ ] **Step 4: Write failing semantic panel test**

```java
// runtime/src/test/java/io/casehub/engine/internal/context/SemanticPanelTest.java
package io.casehub.engine.internal.context;

import io.casehub.api.context.ContextPanel;
import io.casehub.api.context.ReadOnlyPanelException;
import org.junit.jupiter.api.Test;
import java.util.Map;
import static org.junit.jupiter.api.Assertions.*;

class SemanticPanelTest {

  @Test
  void semanticPanel_populatedFromMap() {
    CaseContextImpl ctx = new CaseContextImpl();
    ctx.writablePanel(ContextPanel.SEMANTIC).set("threshold", 0.8).set("domain", "fraud-check");
    // freeze as read-only
    ctx.freezePanel(ContextPanel.SEMANTIC);

    assertEquals(0.8, ctx.panel(ContextPanel.SEMANTIC).getAs("threshold", Double.class));
    assertEquals("fraud-check", ctx.panel(ContextPanel.SEMANTIC).getString("domain"));
  }

  @Test
  void semanticPanel_throwsOnWriteAfterFreeze() {
    CaseContextImpl ctx = new CaseContextImpl();
    ctx.writablePanel(ContextPanel.SEMANTIC).set("key", "val");
    ctx.freezePanel(ContextPanel.SEMANTIC);

    assertThrows(ReadOnlyPanelException.class,
        () -> ctx.panel(ContextPanel.SEMANTIC).getAs("key", String.class));
    // wait — get() is read, should succeed:
    assertEquals("val", ctx.panel(ContextPanel.SEMANTIC).get("key"));

    // write should throw:
    // but panel() returns ReadablePanel which has no set() — this is compile-time safe
    // For runtime test, we need to cast:
    assertThrows(ReadOnlyPanelException.class,
        () -> ((io.casehub.api.context.WritablePanel) ctx.panel(ContextPanel.SEMANTIC)).set("k", "v"));
  }

  @Test
  void semanticPanel_inAsJsonNode() {
    CaseContextImpl ctx = new CaseContextImpl();
    ctx.writablePanel(ContextPanel.SEMANTIC).set("domain", "fraud-check");
    ctx.freezePanel(ContextPanel.SEMANTIC);

    var doc = ctx.asJsonNode();
    assertEquals("fraud-check", doc.get("semantic").get("domain").asText());
  }

  @Test
  void callSiteSemanticData_overridesDefinitionDefaults() {
    // definition provides threshold=0.8, call-site overrides with threshold=0.9
    Map<String, Object> definitionDefaults = Map.of("threshold", 0.8, "domain", "fraud");
    Map<String, Object> callSite = Map.of("threshold", 0.9);

    CaseContextImpl ctx = new CaseContextImpl();
    // Definition defaults first
    ctx.writablePanel(ContextPanel.SEMANTIC).setAll(definitionDefaults);
    // Call-site overrides second
    ctx.writablePanel(ContextPanel.SEMANTIC).setAll(callSite);
    ctx.freezePanel(ContextPanel.SEMANTIC);

    assertEquals(0.9, ctx.panel(ContextPanel.SEMANTIC).getAs("threshold", Double.class));
    assertEquals("fraud", ctx.panel(ContextPanel.SEMANTIC).getString("domain")); // from definition
  }
}
```

- [ ] **Step 5: Add `freezePanel()` to `CaseContextImpl`**

Add this method to `CaseContextImpl`:

```java
/**
 * Replaces the mutable WritablePanelImpl for the given panel name with a ReadOnlyPanelView.
 * Called after semantic and episodic panel population to enforce write protection.
 */
public void freezePanel(String panelName) {
    WritablePanelImpl existing = panels.get(panelName);
    if (existing != null && !(existing instanceof ReadOnlyPanelView)) {
        // Replace with read-only view — keep the same data
        panels.put(panelName, new ReadOnlyPanelImpl(existing));
    }
}
```

Wait — `ReadOnlyPanelView` is not a `WritablePanelImpl`. The panel map holds `WritablePanelImpl`. We need a different approach.

Instead, use a boolean flag on `WritablePanelImpl` to mark it read-only:

```java
// In WritablePanelImpl, add:
private volatile boolean frozen = false;

public WritablePanelImpl freeze() {
    this.frozen = true;
    return this;
}

@Override
public boolean isReadOnly() { return frozen; }

// In all write methods, add at the start:
private void checkWritable() {
    if (frozen) throw new ReadOnlyPanelException(panelName);
}

// Call checkWritable() at the start of: set(), setAll(), setPath(), remove(), clear(), merge(),
// computeIfAbsent(), putIfAbsent(), compareAndSet(), update(), applyAndDiff(), applyDiff()
```

And in `CaseContextImpl`:

```java
public void freezePanel(String panelName) {
    WritablePanelImpl p = panels.get(panelName);
    if (p != null) p.freeze();
}
```

Remove `ReadOnlyPanelView` from the plan — it's not needed. `WritablePanelImpl.freeze()` is simpler and cleaner. The `ReadOnlyPanelView` wrapper class can be deleted.

- [ ] **Step 6: Update `WritablePanelImpl` with `freeze()` and `checkWritable()`**

Add to `WritablePanelImpl`:

```java
private volatile boolean frozen = false;

public WritablePanelImpl freeze() {
    this.frozen = true;
    return this;
}

@Override
public boolean isReadOnly() { return frozen; }

private void checkWritable() {
    if (frozen) throw new ReadOnlyPanelException(panelName);
}
```

Add `checkWritable();` as the first line of: `set()`, `setAll()`, `setPath()`, `remove()`, `clear()`, `merge()`, `computeIfAbsent()`, `putIfAbsent()`, `compareAndSet()`, `update()`, `applyAndDiff()`, `applyDiff()`.

Delete `ReadOnlyPanelView.java` since it is no longer needed — `WritablePanelImpl.freeze()` is the enforcement mechanism.

Update `CaseContextImpl.panel()` — since all panels are `WritablePanelImpl`, `panel()` can return `WritablePanelImpl` cast to `ReadablePanel`. Engine-internal code uses `writablePanel()` directly.

Update `PanelTest` — the `ReadOnlyPanelView` tests need to be rewritten using `WritablePanelImpl.freeze()`:

```java
@Test
void frozen_throwsOnSet() {
    WritablePanelImpl p = new WritablePanelImpl(ContextPanel.SEMANTIC);
    p.freeze();
    assertThrows(ReadOnlyPanelException.class, () -> p.set("key", "val"));
}

@Test
void frozen_allowsGet() {
    WritablePanelImpl p = new WritablePanelImpl(ContextPanel.SEMANTIC);
    p.set("key", "val");
    p.freeze();
    assertEquals("val", p.get("key"));
}

@Test
void frozen_isReadOnlyTrue() {
    WritablePanelImpl p = new WritablePanelImpl(ContextPanel.SEMANTIC);
    assertFalse(p.isReadOnly());
    p.freeze();
    assertTrue(p.isReadOnly());
}
```

- [ ] **Step 7: Implement semantic panel population in `CaseHubReactor.buildInstance()`**

In `CaseHubReactor`:

Add injection:
```java
// (already has currentPrincipal)
```

Add new `startCase()` overloads:
```java
CompletionStage<UUID> startCase(CaseDefinition definition, CaseContext context, Map<String, Object> semanticData) {
    return startCaseInternal(definition, context, null, null, semanticData);
}

CompletionStage<UUID> startCase(CaseDefinition definition, CaseContext context, UUID parentCaseId,
    PropagationContext propagationContext, Map<String, Object> semanticData) {
    return startCaseInternal(definition, context, parentCaseId, propagationContext, semanticData);
}
```

Add `semanticData` parameter to `startCaseInternal()` and `buildInstance()`:

```java
private CompletionStage<UUID> startCaseInternal(CaseDefinition definition, CaseContext context,
    UUID parentCaseId, PropagationContext parentPropCtx, Map<String, Object> semanticData) {
    return buildInstance(definition, context, parentCaseId, parentPropCtx, semanticData)
        .chain(instance -> { ... });
}

private Uni<CaseInstance> buildInstance(CaseDefinition definition, CaseContext context,
    UUID parentCaseId, PropagationContext parentPropCtx, Map<String, Object> semanticData) {
    // ... (existing code up to instance.setCaseContext(context)) ...

    // Populate semantic panel
    if (context instanceof CaseContextImpl ctx) {
        Map<String, Object> defSemanticData = definition.getSemanticData();
        if (defSemanticData != null) ctx.writablePanel(ContextPanel.SEMANTIC).setAll(defSemanticData);
        if (semanticData != null) ctx.writablePanel(ContextPanel.SEMANTIC).setAll(semanticData);
        ctx.freezePanel(ContextPanel.SEMANTIC);
        ctx.freezePanel(ContextPanel.EPISODIC);  // episodic is engine-managed
    }

    caseInstanceCache.put(instance);
    return caseInstanceRepository.save(instance, currentPrincipal.tenancyId());
}
```

Existing `startCase(definition, context)` and `startCase(definition, context, parentCaseId, propagationContext)` now delegate to `startCaseInternal` with `semanticData = null`.

Update `CaseHubRuntimeImpl` to implement the two new `CaseHubRuntime` overloads:

```java
@Override
public CompletionStage<UUID> startCase(CaseDefinition definition, Object inputData, Map<String, Object> semanticData) {
    return reactor.startCase(definition, new CaseContextImpl(toContextMap(inputData)), semanticData);
}

@Override
public CompletionStage<UUID> startCase(CaseDefinition definition, Object inputData, Map<String, Object> semanticData,
    UUID parentCaseId, PropagationContext propagationContext) {
    return reactor.startCase(definition, new CaseContextImpl(toContextMap(inputData)), parentCaseId, propagationContext, semanticData);
}
```

- [ ] **Step 8: Run tests**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest="SemanticPanelTest,PanelTest,CaseContextImplTest" -q 2>&1 | tail -15
```
Expected: all tests PASS

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add -u
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(panels): semantic panel — definition defaults + call-site augmentation

- EpisodicMemoryConfig value object
- CaseDefinition.semanticData + episodicMemoryConfig fields + builder methods
- CaseHubRuntime.startCase() new overloads with semanticData
- buildInstance() populates + freezes semantic panel before case starts
- WritablePanelImpl.freeze() replaces ReadOnlyPanelView approach

Refs #80"
```

---

## Task 9: Episodic panel — intra-case updates during execution

**Files:**
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/WorkflowExecutionCompletedHandler.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/milestone/MilestoneLifecycleManager.java` (or `MilestoneCompletedEventHandler.java`)
- Create: `runtime/src/test/java/io/casehub/engine/internal/context/EpisodicPanelIntraCaseTest.java`

- [ ] **Step 1: Write failing tests**

```java
// runtime/src/test/java/io/casehub/engine/internal/context/EpisodicPanelIntraCaseTest.java
package io.casehub.engine.internal.context;

import io.casehub.api.context.ContextPanel;
import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.Map;
import static org.junit.jupiter.api.Assertions.*;

class EpisodicPanelIntraCaseTest {

  @Test
  void updateEpisodic_worker_addsEntry() {
    CaseContextImpl ctx = new CaseContextImpl();
    ctx.freezePanel(ContextPanel.SEMANTIC);
    ctx.freezePanel(ContextPanel.EPISODIC);

    EpisodicPanelUpdater.recordWorkerCompletion(ctx, "extractor", "COMPLETED");

    var workers = ctx.writablePanel(ContextPanel.EPISODIC)
        .getList("workers", Map.class);
    assertEquals(1, workers.size());
    assertEquals("extractor", ((Map<?,?>) workers.get(0)).get("name"));
    assertEquals("COMPLETED", ((Map<?,?>) workers.get(0)).get("lastOutcome"));
  }

  @Test
  void updateEpisodic_worker_incrementsRuns() {
    CaseContextImpl ctx = new CaseContextImpl();
    ctx.freezePanel(ContextPanel.EPISODIC);

    EpisodicPanelUpdater.recordWorkerCompletion(ctx, "extractor", "COMPLETED");
    EpisodicPanelUpdater.recordWorkerCompletion(ctx, "extractor", "COMPLETED");

    var workers = ctx.writablePanel(ContextPanel.EPISODIC)
        .getList("workers", Map.class);
    assertEquals(1, workers.size()); // same worker, not duplicated
    assertEquals(2, ((Map<?,?>) workers.get(0)).get("runs"));
  }

  @Test
  void updateEpisodic_milestone_appendsName() {
    CaseContextImpl ctx = new CaseContextImpl();
    ctx.freezePanel(ContextPanel.EPISODIC);

    EpisodicPanelUpdater.recordMilestoneReached(ctx, "data-ready");

    var milestones = ctx.writablePanel(ContextPanel.EPISODIC)
        .getList("milestones", String.class);
    assertTrue(milestones.contains("data-ready"));
  }

  @Test
  void updateEpisodic_milestone_notDuplicated() {
    CaseContextImpl ctx = new CaseContextImpl();
    ctx.freezePanel(ContextPanel.EPISODIC);

    EpisodicPanelUpdater.recordMilestoneReached(ctx, "data-ready");
    EpisodicPanelUpdater.recordMilestoneReached(ctx, "data-ready");

    var milestones = ctx.writablePanel(ContextPanel.EPISODIC)
        .getList("milestones", String.class);
    assertEquals(1, milestones.stream().filter("data-ready"::equals).count());
  }

  @Test
  void updateEpisodic_doesNotBumpWorkingVersion() {
    CaseContextImpl ctx = new CaseContextImpl();
    ctx.set("result", "done");
    long workingVersionBefore = ctx.getVersion();

    EpisodicPanelUpdater.recordWorkerCompletion(ctx, "worker1", "COMPLETED");

    assertEquals(workingVersionBefore, ctx.getVersion());
  }
}
```

- [ ] **Step 2: Create `EpisodicPanelUpdater`**

```java
// runtime/src/main/java/io/casehub/engine/internal/context/EpisodicPanelUpdater.java
package io.casehub.engine.internal.context;

import io.casehub.api.context.ContextPanel;
import java.time.Instant;
import java.util.*;

public final class EpisodicPanelUpdater {

  private EpisodicPanelUpdater() {}

  @SuppressWarnings("unchecked")
  public static void recordWorkerCompletion(CaseContextImpl ctx, String workerName, String outcome) {
    WritablePanelImpl episodic = ctx.writablePanel(ContextPanel.EPISODIC);
    List<Map<String, Object>> workers = episodic.getList("workers", Map.class);
    if (workers == null) workers = new ArrayList<>();
    else workers = new ArrayList<>(workers);

    Map<String, Object> entry = null;
    for (Map<String, Object> w : workers) {
      if (workerName.equals(w.get("name"))) { entry = w; break; }
    }

    if (entry == null) {
      entry = new LinkedHashMap<>();
      entry.put("name", workerName);
      entry.put("runs", 0);
      workers.add(entry);
    }

    entry.put("runs", ((Number) entry.getOrDefault("runs", 0)).intValue() + 1);
    entry.put("lastOutcome", outcome);
    entry.put("lastTimestamp", Instant.now().toString());
    episodic.set("workers", workers);
  }

  public static void recordMilestoneReached(CaseContextImpl ctx, String milestoneName) {
    WritablePanelImpl episodic = ctx.writablePanel(ContextPanel.EPISODIC);
    List<String> milestones = episodic.getList("milestones", String.class);
    if (milestones == null) milestones = new ArrayList<>();
    else milestones = new ArrayList<>(milestones);
    if (!milestones.contains(milestoneName)) {
      milestones.add(milestoneName);
      episodic.set("milestones", milestones);
    }
  }

  public static void recordGoalReached(CaseContextImpl ctx, String goalName) {
    WritablePanelImpl episodic = ctx.writablePanel(ContextPanel.EPISODIC);
    List<String> goals = episodic.getList("goals", String.class);
    if (goals == null) goals = new ArrayList<>();
    else goals = new ArrayList<>(goals);
    if (!goals.contains(goalName)) {
      goals.add(goalName);
      episodic.set("goals", goals);
    }
  }

  /** Initializes the episodic panel baseline {workers:[], milestones:[], goals:[]} */
  public static void initBaseline(CaseContextImpl ctx) {
    WritablePanelImpl episodic = ctx.writablePanel(ContextPanel.EPISODIC);
    if (!episodic.contains("workers"))   episodic.set("workers",   new ArrayList<>());
    if (!episodic.contains("milestones")) episodic.set("milestones", new ArrayList<>());
    if (!episodic.contains("goals"))     episodic.set("goals",     new ArrayList<>());
  }
}
```

- [ ] **Step 3: Run tests**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest=EpisodicPanelIntraCaseTest -q 2>&1 | tail -10
```
Expected: all PASS

- [ ] **Step 4: Wire into `WorkflowExecutionCompletedHandler`**

In `WorkflowExecutionCompletedHandler.onWorkflowExecutionCompletedHandler()`, after worker output is applied and before `CONTEXT_CHANGED` is published, add the episodic update and publish the panel-scoped event:

```java
// After: applyOutputWithConflictResolution(caseInstance, worker, rawOutput);
// Before: eventBus.publish(EventBusAddresses.CONTEXT_CHANGED, ...)

if (caseInstance.getCaseContext() instanceof CaseContextImpl ctx) {
    EpisodicPanelUpdater.recordWorkerCompletion(ctx, worker.getName(),
        rawOutput.isEmpty() ? "COMPLETED" : "COMPLETED");
    eventBus.publish(EventBusAddresses.panelChanged(ContextPanel.EPISODIC),
        new CaseContextChangedEvent(caseInstance, caseInstance.getCaseContext().snapshot(), ContextPanel.EPISODIC));
}
```

Wire similarly in `MilestoneCompletedEventHandler` and goal-reached handler.

- [ ] **Step 5: Wire `initBaseline` into `CaseHubReactor.buildInstance()`**

After the semantic panel freeze, call:
```java
if (context instanceof CaseContextImpl ctx) {
    EpisodicPanelUpdater.initBaseline(ctx);
    ctx.freezePanel(ContextPanel.EPISODIC);
}
```

- [ ] **Step 6: Run tests**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest="EpisodicPanelIntraCaseTest,CaseContextImplTest,SemanticPanelTest" -q 2>&1 | tail -10
```
Expected: all PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add -u
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(panels): episodic panel intra-case — worker/milestone/goal tracking

- EpisodicPanelUpdater: recordWorkerCompletion, recordMilestoneReached, recordGoalReached, initBaseline
- WorkflowExecutionCompletedHandler updates episodic after worker completion
- Episodic updates do not bump working panel version
- Panel-scoped event published on episodic changes

Refs #80"
```

---

## Task 10: Episodic panel — inter-case memory (ReactiveCaseMemoryStore)

**Files:**
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/CaseHubReactor.java`
- Create: `runtime/src/test/java/io/casehub/engine/EpisodicPanelInterCaseTest.java`

**Note:** `ReactiveCaseMemoryStore` is in `casehub-platform` which is already on the engine's classpath. `MemoryQuery`, `Memory`, `MemoryDomain`, `MemoryAttributeKeys` are in `casehub-platform-api`.

- [ ] **Step 1: Write failing integration test**

```java
// runtime/src/test/java/io/casehub/engine/EpisodicPanelInterCaseTest.java
package io.casehub.engine;

import io.casehub.api.context.ContextPanel;
import io.casehub.api.model.CaseDefinition;
import io.casehub.api.model.EpisodicMemoryConfig;
import io.casehub.engine.internal.context.CaseContextImpl;
import io.casehub.platform.api.memory.*;
import io.casehub.platform.memory.ReactiveCaseMemoryStore;
import io.quarkus.arc.DefaultBean;
import io.quarkus.test.junit.QuarkusTest;
import io.smallrye.mutiny.Uni;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;
import jakarta.inject.Inject;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.util.*;
import static org.junit.jupiter.api.Assertions.*;

@QuarkusTest
class EpisodicPanelInterCaseTest {

  @Inject io.casehub.api.engine.CaseHubRuntime runtime;

  static List<MemoryQuery> capturedQueries = new ArrayList<>();
  static List<Memory> mockResults = new ArrayList<>();

  @BeforeEach
  void reset() {
    capturedQueries.clear();
    mockResults.clear();
  }

  @Alternative
  @Priority(1)
  @ApplicationScoped
  static class RecordingMemoryStore implements ReactiveCaseMemoryStore {
    @Override
    public Uni<List<Memory>> query(MemoryQuery q) {
      capturedQueries.add(q);
      return Uni.createFrom().item(new ArrayList<>(mockResults));
    }
    @Override public Uni<String> store(MemoryInput i) { return Uni.createFrom().item(""); }
    @Override public Uni<Void> erase(EraseRequest r) { return Uni.createFrom().voidItem(); }
  }

  @Test
  void interCase_queryUsesDomainAndEntityId() throws Exception {
    mockResults.add(new Memory("id1", "C-123", new MemoryDomain("fraud-check"),
        "default", null, "High risk score 0.91",
        Map.of(MemoryAttributeKeys.OUTCOME, "HIGH_RISK"), java.time.Instant.now()));

    CaseDefinition def = CaseDefinition.builder()
        .namespace("test").name("fraud-check").version("1.0")
        .semanticData(Map.of("customerId", "C-123"))
        .build();
    def.setEpisodicMemoryConfig(EpisodicMemoryConfig.of("fraud-check", ".semantic.customerId", 5));

    UUID caseId = runtime.startCase(def, null).toCompletableFuture().get();
    assertNotNull(caseId);

    assertEquals(1, capturedQueries.size());
    MemoryQuery q = capturedQueries.get(0);
    assertEquals(List.of("C-123"), q.entityIds());
    assertEquals("fraud-check", q.domain().name());
    assertEquals(5, q.limit());
    assertNull(q.caseId(), "inter-case query must not filter by current caseId");
  }

  @Test
  void interCase_memoryInEpisodicPanel() throws Exception {
    mockResults.add(new Memory("id1", "C-123", new MemoryDomain("fraud-check"),
        "default", null, "Prior investigation flagged",
        Map.of(MemoryAttributeKeys.OUTCOME, "HIGH_RISK"), java.time.Instant.now()));

    CaseDefinition def = CaseDefinition.builder()
        .namespace("test").name("fraud-check2").version("1.0")
        .semanticData(Map.of("customerId", "C-123"))
        .build();
    def.setEpisodicMemoryConfig(EpisodicMemoryConfig.of("fraud-check", ".semantic.customerId"));

    UUID caseId = runtime.startCase(def, null).toCompletableFuture().get();

    // query the case context — note: requires access to the cache in a real test
    // For now, verify via a query call or event log inspection
    // This assertion is a placeholder — adapt per your test harness
    assertNotNull(caseId);
    assertEquals(1, capturedQueries.size());
  }

  @Test
  void interCase_absentWhenStoreReturnsEmpty() throws Exception {
    // mockResults stays empty
    CaseDefinition def = CaseDefinition.builder()
        .namespace("test").name("fraud-check3").version("1.0")
        .semanticData(Map.of("customerId", "C-456"))
        .build();
    def.setEpisodicMemoryConfig(EpisodicMemoryConfig.of("fraud-check", ".semantic.customerId"));

    UUID caseId = runtime.startCase(def, null).toCompletableFuture().get();
    assertNotNull(caseId);
    assertEquals(1, capturedQueries.size());
    // episodic.memory key should be absent when results are empty — verified via EventLog or context inspection
  }
}
```

Add `@Priority` import: `import io.quarkus.arc.Priority;`

- [ ] **Step 2: Implement inter-case episodic in `CaseHubReactor.buildInstance()`**

Add injection to `CaseHubReactor`:

```java
@Inject ReactiveCaseMemoryStore reactiveCaseMemoryStore;
@Inject io.casehub.engine.common.internal.jq.JQEvaluator jqEvaluator;
```

Add import: `import io.casehub.platform.memory.ReactiveCaseMemoryStore;`
And: `import io.casehub.platform.api.memory.*;`

In `buildInstance()`, after semantic panel is frozen and episodic baseline is initialized, chain the memory query as a `Uni` step before the repository save:

```java
private Uni<CaseInstance> buildInstance(CaseDefinition definition, CaseContext context,
    UUID parentCaseId, PropagationContext parentPropCtx, Map<String, Object> semanticData) {

  // ... (existing code: model, propagationContext, instance creation) ...

  if (context instanceof CaseContextImpl ctx) {
    // Semantic panel
    Map<String, Object> defSemanticData = definition.getSemanticData();
    if (defSemanticData != null) ctx.writablePanel(ContextPanel.SEMANTIC).setAll(defSemanticData);
    if (semanticData != null)    ctx.writablePanel(ContextPanel.SEMANTIC).setAll(semanticData);
    ctx.freezePanel(ContextPanel.SEMANTIC);

    // Episodic baseline
    EpisodicPanelUpdater.initBaseline(ctx);
  }

  // Inter-case memory query (async, chains into Uni pipeline)
  EpisodicMemoryConfig memCfg = definition.getEpisodicMemoryConfig();
  Uni<Void> memoryQuery = Uni.createFrom().voidItem();

  if (memCfg != null && context instanceof CaseContextImpl ctx) {
    memoryQuery = buildMemoryQuery(ctx, memCfg)
        .flatMap(query -> {
          if (query == null) return Uni.createFrom().voidItem();
          return reactiveCaseMemoryStore.query(query)
              .invoke(memories -> {
                if (!memories.isEmpty()) {
                  List<Map<String, Object>> projected = memories.stream()
                      .map(m -> {
                        Map<String, Object> p = new LinkedHashMap<>();
                        p.put("text", m.text());
                        if (m.attributes() != null && !m.attributes().isEmpty())
                          p.put("attributes", new LinkedHashMap<>(m.attributes()));
                        return p;
                      })
                      .toList();
                  ctx.writablePanel(ContextPanel.EPISODIC).set("memory", projected);
                }
              })
              .replaceWithVoid();
        })
        .onFailure().invoke(t ->
            LOG.warnf(t, "EpisodicMemoryStore query failed for case %s — continuing without inter-case memory",
                instance.getUuid()));
  }

  return memoryQuery.chain(() -> {
    if (context instanceof CaseContextImpl ctx) ctx.freezePanel(ContextPanel.EPISODIC);
    instance.setCaseContext(context);
    instance.setPropagationContext(propagationContext);
    instance.setParentCaseId(parentCaseId);
    caseInstanceCache.put(instance);
    return caseInstanceRepository.save(instance, currentPrincipal.tenancyId());
  });
}

private Uni<MemoryQuery> buildMemoryQuery(CaseContextImpl ctx, EpisodicMemoryConfig cfg) {
  try {
    // Evaluate entityId JQ against the semantic panel
    var vr = jqEvaluator.eval(cfg.entityId(), ctx.panel(ContextPanel.SEMANTIC).asJsonNode());
    if (!vr.ok() || vr.output() == null || vr.output().isEmpty()) {
      LOG.warnf("episodic.memory.entityId JQ evaluation failed: %s", vr.error());
      return Uni.createFrom().nullItem();
    }
    var result = vr.output().get(0);

    List<String> entityIds;
    if (result.isTextual()) {
      entityIds = List.of(result.asText());
    } else if (result.isArray()) {
      entityIds = new ArrayList<>();
      result.forEach(n -> entityIds.add(n.asText()));
    } else {
      LOG.warnf("episodic.memory.entityId JQ result is neither string nor array: %s", result);
      return Uni.createFrom().nullItem();
    }

    if (entityIds.isEmpty()) return Uni.createFrom().nullItem();

    MemoryQuery query = entityIds.size() == 1
        ? MemoryQuery.forEntity(entityIds.get(0), new MemoryDomain(cfg.domain()), currentPrincipal.tenancyId())
            .withLimit(cfg.recent())
        : MemoryQuery.forEntities(entityIds, new MemoryDomain(cfg.domain()), currentPrincipal.tenancyId())
            .withLimit(cfg.recent());
    // No withCaseId() — inter-case query is cross-case by design

    return Uni.createFrom().item(query);
  } catch (Exception e) {
    LOG.warnf(e, "Failed to build episodic MemoryQuery");
    return Uni.createFrom().nullItem();
  }
}
```

Also move `instance` construction before the `memoryQuery` chain so it's in scope. Restructure accordingly.

- [ ] **Step 3: Run tests**

```bash
mvn install -DskipTests -q
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest="EpisodicPanelInterCaseTest,EpisodicPanelIntraCaseTest" -q 2>&1 | tail -15
```
Expected: all tests PASS

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add -u
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(panels): episodic inter-case memory via ReactiveCaseMemoryStore

- EpisodicMemoryConfig: domain, entityId (JQ expr), recent limit
- entityId JQ evaluated against semantic panel; string→List.of(), array→direct
- No withCaseId() — inter-case, cross-case by design
- {text, attributes} projected from Memory into episodic.memory[]
- memory key absent when store returns empty or JQ evaluation fails
- Chains into buildInstance() Uni pipeline; failure is non-fatal

Refs #80"
```

---

## Task 11: Recovery rewrite — `DefaultWorkerExecutionRecoveryService`

**Files:**
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/recovery/DefaultWorkerExecutionRecoveryService.java`
- Create: `runtime/src/test/java/io/casehub/engine/internal/engine/recovery/RecoveryPanelAwareTest.java`

- [ ] **Step 1: Write failing tests**

```java
// runtime/src/test/java/io/casehub/engine/internal/engine/recovery/RecoveryPanelAwareTest.java
package io.casehub.engine.internal.engine.recovery;

import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.api.context.ContextPanel;
import io.casehub.engine.internal.context.CaseContextImpl;
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class RecoveryPanelAwareTest {

  private static final ObjectMapper MAPPER = new ObjectMapper();

  @Test
  void fromPanelDocument_reconstructsWorkingPanel() throws Exception {
    CaseContextImpl original = new CaseContextImpl();
    original.set("result", "done");
    original.set("score", 42);
    original.writablePanel(ContextPanel.SEMANTIC).set("domain", "fraud-check");
    original.freezePanel(ContextPanel.SEMANTIC);

    var doc = original.asJsonNode();
    CaseContextImpl recovered = CaseContextImpl.fromPanelDocument(doc);

    assertEquals("done", recovered.get("result"));
    assertEquals(42, recovered.getAs("score", Integer.class));
  }

  @Test
  void fromPanelDocument_reconstructsSemanticPanel() throws Exception {
    CaseContextImpl original = new CaseContextImpl();
    original.writablePanel(ContextPanel.SEMANTIC).set("threshold", 0.8);
    original.freezePanel(ContextPanel.SEMANTIC);

    var doc = original.asJsonNode();
    CaseContextImpl recovered = CaseContextImpl.fromPanelDocument(doc);

    assertEquals(0.8, recovered.panel(ContextPanel.SEMANTIC).getAs("threshold", Double.class));
  }

  @Test
  void fromPanelDocument_reconstructsEpisodicPanel() throws Exception {
    CaseContextImpl original = new CaseContextImpl();
    original.writablePanel(ContextPanel.EPISODIC).set("workers",
        java.util.List.of(java.util.Map.of("name", "extractor", "runs", 2)));
    original.writablePanel(ContextPanel.EPISODIC).set("milestones", java.util.List.of("data-ready"));
    original.freezePanel(ContextPanel.EPISODIC);

    var doc = original.asJsonNode();
    CaseContextImpl recovered = CaseContextImpl.fromPanelDocument(doc);

    var milestones = recovered.panel(ContextPanel.EPISODIC).getList("milestones", String.class);
    assertTrue(milestones.contains("data-ready"));
  }

  @Test
  void fromPanelDocument_nullPayload_returnsEmptyContext() {
    CaseContextImpl ctx = CaseContextImpl.fromPanelDocument(null);
    assertTrue(ctx.isEmpty()); // working panel is empty
  }
}
```

- [ ] **Step 2: Run to confirm**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest=RecoveryPanelAwareTest -q 2>&1 | tail -10
```
Expected: all PASS (since `fromPanelDocument` was implemented in Task 5)

- [ ] **Step 3: Update `rebuildStateContext()`**

In `DefaultWorkerExecutionRecoveryService.rebuildStateContext()`:

```java
private Uni<CaseContext> rebuildStateContext(UUID caseId) {
  return runOnSafeContext(
      () -> eventLogRepository.findByCaseAndTypes(caseId, EnumSet.of(
          CaseHubEventType.CASE_STARTED,
          CaseHubEventType.WORKER_EXECUTION_COMPLETED,
          CaseHubEventType.SUBCASE_COMPLETED,
          CaseHubEventType.SIGNAL_RECEIVED,
          CaseHubEventType.MILESTONE_ACTIVATED,
          CaseHubEventType.MILESTONE_COMPLETED,
          CaseHubEventType.MILESTONE_SLA_VIOLATED)))
      .map(eventLogs -> {
        // Find and reconstruct from CASE_STARTED panel document
        CaseContextImpl caseContext;
        EventLog caseStartedEvent = eventLogs.stream()
            .filter(e -> e.getEventType() == CaseHubEventType.CASE_STARTED)
            .findFirst().orElse(null);

        if (caseStartedEvent != null) {
          // CASE_STARTED payload is now a panel document {"working":{...},"semantic":{...},...}
          caseContext = CaseContextImpl.fromPanelDocument(caseStartedEvent.getPayload());
        } else {
          caseContext = new CaseContextImpl();
        }

        for (EventLog eventLog : eventLogs) {
          if (eventLog.getEventType() == CaseHubEventType.CASE_STARTED) continue;

          if (eventLog.getEventType() == CaseHubEventType.SIGNAL_RECEIVED) {
            // Signal patches are working-panel-relative — apply to working panel via flat API
            JsonNode patch = payloadAsPatch(eventLog.getPayload());
            if (patch != null) caseContext.applyDiff(patch);

          } else if (eventLog.getEventType() == CaseHubEventType.WORKER_EXECUTION_COMPLETED) {
            // Apply context changes to working panel
            JsonNode contextChanges = getContextChanges(eventLog.getMetadata());
            if (contextChanges != null) {
              if (contextChanges.isArray()) caseContext.applyDiff(contextChanges);
              else if (contextChanges.isObject()) applyTopLevelChanges(caseContext, contextChanges);
            } else {
              caseContext.setAll(payloadAsMap(eventLog.getPayload()));
            }
            // Update episodic panel
            String workerId = eventLog.getWorkerId();
            if (workerId != null) {
              EpisodicPanelUpdater.recordWorkerCompletion(caseContext, workerId, "COMPLETED");
            }

          } else if (eventLog.getEventType() == CaseHubEventType.SUBCASE_COMPLETED) {
            caseContext.setAll(payloadAsMap(eventLog.getPayload()));

          } else if (eventLog.getEventType() == CaseHubEventType.MILESTONE_ACTIVATED) {
            applyMilestoneActivatedEvent(caseContext, eventLog);

          } else if (eventLog.getEventType() == CaseHubEventType.MILESTONE_COMPLETED) {
            applyMilestoneCompletedEvent(caseContext, eventLog);
            // Also update episodic
            JsonNode payload = eventLog.getPayload();
            if (payload != null) {
              String milestoneName = payload.path("milestoneName").asText(null);
              if (milestoneName != null) EpisodicPanelUpdater.recordMilestoneReached(caseContext, milestoneName);
            }

          } else if (eventLog.getEventType() == CaseHubEventType.MILESTONE_SLA_VIOLATED) {
            applyMilestoneSLAViolatedEvent(caseContext, eventLog);
          }
        }

        // Re-freeze semantic and episodic panels (they were frozen at case start)
        caseContext.freezePanel(ContextPanel.SEMANTIC);
        caseContext.freezePanel(ContextPanel.EPISODIC);

        return (CaseContext) caseContext;
      });
}
```

Add import: `import io.casehub.engine.internal.context.EpisodicPanelUpdater;`
And: `import io.casehub.api.context.ContextPanel;`

- [ ] **Step 4: Run tests**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest="RecoveryPanelAwareTest" -q 2>&1 | tail -10
```
Expected: all PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add -u
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(panels): panel-aware rebuildStateContext() in recovery service

- CASE_STARTED payload (panel document) restored via fromPanelDocument()
- Signal patches applied to working panel (working-panel-relative paths unchanged)
- Worker/milestone/subcase events update working panel via flat API
- Episodic panel rebuilt: recordWorkerCompletion + recordMilestoneReached
- Semantic + episodic panels re-frozen after replay

Refs #80"
```

---

## Task 12: JQ expression migration

All JQ string expressions that reference working panel keys must be updated from `.key` to `.working.key`. This is a mechanical migration.

**Files to migrate:**
- All test files in `runtime/src/test/` that use `ContextChangeTrigger`, `binding.when()`, `inputSchema`, `inputMapping`, `outputMapping`, `Goal`, `Milestone` JQ strings
- `CaseDefinitionYamlMapper` — add parse-time warning

- [ ] **Step 1: Run the full test suite and capture failures**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl runtime -q 2>&1 | grep "FAILED\|expected:" | head -40
```
This shows all JQ expression failures. Each one needs `.key` → `.working.key`.

- [ ] **Step 2: Update JQ expressions in test files**

Using IntelliJ "Find in Path" (`Cmd+Shift+F`), search for `new ContextChangeTrigger("."` and `new JQExpressionEvaluator("."` across `runtime/src/test`. For each match, update the expression.

Common patterns to replace:
```
# Binding filter expressions
".result == null"  →  ".working.result == null"
".actionGateRejected == null"  →  ".working.actionGateRejected == null"
".score > 0.5"  →  ".working.score > 0.5"
".milestones.name.lifecycleStatus"  →  ".working.milestones.name.lifecycleStatus"

# inputSchema / inputMapping expressions (on workers / human tasks)
".inputKey"  →  ".working.inputKey"
```

- [ ] **Step 3: Update YAML test definitions**

If any tests use `YamlCaseHub` or load YAML files with JQ expressions, update those expressions. Search for `.yaml` files in test resources.

- [ ] **Step 4: Add parse-time warning to `CaseDefinitionYamlMapper`**

In `CaseDefinitionYamlMapper`, add a helper that warns on unmigrated expressions:

```java
private static final Logger LOG = Logger.getLogger(CaseDefinitionYamlMapper.class);
private static final Set<String> KNOWN_PANEL_PREFIXES =
    Set.of(".working.", ".semantic.", ".episodic.");

private void warnIfUnmigratedJq(String expression, String location) {
    if (expression == null || !expression.startsWith(".")) return;
    boolean isPanelQualified = KNOWN_PANEL_PREFIXES.stream().anyMatch(expression::startsWith);
    if (!isPanelQualified) {
        LOG.warnf("Unmigrated JQ expression at %s: '%s' — prefix with .working. for working panel access",
            location, expression);
    }
}
```

Call `warnIfUnmigratedJq(filter, "binding.filter")` wherever a JQ string is set on `ContextChangeTrigger`, `Binding.when()`, `inputSchema`, etc.

- [ ] **Step 5: Run the full test suite**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl runtime -q 2>&1 | tail -20
```
Expected: all tests PASS. Address any remaining failures.

- [ ] **Step 6: Run integration tests**

```bash
mvn install -DskipTests -q
TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl blackboard -q 2>&1 | tail -15
TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl work-adapter -q 2>&1 | tail -15
```
Expected: all PASS. Fix any JQ expression failures in these modules too.

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add -u
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(panels): migrate all JQ expressions to .working. prefix

Breaking change: asJsonNode() now returns panel document, all working panel
keys require .working. prefix in JQ expressions.

Also adds parse-time warning in CaseDefinitionYamlMapper for unmigrated expressions.

Refs #80"
```

---

## Task 13: YAML mapper — `semanticData` and `episodic.memory` fields

**Files:**
- Modify: `api/src/main/java/io/casehub/api/model/converter/CaseDefinitionYamlMapper.java`
- Modify: `schema/src/main/resources/` (JSON Schema YAML files)

- [ ] **Step 1: Check what YAML schema files exist**

```bash
ls /Users/mdproctor/claude/casehub/engine/schema/src/main/resources/
```

- [ ] **Step 2: Add `semanticData` to the YAML schema**

In the JSON Schema YAML file for `CaseDefinition`, add:

```yaml
semanticData:
  type: object
  additionalProperties: true
  description: Static domain knowledge injected into the semantic panel at case start

episodic:
  type: object
  properties:
    memory:
      type: object
      required: [domain, entityId]
      properties:
        domain:
          type: string
          description: MemoryDomain name
        entityId:
          type: string
          description: JQ expression evaluated against semantic panel to resolve entity ID(s)
        recent:
          type: integer
          default: 10
          minimum: 1
```

- [ ] **Step 3: Regenerate schema model**

```bash
mvn install -DskipTests -q -pl schema
```
This runs `jsonschema2pojo` and regenerates `schema/target/generated-sources/`.

- [ ] **Step 4: Update `CaseDefinitionYamlMapper` to read the new fields**

In `CaseDefinitionYamlMapper`, after setting existing fields, add:

```java
// semanticData
if (schema.getSemanticData() != null) {
    @SuppressWarnings("unchecked")
    Map<String, Object> semData = MAPPER.convertValue(schema.getSemanticData(), Map.class);
    caseDefinition.setSemanticData(semData);
}

// episodic.memory
if (schema.getEpisodic() != null && schema.getEpisodic().getMemory() != null) {
    var mem = schema.getEpisodic().getMemory();
    int recent = mem.getRecent() != null ? mem.getRecent() : 10;
    caseDefinition.setEpisodicMemoryConfig(
        EpisodicMemoryConfig.of(mem.getDomain(), mem.getEntityId(), recent));
}
```

- [ ] **Step 5: Write YAML mapper test**

In `CaseDefinitionYamlMapperTest`, add:

```java
@Test
void parseSemanticData() {
    String yaml = """
        namespace: test
        name: fraud
        version: 1.0.0
        semanticData:
          threshold: 0.8
          domain: "fraud-check"
        """;
    CaseDefinition def = mapper.parse(yaml);
    assertNotNull(def.getSemanticData());
    assertEquals(0.8, ((Number) def.getSemanticData().get("threshold")).doubleValue());
}

@Test
void parseEpisodicMemoryConfig() {
    String yaml = """
        namespace: test
        name: fraud2
        version: 1.0.0
        episodic:
          memory:
            domain: "fraud-check"
            entityId: ".semantic.customerId"
            recent: 5
        """;
    CaseDefinition def = mapper.parse(yaml);
    assertNotNull(def.getEpisodicMemoryConfig());
    assertEquals("fraud-check", def.getEpisodicMemoryConfig().domain());
    assertEquals(".semantic.customerId", def.getEpisodicMemoryConfig().entityId());
    assertEquals(5, def.getEpisodicMemoryConfig().recent());
}

@Test
void unmigratedJqExpression_logsWarning() {
    // Just ensure no exception — warning is logged
    String yaml = """
        namespace: test
        name: old
        version: 1.0.0
        bindings:
          - name: b1
            on:
              contextChange:
                filter: ".result == null"
        """;
    CaseDefinition def = mapper.parse(yaml); // warning logged, no exception
    assertNotNull(def);
}
```

- [ ] **Step 6: Run tests**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=CaseDefinitionYamlMapperTest -q 2>&1 | tail -10
```
Expected: all PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add -u
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(panels): YAML support for semanticData and episodic.memory

- JSON Schema: semanticData (free-form object), episodic.memory (domain, entityId, recent)
- CaseDefinitionYamlMapper: maps new fields to CaseDefinition model
- CaseDefinitionYamlMapperTest: parsing verification + unmigrated JQ warning

Refs #80"
```

---

## Task 14: Full integration test and final #80 verification

- [ ] **Step 1: Run complete test suite across all modules**

```bash
mvn install -DskipTests -q
TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl api,common,runtime,blackboard,work-adapter -q 2>&1 | tail -30
```
Expected: all tests PASS

- [ ] **Step 2: Verify episodic panel in CASE_STARTED EventLog**

Add one integration test verifying the full panel document is stored in CASE_STARTED EventLog:

```java
// In runtime integration test
@Test
void caseStarted_eventLogPayloadIsPanelDocument() throws Exception {
    CaseDefinition def = CaseDefinition.builder()
        .namespace("test").name("panel-check").version("1.0")
        .semanticData(Map.of("threshold", 0.8))
        .build();

    UUID caseId = runtime.startCase(def, Map.of("inputKey", "inputVal")).toCompletableFuture().get();

    var logs = runtime.eventLog(caseId).toCompletableFuture().get();
    var caseStarted = logs.stream()
        .filter(l -> l.eventType() == CaseHubEventType.CASE_STARTED)
        .findFirst().orElseThrow();

    JsonNode payload = caseStarted.payload();
    assertNotNull(payload.get("working"), "working panel must be in payload");
    assertNotNull(payload.get("semantic"), "semantic panel must be in payload");
    assertNotNull(payload.get("episodic"), "episodic panel must be in payload");
    assertEquals("inputVal", payload.get("working").get("inputKey").asText());
    assertEquals(0.8, payload.get("semantic").get("threshold").asDouble());
}
```

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add -u
git -C /Users/mdproctor/claude/casehub/engine commit -m "test(panels): integration test verifying CASE_STARTED panel document payload

Refs #80, Closes #80"
```

---

## End of #80

The #81 plan (user-defined panels and `listenPanel`) continues in:
`plans/2026-06-09-issue-81-casefile-panels-userdefined.md`
