---
title: "A2A Outbound: Your Case Workflows Can Now Invoke Remote AI Agents"
date: 2026-08-02
tags: [a2a, agent-interop, worker-execution, casehub-engine]
---

# A2A Outbound: Your Case Workflows Can Now Invoke Remote AI Agents

The agent ecosystem has a fragmentation problem. You've built your case workflows, wired up your local AI workers, tuned your routing strategies — and then someone tells you the best agent for a particular capability lives on a different server, behind a different API, run by a different team. So you write a custom HTTP client, handle retries yourself, bolt on auth, figure out how to get the result back into your case context, and hope the error handling covers the edge cases. For every remote agent.

We just eliminated that entire layer.

## What shipped

`casehub-engine-a2a` is a new optional module that lets you invoke remote A2A-compliant agents as native casehub workers. Drop it on the classpath, add an `a2a:` block to your YAML case definition, and the remote agent participates in your case execution pipeline exactly like a local worker — same routing, same retry policies, same outcome handling, same EventLog provenance.

```yaml
workers:
  - name: remote-analyst
    capabilities: [anomaly-detection]
    a2a:
      endpoint: https://analyst-agent.example.com
      skill: anomaly-detection
      streaming: true
      auth:
        type: bearer
        tokenConfigKey: analyst.token
```

That's it. No HTTP client code. No retry logic. No custom error handling. The engine's existing infrastructure handles all of it.

## Why A2A matters for case management

A2A (Agent-to-Agent) is the Google-initiated, Linux Foundation-governed protocol for agent interoperability. It's horizontal — agent-to-agent communication across platforms and organisational boundaries. Over 150 organisations have adopted it. Azure AI Foundry, Amazon Bedrock AgentCore, and Google Cloud all have native A2A support.

For case management, this is the difference between "our agents" and "all agents." A financial crime case might need a KYC verification agent run by a partner institution. A clinical trial case might need a pharmacovigilance agent hosted by a regulatory body. A DevOps case might need a security scanning agent from a third-party provider. None of these will ever be local workers in your JVM — they're remote, independently managed, and behind their own auth boundaries.

A2A gives them a standard protocol. casehub-engine-a2a gives them a standard integration point.

## How it works under the hood

We chose to implement A2A as a `WorkerFunctionHandler` — the same execution model as local sync workers and Serverless Workflow steps — rather than the `WorkerProvisioner` path used for async external agents. The reasoning: A2A is fundamentally request/response. You send a message, you get a result. That maps cleanly to a virtual thread blocking on an HTTP call with timeout enforcement, not to the provisioner's async channel-based delivery model.

The architecture is four classes:

- **`A2AWorkerFunction`** — an immutable record carrying endpoint config. Implements `WorkerFunction<Map, Map>`. No A2A transport types leak into the record.
- **`A2AClient`** — thin wrapper over JDK `HttpClient` speaking A2A JSON-RPC. `message/send` for sync, `message/stream` for SSE streaming. ~160 lines — the A2A wire protocol is genuinely small.
- **`A2AClientRegistry`** — one HTTP client per remote endpoint, lazily created, connection reuse across workers targeting the same server. Auth conflict detection. Graceful shutdown.
- **`A2AWorkerFunctionHandler`** — the CDI bean that ties it together. Dispatches on virtual threads with timeout enforcement. Maps A2A task states to `WorkerResult` outcomes. Threads protocol metadata (endpoint, taskId, messageId) through to EventLog entries via a new `HandlerResult` type.

The error handling distinguishes transient failures (connection errors, HTTP 5xx, 401, 429) from semantic failures (agent returned FAILED, unknown skill, protocol errors). Transient failures propagate as exceptions — `QuartzRetryService` retries with backoff against the same endpoint. Semantic failures return `WorkerResult.failed()` — `OutcomePolicy` can reroute to a different agent. This distinction is critical: retrying a semantic failure wastes time; rerouting a transient failure misses the point.

## HandlerResult — a cross-cutting improvement

Building A2A support surfaced a gap in the engine: `WorkerFunctionHandler.execute()` returned `WorkerResult<?>`, which is a public API type in `casehub-worker-api`. A2A needs to carry protocol metadata (endpoint URL, remote task ID, streaming status) through to EventLog entries, but adding that to `WorkerResult` would pollute the consumer-facing API with engine internals.

The fix: `HandlerResult` — a new record in `engine-common` that wraps `WorkerResult` with an optional `Map<String, Object> protocolMetadata`. The handler interface now returns `HandlerResult`. Existing handlers wrap with empty metadata (zero-cost change). Handlers that speak external protocols (A2A today, MCP next) populate the metadata map, and it threads through the executor → Quartz job → completion handler → EventLog chain automatically.

This is the kind of design improvement that only surfaces when you wire a real protocol through the stack. The abstraction was missing because no handler needed it until now.

## What's next

This is the first half of engine#830/#831 — A2A outbound is functionally complete (6 of 8 implementation tasks done), with MCP tool integration to follow. The integration test and CLAUDE.md documentation are the remaining tasks.

Beyond v1, there's a follow-on epic (engine#835) tracking deeper platform integration: mapping A2A AgentCards to eidos AgentDescriptor for subsumption matching and personality routing, health probing via the A2A health endpoint, vocabulary grounding for semantic matching.

The longer-term vision is semantic runtime worker discovery — instead of statically declaring which workers handle which capabilities, the engine discovers workers at dispatch time from an embedding-based index over thousands of registered agents. A2A and MCP discovery federation feeds that index. The `WorkerDiscoveryProvider` SPI (engine#841) is the foundational seam. But that's a design conversation for another day.

For now: add `casehub-engine-a2a` to your classpath, point it at a remote agent, and your cases can reach the entire A2A ecosystem.
