# ADR 0014: Postgres as primary storage, OpenSearch as retrieval optimization

## Status

Accepted

## Context

The platform has three distinct storage roles: a system of record for
relational state and checkpoints, a retrieval-optimization layer for vector
and full-text search, and an immutable archive for long-term retention and
rebuild. The architecture overview (§4 and §6.3) assigns these to Postgres,
OpenSearch, and object storage respectively, but the role split itself is the
load-bearing decision and deserves its own ADR — separate from the product
selection in ADR 0009 (OpenSearch) and the consumers in ADR 0005 (Letta) and
ADR 0007 (LibreChat).

State that needs relational semantics, transactions, and recoverable backups
includes the LibreChat DB, Letta's durable memory state, long-running Platform
Agent checkpoints, and any other application state custom components persist.
OpenSearch, by contrast, holds RAG vectors, the audit index, and Letta's
retrieval index — all of which can be regenerated from a primary source.

The backlog (§6) records the underlying invariant: **anything in OpenSearch
must be reproducible from a primary source (Postgres, object storage, or
external system).** This ADR makes that invariant binding and assigns each
storage role to a specific system.

## Decision

**Postgres is the system of record. OpenSearch is retrieval optimization
only. Object storage holds the immutable archive.**

Postgres holds:

- The LibreChat DB (conversations, users — see ADR 0007).
- Letta state (durable memory contents — see ADR 0005).
- Long-running Platform Agent checkpoints (workflow state, agent run state
  that must survive restarts).
- Any other relational state custom components require.

Use **managed Postgres in the host cloud** — RDS / Cloud SQL / Azure Database
for Postgres — with the provider's built-in backup, **point-in-time recovery**,
and replication. Each cluster install provisions its own Postgres instance via
Crossplane (per ADR 0026, independent-cluster-install). We do not run Postgres
in-cluster for production.

OpenSearch holds only derived data: RAG indexes, the audit index, Letta's
retrieval index. Loss of any OpenSearch index is a performance incident, not a
data-loss incident; rebuild procedures must exist for every index.

Object storage holds source documents (KB content, vendor docs) and the
immutable audit archive. The **audit index in OpenSearch is reproducible from
the object-storage archive** — the dual-write path (overview §6.3) exists
specifically to make this true.

The invariant is enforced by review: a component proposing to store
non-reproducible data in OpenSearch must show where the primary copy lives, or
the design is rejected.

## Consequences

- Disaster recovery is a Postgres DR drill plus an OpenSearch rebuild, not a
  multi-system data-recovery exercise. Managed Postgres PITR is the recovery
  path for state.
- Every component that writes to OpenSearch must also document where the
  primary copy lives and how to rebuild the index from it. This is a
  Workstream F deliverable (overview §14.6) for every index template shipped.
- Letta's split storage (state in Postgres, retrieval in OpenSearch) is
  explicit and the adapter is built that way — see ADR 0005.
- LibreChat's Postgres DB is part of the platform DR plan, not a LibreChat
  internal concern — see ADR 0007.
- OpenSearch sizing decisions are about query performance, not durability;
  retention can be tuned without data-loss risk.
- Cost: managed Postgres adds a per-cluster line item. Accepted as the price
  of a defensible DR story.

## Alternatives considered

- **OpenSearch as primary for some data.** Rejected. OpenSearch is operated as
  a search engine, not a transactional store; backup/restore is heavier and
  less granular than managed Postgres PITR, and mixing roles destroys the
  invariant that lets us rebuild indexes freely.
- **Document database (e.g., MongoDB) as primary.** Rejected. Letta state,
  checkpoints, and LibreChat all benefit from relational semantics and
  transactions. A document store would force application-level workarounds
  for joins and consistency.
- **Self-managed Postgres in-cluster.** Rejected for production. Loses
  managed-PITR, automated failover, and the operational maturity of cloud
  provider offerings. Acceptable for kind-based dev environments only.
- **Single store doing everything (e.g., Postgres with `pgvector` and full-
  text only).** Rejected — see ADR 0009. Audit and KB search at platform
  scale want a real search engine.

## Related

- ADR 0005 — Letta memory backend (state in Postgres, retrieval in OpenSearch).
- ADR 0007 — LibreChat locked-down frontend (LibreChat DB in Postgres).
- ADR 0009 — OpenSearch selection (this ADR is the role split; 0009 is the
  product choice).
- ADR 0026 — Independent-cluster install (Postgres is per-cluster, not shared).
- Overview §4 (high-level), §6.3 (memory and data — rule of thumb), §14
  (Workstream F covers DR drills and audit retention).
- Backlog §6 invariant: "Anything in OpenSearch must be reproducible from a
  primary source (Postgres, object storage, or external system)."
