# SPEC ADR-0005 — Letta as the memory backend [PROPOSED]

> kind: ADR · workstream: — · tier: T1
> upstream: [A11] · downstream: [B11] · adrs: [0005] · views: [6.3]
> canon-glossary: 6aadcc2a4f38 · canon-interface: 54f5ede58e5f

## 1. Purpose & Problem Statement
ADR 0005 fixes Letta as the memory backend behind ARK's `Memory` CRD. This SPEC states what honoring that decision requires: Platform Agents reach Letta only through ARK's `Memory` CRD and the Platform SDK's `memory.*` API (never directly); Letta state persists into managed Postgres as the system of record; any retrieval indexes Letta keeps in OpenSearch remain reproducible from primary sources; Letta is provisioned via the `MemoryStore` Crossplane XR; and a thin B11 adapter wraps Letta-specific calls. The decision is settled; this SPEC captures the obligations it imposes and the acceptance criteria that prove it is honored.

## 2. Scope
### 2.1 In scope
- Letta as component A10's memory service behind ARK's `Memory` CRD.
- The constraint: agent code reaches memory only via `Memory` CRD + SDK `memory.*` (no direct Letta calls).
- Letta state persisted to managed Postgres as system of record; OpenSearch indexes reproducible-only.
- Provisioning via the `MemoryStore` Crossplane XR (B4); B11 memory-backend adapter wraps Letta.
- SDK version-pinned compatibility matrix against Letta.

### 2.2 Out of scope (and where it lives instead)
- Letta install mechanics + deliverables — component A10 SPEC.
- Per-store memory access-mode enforcement semantics — ADR 0025 (declared on `MemoryStore`).
- `memory.*` SDK method signatures — B6 (not specified in source beyond the named surface).
- OpenSearch retrieval tier — ADR 0009; Postgres-primary invariant — ADR 0014.
- `MemoryStore` XR Composition mechanics — B4 / ADR 0044.

## 3. Context & Dependencies
Upstream consumed: A11 (OpenSearch retrieval tier Letta indexes land in). Downstream consumers: B11 (memory backend adapter) wraps Letta.
ADR decisions honored: **0005** — Letta is the memory backend, reached only via `Memory` CRD + SDK; **0025** — access modes declared on `MemoryStore`; **0014** — Postgres primary, OpenSearch reproducible-only; **0009** — OpenSearch indexes rebuildable; **0002** — `Memory`/`MemoryStore` admission-gated; **0030** — SDK compatibility matrix pins Letta.

## 4. Interfaces & Contracts
### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
- `Memory` (ARK-reconciled, namespaced) — `memoryStoreRef`. Access mode lives on `MemoryStore`, not here.
- `MemoryStore` (Crossplane XR, namespaced) — `accessMode` (private / namespace-shared / RBAC-OPA), `backendType`. Letta is the `backendType`.
Versioning per ADR 0030; `Memory` owned by A5 (ARK), `MemoryStore` by B4.

### 4.2 APIs / SDK surfaces
Platform SDK `memory.*` (B6) is the **only** path Platform Agents use to reach Letta. The SDK ships a version-pinned compatibility matrix against Letta versions. Method signatures beyond the `memory.*` surface group: not specified in source.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
`MemoryStore` lifecycle (created/started/paused/resumed/completed/failed/deleted) under `platform.lifecycle.*`. Per-event-type names deferred to B12.

### 4.4 Data schemas / connection-secret contracts
Letta state lives in managed Postgres (via the substrate `XPostgres` connection-secret shape: `host`, `port`, `user`, `password`, `dbname`). OpenSearch indexes Letta maintains are reproducible from Postgres/object storage and are never a system of record.

## 5. OSS-vs-Custom Decision
Upstream project: **Letta** (fully OSS), installed as a memory service behind `Memory` CRD; B11 is a thin custom adapter. Mode: install + wrap (B11 adapter) + pin. Rationale per ADR 0005: fully-OSS posture (self-host, no vendor dependency) and OS-style tiered memory model. Rejected: Mem0, Zep/Graphiti, Cognee, LangMem (collective rationale, backlog §2.9). Multi-tenancy gaps mitigated by wrapping behind `Memory` CRD + namespace-scoped instances.

## 6. Functional Requirements
- REQ-ADR-0005-01: Platform Agents MUST reach Letta only via ARK's `Memory` CRD and the SDK `memory.*` API; direct Letta calls are not a supported path.
- REQ-ADR-0005-02: Letta state MUST persist to managed Postgres as the system of record.
- REQ-ADR-0005-03: Any retrieval indexes Letta maintains in OpenSearch MUST be reproducible from Postgres/object storage; OpenSearch is never a memory system of record.
- REQ-ADR-0005-04: Letta MUST be provisioned through the `MemoryStore` Crossplane XR.
- REQ-ADR-0005-05: A B11 adapter MUST wrap Letta-specific calls so Platform Agents are insulated from the backend.
- REQ-ADR-0005-06: The SDK MUST ship a version-pinned compatibility matrix against Letta.
- REQ-ADR-0005-07: `Memory` and `MemoryStore` MUST be Gatekeeper-admission-gated; instances run namespace-scoped.

## 7. Non-Functional Requirements
- Security/tenancy: namespace-scoped Letta instances; access modes on `MemoryStore` (ADR 0025); RBAC-floor/OPA-restrictor over memory reads/writes (deferred detail to ADR 0025).
- Data: Postgres backup/PITR/DR posture inherited; OpenSearch reindex paths are first-class (exercised in DR drills).
- Observability: `MemoryStore` lifecycle under `platform.lifecycle.*`.
- Versioning: SDK compatibility matrix pins Letta alongside gateway/ARK (ADR 0030).

## 8. Cross-Cutting Deliverable Checklist
N/A — ADR. The §14.1 deliverable set is owned by A10 (Letta install) + B11 (adapter); conformance items appear in §9 / the PLAN.

## 9. Acceptance Criteria
Decision honored when:
- AC-ADR-0005-01: An agent attempting a direct Letta call (not via `Memory`/SDK) has no reachable path; only `memory.*` resolves. (REQ-01)
- AC-ADR-0005-02: A memory write is durably present in Postgres after a Letta restart. (REQ-02)
- AC-ADR-0005-03: Dropping and rebuilding the OpenSearch memory index from Postgres/object storage restores retrieval with no data loss. (REQ-03)
- AC-ADR-0005-04: Letta is created only by applying a `MemoryStore` XR; no out-of-band Letta provisioning path exists. (REQ-04)
- AC-ADR-0005-05: Swapping Letta for a fake backend through B11 leaves the SDK `memory.*` contract unchanged. (REQ-05)
- AC-ADR-0005-06: The installed Letta version equals the SDK matrix entry; CI fails on drift. (REQ-06)
- AC-ADR-0005-07: A non-conformant `Memory`/`MemoryStore` is denied by a Gatekeeper constraint. (REQ-07)

## 10. Risks & Open Questions
- Letta multi-tenancy gaps (blast radius: med) — mitigated by `Memory`-CRD wrapping + namespace-scoped instances per ADR 0005.
- Access-mode enforcement semantics deferred (med) — owned by ADR 0025, not this ADR; `[PROPOSED]` resolution tracked there.
- Migration off Letta would require re-implementing B11 (low) — SDK `memory.*` contract insulates agents.

## 11. References
- ADR 0005 (`adr/0005-letta-memory-backend.md`) — the decision.
- Enforcing components: A10 (Letta install, owner), B11 (memory backend adapter), B4 (`MemoryStore` XR), B6 (SDK `memory.*` + compat matrix), A11 (OpenSearch indexes), A7 (admission).
- architecture-overview.md §6.3, §6.7, §6.13, §9; architecture-backlog.md §2.9, §6.
- Related: ADR 0009, 0014, 0025, 0044, 0002, 0030.
