# MapCaseFile Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add `MapCaseFile` — a `CaseContextImpl` subclass that bridges the poc's `put/get/keys` naming to the engine's `set/getAs/getKeys`, easing casehub-poc migration.

**Architecture:** `MapCaseFile extends CaseContextImpl` in a new public package `io.casehub.engine.context` inside the `runtime` module. Three alias methods delegate to the inherited implementation. No new state; thread safety and all other behaviour inherited.

**Tech Stack:** Java 21, JUnit 5, AssertJ. No Quarkus, no containers.

**Issue:** treblereel/casehub-engine#239  
**Spec:** `specs/2026-05-12-map-case-file-design.md`  
**Protocol:** PP-20260512-5f055d (Optional return types — map accessors use null, not Optional)

---

## File Map

| Action | Path |
|--------|------|
| Create | `runtime/src/main/java/io/casehub/engine/context/MapCaseFile.java` |
| Create | `runtime/src/test/java/io/casehub/engine/context/MapCaseFileTest.java` |

---

### Task 1: Write the failing tests

**Files:**
- Create: `runtime/src/test/java/io/casehub/engine/context/MapCaseFileTest.java`

- [ ] **Step 1: Create the test file**

```java
/*
 * Copyright 2026-Present The Case Hub Authors
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 * http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
package io.casehub.engine.context;

import static org.assertj.core.api.Assertions.assertThat;

import io.casehub.api.context.CaseContext;
import java.util.Map;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;

@DisplayName("MapCaseFile")
class MapCaseFileTest {

  @Nested
  @DisplayName("construction")
  class Construction {

    @Test
    @DisplayName("empty constructor creates empty map at version 0")
    void empty_createsEmpty() {
      final var map = new MapCaseFile();
      assertThat(map.isEmpty()).isTrue();
      assertThat(map.size()).isZero();
      assertThat(map.getVersion()).isZero();
    }

    @Test
    @DisplayName("map constructor pre-populates data")
    void mapConstructor_prePopulates() {
      final var map = new MapCaseFile(Map.of("a", "hello", "b", 42));
      assertThat(map.getString("a")).isEqualTo("hello");
      assertThat(map.getInt("b")).isEqualTo(42);
    }

    @Test
    @DisplayName("is a CaseContext")
    void isCaseContext() {
      assertThat(new MapCaseFile()).isInstanceOf(CaseContext.class);
    }
  }

  @Nested
  @DisplayName("put / get")
  class PutGet {

    @Test
    @DisplayName("put then get returns typed value")
    void put_get_roundtrip() {
      final var map = new MapCaseFile();
      map.put("score", 99);
      assertThat(map.get("score", Integer.class)).isEqualTo(99);
    }

    @Test
    @DisplayName("get returns null for missing key")
    void get_missingKey_returnsNull() {
      final var map = new MapCaseFile();
      assertThat(map.get("missing", String.class)).isNull();
    }

    @Test
    @DisplayName("put overwrites existing value")
    void put_overwrites() {
      final var map = new MapCaseFile();
      map.put("x", "first");
      map.put("x", "second");
      assertThat(map.get("x", String.class)).isEqualTo("second");
    }

    @Test
    @DisplayName("put null on absent key does not add the key — value absence is idempotent")
    void put_nullOnAbsentKey_doesNotAddKey() {
      // CaseContextImpl.set() uses Objects.equals(prev, value); when prev==null and
      // value==null the condition is false so no write occurs. This is a migration
      // behaviour difference from the poc — document it explicitly.
      final var map = new MapCaseFile();
      map.put("k", null);
      assertThat(map.contains("k")).isFalse();
      assertThat(map.get("k", String.class)).isNull();
    }

    @Test
    @DisplayName("get with type performs type coercion via Jackson")
    void get_typeCoercion() {
      final var map = new MapCaseFile();
      map.put("num", "42");
      assertThat(map.get("num", Integer.class)).isEqualTo(42);
    }
  }

  @Nested
  @DisplayName("keys")
  class Keys {

    @Test
    @DisplayName("keys() reflects all put keys")
    void keys_reflectsPuts() {
      final var map = new MapCaseFile();
      map.put("a", 1);
      map.put("b", 2);
      assertThat(map.keys()).containsExactlyInAnyOrder("a", "b");
    }

    @Test
    @DisplayName("keys() is empty before any put")
    void keys_emptyInitially() {
      assertThat(new MapCaseFile().keys()).isEmpty();
    }
  }

  @Nested
  @DisplayName("contains")
  class Contains {

    @Test
    @DisplayName("contains returns true after put")
    void contains_afterPut() {
      final var map = new MapCaseFile();
      map.put("present", "value");
      assertThat(map.contains("present")).isTrue();
    }

    @Test
    @DisplayName("contains returns false for missing key")
    void contains_missingKey() {
      assertThat(new MapCaseFile().contains("absent")).isFalse();
    }
  }

  @Nested
  @DisplayName("inherited behaviour")
  class Inherited {

    @Test
    @DisplayName("putIfAbsent does not overwrite existing value")
    void putIfAbsent_doesNotOverwrite() {
      final var map = new MapCaseFile();
      map.put("k", "original");
      map.putIfAbsent("k", "replacement");
      assertThat(map.get("k", String.class)).isEqualTo("original");
    }

    @Test
    @DisplayName("snapshot is an independent copy")
    void snapshot_isIndependent() {
      final var map = new MapCaseFile();
      map.put("k", "v1");
      final var snap = map.snapshot();
      map.put("k", "v2");
      assertThat(snap.getString("k")).isEqualTo("v1");
      assertThat(map.getString("k")).isEqualTo("v2");
    }
  }
}
```

- [ ] **Step 2: Run tests — expect compilation failure (MapCaseFile does not exist yet)**

```bash
cd /Users/mdproctor/claude/casehub/engine
mvn install -DskipTests -q
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest=MapCaseFileTest 2>&1 | tail -20
```

Expected output contains: `cannot find symbol` or `class MapCaseFile does not exist`

---

### Task 2: Implement MapCaseFile

**Files:**
- Create: `runtime/src/main/java/io/casehub/engine/context/MapCaseFile.java`

- [ ] **Step 1: Create the implementation**

```java
/*
 * Copyright 2026-Present The Case Hub Authors
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 * http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
package io.casehub.engine.context;

import io.casehub.engine.internal.context.CaseContextImpl;
import java.util.Map;
import java.util.Set;

/**
 * Migration shim for code written against the casehub-poc {@code CaseFile} API.
 *
 * <p>Bridges the naming gap between the poc's {@code put/get/keys} and the engine's
 * {@code set/getAs/getKeys}. Intended as a stepping-stone; once migration is complete
 * call sites should use {@link io.casehub.api.context.CaseContext} directly.
 *
 * <p><b>Null behaviour:</b> {@code put(key, null)} on an absent key is a no-op — the
 * key is not added. This differs from the poc's {@code HibernateCaseFile} which stored
 * null entries directly. Use {@code contains(key)} to test presence.
 */
public class MapCaseFile extends CaseContextImpl {

  public MapCaseFile() {}

  public MapCaseFile(final Map<String, Object> initial) {
    super(initial);
  }

  /** poc-compatible alias for {@link #set(String, Object)}. */
  public void put(final String key, final Object value) {
    set(key, value);
  }

  /**
   * poc-compatible alias for {@link #getAs(String, Class)}.
   *
   * <p>Returns {@code null} when the key is absent — not {@code Optional.empty()}.
   * See platform protocol PP-20260512-5f055d.
   */
  public <T> T get(final String key, final Class<T> type) {
    return getAs(key, type);
  }

  /** poc-compatible alias for {@link #getKeys()}. */
  public Set<String> keys() {
    return getKeys();
  }
}
```

- [ ] **Step 2: Run tests — expect all to pass**

```bash
cd /Users/mdproctor/claude/casehub/engine
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest=MapCaseFileTest 2>&1 | tail -20
```

Expected output:
```
[INFO] Tests run: 12, Failures: 0, Errors: 0, Skipped: 0
[INFO] BUILD SUCCESS
```

- [ ] **Step 3: Run full runtime test suite to check for regressions**

```bash
cd /Users/mdproctor/claude/casehub/engine
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime 2>&1 | tail -20
```

Expected: `BUILD SUCCESS` with no new failures.

---

### Task 3: Commit

- [ ] **Step 1: Invoke `superpowers:requesting-code-review` before committing**

- [ ] **Step 2: Commit (after review passes)**

```bash
cd /Users/mdproctor/claude/casehub/engine
git add runtime/src/main/java/io/casehub/engine/context/MapCaseFile.java \
        runtime/src/test/java/io/casehub/engine/context/MapCaseFileTest.java
git commit -m "feat: add MapCaseFile migration shim (Closes #239)

MapCaseFile extends CaseContextImpl, adding put/get/keys aliases
to ease migration from casehub-poc's CaseFile API.

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```
