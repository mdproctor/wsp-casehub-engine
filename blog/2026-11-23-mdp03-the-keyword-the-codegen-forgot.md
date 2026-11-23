---
entry_type: diary
subtype: note
projects:
  - casehubio/engine
date: 2026-11-23
---

# The keyword the codegen forgot

CaseHub's YAML record codegen generates Java records from a schema definition
and a mapping file. When a schema field name collides with a Java keyword — `do`
is the real example — the mapping renames it: `do: { name: doBlock, type: JsonNode }`.
The generated record gets a `doBlock` component, and the YAML stays clean.

Except Jackson doesn't know about the rename. It deserialises `do:` from the
YAML, looks for a `do` component on the record, doesn't find one, and silently
drops the value. The `doBlock` field arrives as null. No error, no warning.

This is exactly the kind of bug that integration tests catch and unit tests
miss. The codegen module compiles cleanly. The generated record compiles cleanly.
The null only surfaces at runtime when a real YAML document hits the mapper —
and then only if the test actually asserts on the value. If the field is optional
or the test is checking something else, the null sails through.

The fix is a single conditional in `RecordEmitter.buildAnnotations()`: when the
schema name differs from the Java field name and the schema name is a Java
keyword, auto-emit `@JsonProperty("do")` on the generated component. The
annotation tells Jackson to bind the YAML key to the renamed field.

The keyword guard matters. Not every rename is a keyword escape — some renames
simplify the DSL. `defaultQuorum` becomes `quorum` because users should write
`quorum:` in their YAML, not `defaultQuorum:`. Auto-emitting
`@JsonProperty("defaultQuorum")` on that field would break every existing
document that uses the shorter name. The first version of the fix did exactly
that — nine tests failed across the quorum and GOAP deserialisers before the
guard was added.

The root cause surfaced during the investigation of #1336, where a cascade of
three bugs traced back to `doBlock` arriving as null. The codegen now prevents
the class of bug entirely for keyword escapes. For non-keyword renames, the
existing `property:` override in the mapping file remains the right mechanism —
it's explicit, and the rename is intentional.
