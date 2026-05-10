# ADR 0014: Postgres as primary storage; OpenSearch as retrieval optimization only

- **Status**: Accepted
- **Date**: 2026-05-10

## Context

The platform has two stateful tiers that can both store text-shaped data: Postgres (state, checkpoints, the LibreChat database, Letta-resident memory, the audit `audit_events` table per ADR 0034) and OpenSearch (vector + hybrid search, RAG indexes including the Knowledge Base, advisory audit search index). Without an explicit rule, it is tempting to treat OpenSearch as authoritative for derived artifacts — embeddings, audit indexes, search-over-checkpoints — because it is what serves the read path. Architecture-overview.md §6.3 deliberately separates these roles and ADR 0034 establishes the canonical audit pipeline: Postgres + S3 are the durable system of record, and OpenSearch is an advisory fanout rebuildable from those primaries. Architecture-backlog.md §6 elevates this separation to a platform invariant that several other components (Letta, the audit pipeline, the Knowledge Base, DR procedures in Workstream F) depend on.

Postgres itself runs in two modes consistent with the platform's dual-mode hosting rule (ADR 0033): **in-cluster on kind** for developer laptops and integration testing, and as **AWS RDS provisioned via a Crossplane MR/XRD** on AWS deployments. Consuming components see only a Postgres connection string supplied through the standard secret/config path; the dual-mode is invisible above the connection. The system-of-record commitment in this ADR is what makes that invisibility safe: components depend on Postgres semantics, not on the hosting shape.

## Decision

The platform treats **Postgres (and, where appropriate, object storage or an external system of record) as the primary store for all stateful data**, and **OpenSearch as a retrieval-optimization layer only**. This is the architecture-backlog.md §6 invariant: *"Anything in OpenSearch must be reproducible from a primary source (Postgres, object storage, or external system."* OpenSearch is never the system of record; every index in it must have a documented rebuild path from a primary. Components that write platform state write to a primary first, and any OpenSearch index is derived from that primary either synchronously (dual-write) or by reindex.

## Consequences

- **OpenSearch is rebuildable from primaries.** Vectors and embeddings are re-derived from source documents in object storage; audit indexes are re-derived from the object-storage audit archive; any search-over-checkpoint indexes (if added) are re-derived from Postgres (architecture-overview.md §6.3). Reindex paths are first-class operational procedures, not emergency improvisation.
- **Simpler DR posture.** Disaster recovery focuses on managed-Postgres restore (PITR, replication via the cloud provider per §6.3) and object-storage durability. OpenSearch reindex from primaries is the recovery path, not an OpenSearch backup/restore loop. This is the shape exercised by Workstream F2 (architecture-backlog.md §1, F2 — DR testing).
- **Write paths must hit a primary first.** Any new component holding state writes to Postgres or object storage (or an external system of record) before — or instead of — writing to OpenSearch. A component that writes only to OpenSearch is by definition violating the invariant and must be redesigned.
- **Audit pipeline rides on this invariant (ADR 0034).** The audit pipeline's system of record is Postgres + S3: the audit endpoint writes received events into a Postgres `audit_events` table, and on AWS a periodic batch job copies pending rows to S3 and deletes them from Postgres only after the S3 write is verified. On kind, S3 is not provisioned and Postgres alone is the system of record — the durability of the audit pipeline on kind rests entirely on Postgres being a primary, not a cache. The OpenSearch audit index is advisory fanout and is rebuildable from S3 on AWS or from Postgres on kind. The reproducibility-from-primaries property (architecture-backlog.md §6) therefore holds equally for audit indexes, KB indexes, Letta-derived indexes, and test-result indexes (ADR 0011): the rebuild source differs, but every OpenSearch index has one.
- **Test result streaming follows the same shape (ADR 0011).** Non-unit test runs stream per-run results to OpenSearch as an advisory index and emit audit events through the audit adapter. The OpenSearch test-result index is derived; the audit trail of "who ran what test against which environment" lives in the audit pipeline's primary store.
- **Letta-maintained indexes are derived.** Letta's Postgres state is primary; its OpenSearch-resident retrieval indexes are derived and rebuildable.
- **Knowledge Base content is owned by its source.** RAGStore content originates from Git (platform docs, runbooks) or from the vendor-doc companion project (ADR 0024); OpenSearch holds the index, not the source of truth. Re-indexing on major/minor releases (architecture-overview.md §6.4) is the rebuild path.
- **Migration risk for OpenSearch is bounded.** Because every index has a primary-source rebuild path, replacing or upgrading OpenSearch (ADR 0009) is a reindex exercise rather than a data-migration exercise.

## References

- [architecture-overview.md](../architecture-overview.md) [§ 6.3](../architecture-overview.md#63-memory-and-data-architecture), [§ 6.5](../architecture-overview.md#65-observability-architecture)
- [architecture-backlog.md § 6](../architecture-backlog.md#6-architecture-level-invariants-worth-documenting-as-adrs)
- [ADR 0009](./0009-opensearch-search-vector-store.md) (OpenSearch)
- [ADR 0011](./0011-three-layer-testing-cli-orchestration.md) (three-layer testing — test results streamed to OpenSearch as advisory index)
- [ADR 0033](./0033-initial-implementation-targets-aws-github.md) (initial implementation targets — dual-mode hosting: kind locally, AWS RDS via Crossplane on AWS)
- [ADR 0034](./0034-audit-pipeline-durable-adapter.md) (audit pipeline — Postgres + S3 system of record, OpenSearch advisory only)
