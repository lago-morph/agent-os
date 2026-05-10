# ADR 0014: Postgres as primary storage; OpenSearch as retrieval optimization only

- **Status**: Accepted
- **Date**: 2026-05-10

## Context

The platform has two stateful tiers that can both store text-shaped data: Postgres (state, checkpoints, the LibreChat database, Letta-resident memory) and OpenSearch (vector + hybrid search, RAG indexes including the Knowledge Base, audit search). Without an explicit rule, it is tempting to treat OpenSearch as authoritative for derived artifacts — embeddings, audit indexes, search-over-checkpoints — because it is what serves the read path. Architecture-overview.md §6.3 deliberately separates these roles and §6.5 routes audit through OpenSearch for short-term searchable retention while object storage holds the long-term archive. Architecture-backlog.md §6 elevates this separation to a platform invariant that several other components (Letta, the audit pipeline, the Knowledge Base, DR procedures in Workstream F) depend on.

## Decision

The platform treats **Postgres (and, where appropriate, object storage or an external system of record) as the primary store for all stateful data**, and **OpenSearch as a retrieval-optimization layer only**. This is the architecture-backlog.md §6 invariant: *"Anything in OpenSearch must be reproducible from a primary source (Postgres, object storage, or external system."* OpenSearch is never the system of record; every index in it must have a documented rebuild path from a primary. Components that write platform state write to a primary first, and any OpenSearch index is derived from that primary either synchronously (dual-write) or by reindex.

## Consequences

- **OpenSearch is rebuildable from primaries.** Vectors and embeddings are re-derived from source documents in object storage; audit indexes are re-derived from the object-storage audit archive; any search-over-checkpoint indexes (if added) are re-derived from Postgres (architecture-overview.md §6.3). Reindex paths are first-class operational procedures, not emergency improvisation.
- **Simpler DR posture.** Disaster recovery focuses on managed-Postgres restore (PITR, replication via the cloud provider per §6.3) and object-storage durability. OpenSearch reindex from primaries is the recovery path, not an OpenSearch backup/restore loop. This is the shape exercised by Workstream F2 (architecture-backlog.md §1, F2 — DR testing).
- **Write paths must hit a primary first.** Any new component holding state writes to Postgres or object storage (or an external system of record) before — or instead of — writing to OpenSearch. A component that writes only to OpenSearch is by definition violating the invariant and must be redesigned.
- **Audit dual-write is the canonical pattern.** Audit events go to OpenSearch for short-term searchable retention *and* to object storage (S3 or equivalent) for long-term archive (architecture-overview.md §6.5). Object storage is the primary; OpenSearch is the searchable derivative. Retention policy and lifecycle rules are deferred to Workstream F (architecture-backlog.md §1.13), but the dual-write path is committed by this ADR.
- **Letta-maintained indexes are derived.** Letta's Postgres state is primary; its OpenSearch-resident retrieval indexes are derived and rebuildable.
- **Knowledge Base content is owned by its source.** RAGStore content originates from Git (platform docs, runbooks) or from the vendor-doc companion project (ADR 0024); OpenSearch holds the index, not the source of truth. Re-indexing on major/minor releases (architecture-overview.md §6.4) is the rebuild path.
- **Migration risk for OpenSearch is bounded.** Because every index has a primary-source rebuild path, replacing or upgrading OpenSearch (ADR 0009) is a reindex exercise rather than a data-migration exercise.

## References

- architecture-overview.md § 6.3, § 6.5
- architecture-backlog.md § 6
- ADR 0009 (OpenSearch)
