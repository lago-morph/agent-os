# ADR 0005: Letta as the memory backend

## Status

Accepted

## Context

Platform Agents need durable, structured memory beyond what fits in an LLM context window: per-conversation scratch state, longer-term agent recollection, and selectively shared knowledge across agents. The architecture (overview §6.3) treats memory as a first-class primitive: agents reach it through the Platform SDK `memory.*` API, ARK exposes it declaratively as the `Memory` CRD, and Crossplane v2 provisions instances via a `MemoryStore` XR. Storage roles are deliberately separated — Postgres is the system of record for state and checkpoints, OpenSearch is a retrieval-optimization layer, and object storage is the immutable archive (overview §6.3 invariant: anything in OpenSearch must be reproducible from a primary source).

Memory access modes (overview §6.3, ADR 0025) — private, namespace-shared, and RBAC/OPA-controlled — are declared per `MemoryStore` and enforced by the platform. Whatever backend we adopt has to fit underneath that abstraction without forcing its own access model on top.

The candidates evaluated were Mem0, Letta, Zep / Graphiti, Cognee, and LangMem (backlog §2.9). Selection criteria: fully open-source license (no enterprise-only feature gating for capabilities we need), tiered memory model that maps cleanly onto our SDK and CRD shape, ability to persist authoritative state in our managed Postgres while letting OpenSearch handle retrieval, and operational fit for a kopf/Crossplane-shaped install.

## Decision

Adopt **Letta** as the memory backend for Platform Agents.

- Agent code calls memory only through the Platform SDK `memory.*` API; the SDK is the single client of Letta.
- ARK's `Memory` CRD is the declarative surface; `MemoryStore` XRs (Crossplane v2) provision Letta-backed stores and wire access mode, namespace, and credentials.
- Letta persists authoritative state to **Postgres** (overview §6.3, §14: managed RDS / Cloud SQL / Azure Database for Postgres alongside LibreChat DB and long-running checkpoints).
- Retrieval — vector and hybrid search over memory content — runs through **OpenSearch** (per ADR 0009). OpenSearch indexes are reproducible from Postgres + object storage; they are not a system of record.
- A small **memory adapter** (backlog item B11) sits between the SDK and Letta to map our access modes, namespacing, and audit emission onto Letta's native API. Letta itself is consumed unmodified.

## Consequences

Positive:

- Fully OSS license — no enterprise feature paywall on the memory path; aligns with the platform-wide OSS-first posture.
- OS-style tiered memory (working / archival / recall analogues) maps naturally onto the `memory.*` SDK without forcing a custom abstraction.
- Postgres-backed state lets us reuse the platform's existing managed-Postgres operations: backup, point-in-time recovery, replication, DR rehearsal (overview §14.6, Workstream F).
- OpenSearch as the retrieval layer keeps the storage-roles invariant intact and shares infrastructure with audit and RAG retrieval.
- The `MemoryStore` XR lets us stand up namespace-scoped Letta instances, isolating tenants and matching the multi-tenancy model (ADR 0016).

Negative / costs:

- Letta is a younger project than some alternatives; we accept upgrade churn and possible API drift, contained behind the SDK + adapter.
- Multi-tenant features in Letta may need custom configuration; we mitigate by running namespace-scoped instances behind ARK's `Memory` CRD (overview §22 risk register).
- The adapter is custom code we own; keeps the SDK contract stable but is one more component to maintain.

## Alternatives considered

- **Mem0** — capable, but key features sit behind a managed/enterprise tier that conflicts with our OSS-first posture.
- **Zep / Graphiti** — strong knowledge-graph framing; heavier than what the SDK contract requires and not as clean a fit for Postgres-as-primary + OpenSearch-as-retrieval.
- **Cognee** — interesting graph-plus-vector approach; less mature deployment story for a Kubernetes-native install with managed Postgres.
- **LangMem** — tightly coupled to a specific agent SDK trajectory; we want the memory backend independent of agent-SDK choice (see ADR 0019 on Langchain Deep Agents as the v1.0 SDK with multi-SDK shape preserved).

Rationale recorded in backlog §2.9: chose Letta on fully-OSS license and OS-style tiered memory model.

## Related

- ADR 0001 — ARK as the agent operator (provides the `Memory` CRD).
- ADR 0009 — OpenSearch as the search/vector store (memory retrieval path).
- ADR 0014 — Postgres as primary storage, OpenSearch as retrieval optimization only.
- ADR 0025 — Memory access modes (private / namespace-shared / RBAC-OPA-controlled) declared per `MemoryStore`.
- Architecture overview §5 (component inventory), §6.3 (memory and data architecture), §14 (workstreams, including B11 memory adapter).
- Architecture backlog §2.9 (alternatives), §6 (invariants — memory access modes; OpenSearch reproducibility), §7 (ADR candidates).
