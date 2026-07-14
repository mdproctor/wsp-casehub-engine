# Every Boundary Typed

The ContextBridge protocol landed for workers a few weeks ago. It gave us typed input — `.<AmlTransaction>fn().apply(txn -> txn.riskScore())` — with compile-time safety and runtime bridge resolution. Good foundation. But signals were still `Map<String, Object>` going into the working layer, and SubCase context passing was hardcoded JQ with no lambda path. Two of the five boundary points were untyped. That inconsistency was rippling through the platform.

Claude and I closed both gaps in a single session. The result: `SignalType<T>` as a platform-level concept, `SubCaseMapping` as a sealed interface with expression-engine-aware dispatch, and a serialisation principle I hadn't articulated before this work forced me to.

## Signals aren't paths

The original ContextBridge spec projected a simple typed signal overload: `<T> signal(caseId, T data)`. That was wrong, and the design review caught why. A signal without a name gives the handler no idea where to write the payload in the working layer. More fundamentally — signals shouldn't be ad-hoc path writes. They should be declared at the platform level, the same way capabilities are.

`SignalType<T>` is a record with a name and a payload type. `CaseDefinition` declares which signals a case accepts. The runtime validates both the name and the type before publishing. Undeclared signal names are rejected with `SignalRejectedException`. The untyped `signal(caseId, path, value)` path still works — it's a different API for a different purpose, not a deprecated predecessor.

The key insight came from a brainstorming question where I pushed back on the assumption that JQ was the receiver. Every boundary should know its types. Signals are platform concepts; cases integrate with them.

## The serialisation boundary rule

Halfway through the design, I realised we were about to push POJOs through unnecessary Jackson round-trips. The worker boundary serialises at scheduling time because there's a genuine storage gap — EventLog persists the payload for Quartz recovery. But for same-JVM signals and SubCase spawning, there's no storage gap. The object passes directly through the Vert.x event bus as a Java reference.

The rule: `bridge.serialise()` is called only at storage boundaries (EventLog, database) and wire boundaries (Qhorus, HTTP). `bridge.deserialise()` is called only when reconstructing from stored or received data. Everything else passes as a POJO. The bridge has two roles — typing and serialisation — and they happen at different points.

For SubCase: the input mapping produces a typed object, passes it directly to `startCase()`, and `CaseContextImpl` wraps it. Serialisation happens only when EventLog records the event. For in-memory deployments, that's a tree-to-tree operation, not a full JSON round-trip. I can imagine a test that passes an object through every internal path and throws if serialisation is ever attempted — that would enforce the contract structurally.

## SubCaseMapping and the expression engine

SubCase input/output mappings were `String` fields holding JQ expressions, evaluated by direct `jqEvaluator.eval()` calls. Two problems: no lambda path for the Java DSL, and the JQ was hardcoded rather than going through `ExpressionEngineRegistry`.

`SubCaseMapping` is a sealed interface with two permits: `Expression(String)` and `Lambda(Function<CaseContext, Object>)`. The handler dispatches with a switch — exhaustive, compile-time checked. The `String` builder overload wraps in `Expression` transparently, so existing YAML definitions and test code don't break.

The Lambda path bypasses `ExpressionEngineRegistry` entirely. It's not a different expression engine — it IS the implementation. There's no expression string to parse, no engine to dispatch to. This mirrors how `LambdaExpressionEvaluator` works for boolean conditions, except mapping lambdas are `Function<CaseContext, Object>` rather than `Predicate<CaseContext>`.

Output mapping recovery was the interesting puzzle. Expression mappings store the JQ string in EventLog metadata — straightforward. Lambda functions can't be serialised. The solution: store `bindingName` in the metadata, look up the parent's `CaseDefinition` via `CaseDefinitionRegistry` at completion time, find the binding, get the `SubCase`, get the lambda. The definition is the source of truth — the function lives there, not in the event stream.

## What this unblocks

Two of the four remaining ContextBridge boundary issues are now closed (#689 WorkItem and #692 Connector remain). The signal work directly unblocks connectors — they produce signals, so typed signals had to land first. The serialisation boundary rule applies to all future boundary work and prevents the same "unnecessary round-trip" mistake from recurring.

The platform now has typed data at the worker boundary, the signal boundary, and the SubCase boundary. Three of five. The pattern is established and the remaining two follow the same protocol.
