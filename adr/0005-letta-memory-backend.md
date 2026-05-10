# ADR 0005: Letta as the memory backend

- **Status**: Accepted
- **Date**: 2026-05-10

## Context

Platform Agents need a durable memory service consumed through ARK's `Memory` CRD and surfaced to agent code via the Platform SDK's `memory.*` API (architecture-overview.md §6.3).
The backend must persist state into the platform's Postgres system of record, keep any retrieval-optimized indexes in OpenSearch reproducible from primary sources, and be provisionable as a Crossplane v2 `MemoryStore` XR.
It must also support the per-store access modes (private, namespace-shared, RBAC/OPA-controlled) that the architecture treats as an invariant (architecture-backlog.md §6) and remain fully OSS so the platform can self-host without a vendor dependency.
Five OSS options were evaluated (architecture-backlog.md §2.9).

## Decision

The platform adopts **Letta** as the memory backend (component **A10**, architecture-overview.md §5, §14.1). Letta runs as a memory service behind ARK's `Memory` CRD, persists state to managed Postgres (§6.3, §9), and is provisioned through the `MemoryStore` Crossplane XR (B4). The Platform SDK's `memory.*` API (B6) is the only path Platform Agents use to reach it, and a thin memory backend adapter (B11) wraps Letta-specific calls.

## Alternatives considered

- **Mem0** — Rejected under the collective rationale in architecture-backlog.md §2.9 (Letta wins on fully OSS posture and OS-style tiered memory model).
- **Letta** chosen — fully OSS, OS-style tiered memory model.
- **Zep / Graphiti** — Rejected under the collective rationale in architecture-backlog.md §2.9.
- **Cognee** — Rejected under the collective rationale in architecture-backlog.md §2.9.
- **LangMem** — Rejected under the collective rationale in architecture-backlog.md §2.9.

## Consequences

- Letta is consumed exclusively through ARK's `Memory` CRD and the Platform SDK's `memory.*` API; agent code does not call Letta directly (architecture-overview.md §6.2, §6.3).
- Letta state lives in managed Postgres (RDS / Cloud SQL / Azure Database for Postgres), inheriting the platform's backup, PITR, and DR posture (§6.3, §6.13).
- Retrieval indexes that Letta maintains in OpenSearch must remain reproducible from Postgres or object storage per the OpenSearch invariant (§6.3); OpenSearch is never a system of record for memory.
- The per-store **memory access modes** (private, namespace-shared, RBAC/OPA-controlled) are declared on the `MemoryStore` XR; the detailed enforcement semantics across modes — including how the RBAC-as-floor / OPA-as-restrictor model applies to memory reads and writes — are deferred and tracked under ADR 0025.
- Letta multi-tenancy gaps are mitigated by wrapping it behind the `Memory` CRD and running namespace-scoped instances, consistent with the OSS-limitations posture in §9.
- The Platform SDK ships a version-pinned compatibility matrix against Letta versions alongside the gateway and ARK pins (§6.13); upgrades are coordinated through Workstream A.
- Letta lifecycle events flow under the `platform.lifecycle.*` CloudEvent namespace via `MemoryStore` lifecycle (§6.7), and Letta-backed resources (`Memory`, `MemoryStore`) are admission-gated by Gatekeeper (§6.6).
- A migration off Letta would require re-implementing the B11 memory backend adapter; the SDK API contract insulates Platform Agents from that change.

## References

- [architecture-overview.md § 6.3](../architecture-overview.md#63-memory-and-data-architecture)
- [architecture-backlog.md](../architecture-backlog.md) [§ 2.9](../architecture-backlog.md#29-memory-backend-mem0-vs-letta-vs-zep--graphiti-vs-cognee-vs-langmem), [6](../architecture-backlog.md#6-architecture-level-invariants-worth-documenting-as-adrs)
- [ADR 0025](./0025-memory-access-modes-per-store.md) (memory access modes per store) — directly related
