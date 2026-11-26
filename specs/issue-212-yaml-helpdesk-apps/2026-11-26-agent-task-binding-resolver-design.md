# AgentTaskBindingResolver — Design Spec

**Issue:** casehubio/eidos#280
**Branch:** issue-212-yaml-helpdesk-apps
**Scale:** S | **Complexity:** Med

## Problem

The engine resolves `agent-task:` bindings at execution time via `AgentWorkerFunctionHandler.executeByCapability()`. A misconfigured binding — referencing a capability no agent provides — only fails when the case reaches that worker step. There is no early detection.

## Solution

A CDI startup observer in `eidos-runtime/resolver/` that validates agent-task bindings at definition time by querying the `AgentRegistry`.

### Component

**`AgentTaskBindingResolver`** — `@ApplicationScoped`, observes `StartupEvent` at priority 2700.

Priority 2700 runs after:
- CDI bean registration (priority 10)
- Jandex scan (priority 150)
- YAML bootstrap (default priority)
- `YamlCaseHubDiscovery` (priority 2600)

### Behaviour

1. Inject `CaseDefinitionRegistry` and `AgentRegistry`
2. On startup, iterate `caseDefinitionRegistry.allDefinitions()`
3. For each `CaseDefinition`, iterate `getBindings()`
4. For each `Binding` whose `target()` is `AgentTaskTarget`:
   - Extract `capability()` and `domain()` from the target
   - Query `agentRegistry.find(AgentQuery.byCapabilityAndDomain(capability, domain, tenancyId))`
   - If no matches: log `WARN` with case name, binding name, capability, domain
   - If matches found: count as resolved
5. Log `INFO` summary: validated N bindings across M definitions, K resolved, J unresolved

### Constraints

- **Advisory only** — warnings, not hard failures. Agents may register dynamically after case definitions.
- **No caching** — one-shot startup validation. Runtime resolution continues to happen in `AgentWorkerFunctionHandler`.
- **No engine changes** — eidos depends on engine API, not the reverse. If eidos isn't on the classpath, no validation occurs (which is correct — agent-task bindings are meaningless without eidos).
- **Tenancy** — uses `CurrentPrincipal.tenancyId()` for the registry query, consistent with `EidosAgentResolver`.

### File Placement

```
eidos/runtime/src/main/java/io/casehub/eidos/runtime/resolver/AgentTaskBindingResolver.java
eidos/runtime/src/test/java/io/casehub/eidos/runtime/resolver/AgentTaskBindingResolverTest.java
```

### Test Cases

1. Binding resolves — agent exists for the capability/domain. No warning logged.
2. Binding unresolvable — no agent for the capability. Warning logged with case name, binding name, capability.
3. Definition with no agent-task bindings — skipped silently.
4. Multiple bindings across multiple definitions — each validated independently, summary counts correct.
5. Null/empty definitions list — no-op, no errors.

## References

- `engine/common-core/.../BindingExecutorResolver.java` — engine-side binding resolution
- `engine/common-core/.../SyntheticAgentTaskWorker.java` — synthetic worker creation from AgentTaskTarget
- `engine/runtime/.../AgentWorkerFunctionHandler.java:203-254` — execution-time capability resolution
- `engine/runtime/.../YamlCaseHubDiscovery.java` — startup observer precedent (priority 2600)
- `eidos/runtime/resolver/EidosAgentResolver.java` — existing eidos resolver pattern
- `eidos/api/AgentRegistry.java` — query API (`find`, `AgentQuery.byCapabilityAndDomain`)
- `engine/api/.../AgentTaskTarget.java` — binding target with capability/domain/model
