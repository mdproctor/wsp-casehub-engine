---
layout: post
title: "Unblocking AML"
date: 2026-05-29
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-engine]
tags: [ledger, trust-routing, testcontainers, podman, cdi]
---

The batch that closed today was supposed to be an afternoon.

`TrustRoutingPolicy` and `TrustRoutingPolicyProvider` were living in `casehub-engine-ledger` — the implementation module. Any consumer needing to configure trust routing had to pull in the full ledger runtime to do it. AML wanted `AmlTrustRoutingPolicyProvider`, not `TrustWeightedAgentStrategy`, but you couldn't get one without the other. Moving the two types to `casehub-engine-api` fixed it — one IntelliJ semantic move, all import references updated.

The second AML blocker was more interesting. `TrustScoreJob` scores agents by reading `LedgerEntry` records grouped by `actorId`. Post-investigation attestations — the SAR-was-FLAGGED verdict — need a `LedgerEntry` to reference. Without it, attestations float free of any decision. Worker completions weren't writing one.

The fix is `WorkerDecisionEntry` — a JOINED `LedgerEntry` subclass with explicit `workerId` and `capabilityTag` columns, written on each successful worker execution via a CDI `@ObservesAsync` observer. The code review caught a correctness problem straight away. The observer computing the next sequence number was calling `findLatestByCaseId()`:

```java
// Before — only sees CaseLedgerEntry rows for this case
final int seq = ledgerRepo.findLatestByCaseId(event.caseId())
    .map(e -> e.sequenceNumber + 1).orElse(1);

// After — spans all LedgerEntry subtypes for this subjectId
final int seq = ledgerRepo.findLatestBySubjectId(event.caseId())
    .map(e -> e.sequenceNumber + 1).orElse(1);
```

`findLatestByCaseId()` only queries `CaseLedgerEntry`. Once a `WorkerDecisionEntry` exists for the same case, both observers independently compute `seq + 1` from the same max. Silent Merkle chain corruption. Claude caught it. We switched both capture observers to `findLatestBySubjectId()`.

The Podman detour cost an hour. The test suite was failing with `ClosedConnectionException` across every `@QuarkusTest` class — looked like the database container dying. It was: Podman had only 2 GB and was OOM-killing it. We bumped to 4 GB, but the restart broke the Docker API socket. The `gvproxy` process that creates `podman-machine-default-api.sock` needs a few seconds after the machine starts — the `~/.testcontainers.properties` was pointing at a path that didn't exist yet. A curl check against the socket confirmed it was absent:

```
UnixSocketClientProviderStrategy: failed with exception InvalidConfigurationException
  (Could not find unix domain socket). Root cause NoSuchFileException
```

Updated the properties file once the socket appeared and tests ran clean. The permanent fix is `podman-mac-helper install` — creates a stable `/var/run/docker.sock` that survives restarts — but that requires root.

The rest of the batch was smaller. `EmbeddingCache` is a `LinkedHashMap` with `accessOrder=true` and `removeEldestEntry` — avoids re-embedding agent vocabulary on every routing decision without adding a dependency. `LangChain4jAgentEmbeddingProvider` is eight lines delegating to Quarkus's `EmbeddingModel`. `CaseMemoryObserver` captures case completions into `CaseMemoryStore` for any consumer that wants persistent memory without wiring it explicitly.

Eight commits, one PR.
