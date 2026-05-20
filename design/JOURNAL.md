# Design Journal — issue-300-deadline-in-command

### 2026-05-20 · §Dependencies and SPI

COMMAND content dispatched by `WorkerScheduleEventHandler.dispatchCommand()` now
carries an optional `deadline` field (ISO-8601 Instant) alongside the existing
`type`, `capability`, `correlationId`, and `input` fields. The deadline is
forwarded from `PropagationContext.getDeadline()` and is present only when the
case was started with a time budget. Consumers (claudony) use it to bound Qhorus
Commitment `expiresAt` — the content JSON is the only channel available, as
consumers have no access to the engine's `CaseInstance` at dispatch time.
The COMMAND content map is a de facto protocol; typed formalisation is tracked
in engine#301.
