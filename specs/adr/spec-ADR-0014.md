# SPEC ADR-0014 — Postgres as primary storage; OpenSearch as retrieval optimization only [PROPOSED]

> kind: ADR · workstream: — · tier: T1
> upstream: [A11, B4] · downstream: [A10, A18, A11, B11, C8, F2] · adrs: [0014] · views: [6.3, 6.5]
> canon-glossary: 6aadcc2a4f38 · canon-interface: 54f5ede58e5f

## 1. Purpose & Problem Statement
ADR 0014 is a settled decision and a platform invariant: **Postgres (and where appropriate object storage or an external system of record) is the primary store for all stateful data; OpenSearch is a retrieval-optimization layer only.** Every OpenSearch index MUST be reproducible from a primary source via a documented rebuild path; components that hold state write to a primary first. This SPEC states the obligations the invariant imposes on every stateful component, the rebuild-path contract, and the acceptance criteria proving the invariant holds. It does not re-argue the choice of Postgres or OpenSearch.

The problem the decision solves: without an explicit rule it is tempting to treat OpenSearch (the read-path store) as authoritative for derived artifacts — embeddings, audit indexes, search-over-checkpoints — which would make DR and migration data-migration exercises rather than reindex exercises and would put the system of record on a cache.

## 2. Scope
### 2.1 In scope
- The invariant: every stateful write hits a primary (Postgres / object storage / external SoR) first; OpenSearch is never SoR.
- The rebuild-path obligation: every OpenSearch index has a documented rebuild source (sync dual-write or reindex).
- Application to audit indexes (ADR 0034), KB indexes, Letta-derived indexes, test-result indexes (ADR 0011).
- DR posture: managed-Postgres restore + object-storage durability + OpenSearch reindex (not OpenSearch backup/restore).
- The dual-mode Postgres hosting invisibility (kind in-cluster / AWS RDS) resting on system-of-record semantics, via `Postgres` substrate abstraction (ADR 0044).

### 2.2 Out of scope (and where it lives instead)
- The audit pipeline mechanics — ADR 0034 / component **A18**.
- OpenSearch install/operation — ADR 0009 / component **A11**.
- `Postgres` Composition build — ADR 0044 / component **B4**.
- DR test execution — Workstream **F2**.
- KB indexing pipeline — component **C8**.
- Audit retention/redaction — Workstream **F1**.

## 3. Context & Dependencies
Upstream consumed: **A11** OpenSearch (the retrieval layer governed here); **B4** Crossplane Compositions (provide `Postgres` primaries). Downstream consumers / conformers: **A10** Letta (Postgres primary, derived OpenSearch indexes), **A18** audit pipeline, **B11** memory adapter, **C8** KB indexing, **F2** DR testing.

ADR decisions honored:
- **ADR 0014** (this) — Postgres/object-storage/external = SoR; OpenSearch = derived, rebuildable.
- **ADR 0009** — OpenSearch is the search/vector store, advisory only.
- **ADR 0011** — non-unit test results stream to OpenSearch as an advisory index; audit trail lives in the primary.
- **ADR 0033** — dual-mode hosting (kind in-cluster Postgres / AWS RDS); invisible above the connection string.
- **ADR 0034** — audit SoR is Postgres + S3 (Postgres-only on kind); OpenSearch audit index advisory.
- **ADR 0040 / 0044** — `Postgres` XRD with one Composition per substrate writes the uniform connection-secret; Kargo consumes it during promotion.

## 4. Interfaces & Contracts
### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
- `Postgres` (XRD, namespaced XR created directly in the tenant namespace per Crossplane v2, B4) — `version`, `size`, `storage`, `connectionSecretRef`, `substrateClass`. kind: CloudNativePG `Cluster`; AWS: RDS. Status `ready`/`endpoint`/`version` (substrate-agnostic).
- `SearchIndex` (XRD, B4) — backs OpenSearch as the derived retrieval layer; `connectionSecretRef`, `substrateClass`.
- `AuditLog` (XRD, ADR 0034) — provisions the Postgres+S3 SoR pipeline plus the advisory OpenSearch indexer.

### 4.2 APIs / SDK surfaces
- Consuming components see only a Postgres connection string via the standard secret/config path; dual-mode is invisible above the connection. No new API surface introduced by this ADR.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
- Audit events flow under `platform.audit.*`; lifecycle of memory/state stores under `platform.lifecycle.*`. This ADR introduces no new namespace.

### 4.4 Data schemas / connection-secret contracts
- Uniform connection-secret shape (ADR 0044): `host`, `port`, `user`, `password`, `dbname` — written identically by the kind and AWS `Postgres` Compositions so consumers bind the same keys regardless of substrate.
- Audit in-flight rows live in the Postgres `audit_events` table (ADR 0034); on AWS a batch CronJob aggregates to immutable S3 (verified) before deleting rows; on kind Postgres alone is SoR and the batch step is disabled.

## 5. OSS-vs-Custom Decision
N/A — ADR. (Enforcement note: upstream primaries are **CloudNativePG**/**RDS** (Postgres), **MinIO**/**S3** (object storage), with **OpenSearch** as the derived retrieval layer; config/compose via `Postgres`/`ObjectStore`/`SearchIndex`, no fork. Rejected alternative — treating OpenSearch as authoritative for derived artifacts — is forbidden by the invariant.)

## 6. Functional Requirements
- REQ-ADR-0014-01: Postgres (or object storage or an external system of record) MUST be the primary store for all stateful platform data; OpenSearch MUST NOT be the system of record for any data.
- REQ-ADR-0014-02: Every OpenSearch index MUST have a documented rebuild path from a primary (synchronous dual-write or reindex); an index with no rebuild source is a violation.
- REQ-ADR-0014-03: Any component holding state MUST write to a primary before — or instead of — writing to OpenSearch; a component that writes only to OpenSearch MUST be redesigned.
- REQ-ADR-0014-04: Reindex paths MUST be first-class operational procedures (vectors/embeddings re-derived from source docs; audit indexes from the S3 archive on AWS or Postgres on kind; checkpoint indexes from Postgres).
- REQ-ADR-0014-05: DR MUST be a managed-Postgres restore + object-storage durability + OpenSearch reindex exercise, not an OpenSearch backup/restore loop.
- REQ-ADR-0014-06: The audit pipeline MUST keep its system of record in Postgres + S3 (Postgres-only on kind) with the OpenSearch audit index advisory and rebuildable (ADR 0034).
- REQ-ADR-0014-07: Consuming components MUST depend only on Postgres connection-string semantics, not on hosting shape; the kind and AWS `Postgres` Compositions MUST write the same connection-secret keys.
- REQ-ADR-0014-08: Test-result, KB, and Letta-derived OpenSearch indexes MUST each be derived and rebuildable from their respective primary.

## 7. Non-Functional Requirements
- Security/tenancy: primaries inherit namespace/RBAC scoping; audit SoR identifies actor + environment (ADR 0034).
- Observability (§6.5): reindex/rebuild operations emit under `platform.lifecycle.*` / `platform.observability.*`; OpenSearch downtime MUST NOT break audit ingestion.
- Scale: OpenSearch cost/scale is a function of derived indexes, all rebuildable; primary durability is the DR anchor.
- Versioning (ADR 0030): `Postgres`/`SearchIndex` XRDs versioned with conversion webhooks (B4-owned, per ADR 0044).

## 8. Cross-Cutting Deliverable Checklist
N/A — ADR (verification map in the PLAN). The §14.1 set is owned by enforcing components (A11, A18, B4, A10, C8, F2).

## 9. Acceptance Criteria
- AC-ADR-0014-01: Honored when a SoR inventory shows zero indexes whose only copy lives in OpenSearch. (REQ-01)
- AC-ADR-0014-02: Honored when every catalogued OpenSearch index lists a documented rebuild source. (REQ-02)
- AC-ADR-0014-03: Honored when a write-path audit of stateful components shows each writes a primary first; an OpenSearch-only write fails review/CI. (REQ-03)
- AC-ADR-0014-04: Honored when reindexing a wiped OpenSearch from primaries restores each index without data loss. (REQ-04)
- AC-ADR-0014-05: Honored when a DR drill (F2) recovers via Postgres restore + reindex with no OpenSearch backup involved. (REQ-05)
- AC-ADR-0014-06: Honored when, with OpenSearch down, audit ingestion still succeeds and the audit index rebuilds from S3 (AWS) / Postgres (kind). (REQ-06)
- AC-ADR-0014-07: Honored when a consumer binds the same secret keys against the kind and AWS Compositions with no code change. (REQ-07)
- AC-ADR-0014-08: Honored when KB, Letta, and test-result indexes each rebuild from their primary. (REQ-08)

## 10. Risks & Open Questions
- R-1 (med): On kind, with no S3, Postgres alone is the audit SoR; durability rests entirely on Postgres being a primary, not a cache — blast radius is kind/dev only.
- OQ-1 (low): Capability-parity caveat — kind `ObjectStore` may produce "no archive lifecycle" (ADR 0044); affects archive-based rebuild on kind. `[PROPOSED]`
- R-2 (low): Synchronous dual-write paths can drift from the primary under partial failure; mitigated by reindex being the authoritative reconciliation.

## 11. References
- ADR 0014 (`adr/0014-postgres-primary-opensearch-retrieval.md`) — the decision enforced here.
- architecture-overview.md §6.3 (memory & data), §6.5 (observability); architecture-backlog.md §6 (invariant).
- Enforcing components: A11 (OpenSearch), A18 (audit), B4 (Crossplane `Postgres`/`SearchIndex`), A10 (Letta), C8 (KB indexing), F2 (DR testing).
- Related ADRs: 0009, 0011, 0033, 0034, 0040, 0044.
