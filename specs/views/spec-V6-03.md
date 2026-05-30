# SPEC V6-03 — Memory and data architecture `[PROPOSED]`

> kind: VIEW · workstream: — · tier: T1
> upstream: [] · downstream: [] · adrs: [0005, 0009, 0014, 0025, 0033, 0034, 0041] · views: [6.3]
> canon-glossary: 6aadcc2a4f38 · canon-interface: 54f5ede58e5f

## 1. Purpose & Problem Statement

This view defines the **integration contract** for the memory and data slice of the Agentic Execution Platform: the slice that fixes storage roles so they cannot drift. **Postgres** is the system of record for state and checkpoints; **OpenSearch** is a retrieval-optimization layer only; object storage is the immutable archive. **Letta** is the memory service; Platform Agents reach memory and RAG only through the Platform SDK's `memory.*` / `rag.*` surface. It is not a buildable component; it is realized by the Letta install (A10), OpenSearch (A11), the Crossplane Compositions (B4), and the memory backend adapter (B11).

The problem it solves: an agent platform accumulates state across many backends, and without a firm rule about which store is authoritative, retrieval indexes silently become systems of record and become unrecoverable. This view fixes the reproducibility invariant ("anything in OpenSearch must be reproducible from a primary source"), the memory access-mode contract, and the substrate-abstraction contract (one XRD per primitive, one Composition per substrate, uniform connection secret) so the same agent code runs on kind and AWS without branching.

## 2. Scope
### 2.1 In scope
- The storage-role invariant: Postgres = system of record; OpenSearch = retrieval optimization only; object storage = immutable archive (ADR 0014, ADR 0009).
- The reproducibility invariant: every OpenSearch index is re-derivable from a primary (object storage documents, Postgres checkpoints, S3/Postgres audit).
- The memory access-mode contract on `MemoryStore`: private / namespace-shared / RBAC-OPA (ADR 0025).
- The Letta memory service path: agent → Platform SDK `memory.*` → Letta → Postgres / OpenSearch / object storage (ADR 0005).
- The substrate-abstraction contract (ADR 0041): substrate XRDs `XPostgres`, `XSearchIndex`, `XObjectStore`, `XMongoDocStore`; one Composition per substrate; uniform connection secret; substrate-agnostic status; label-driven, admission-validated selection.
- Dual-mode hosting (ADR 0033): in-cluster on kind, AWS managed on AWS, for Postgres and OpenSearch.
- The audit-pipeline topology as it touches data stores: Postgres + S3 system of record, OpenSearch advisory fanout (ADR 0034) — wiring to stores only; full observability view is V6-05.

### 2.2 Out of scope (and where it lives instead)
- Knowledge Base as a primitive (`platform-knowledge-base` RAGStore, re-indexing rules) — view V6-04 / ADR 0022.
- Gateway call path and `RAGStore` reconciliation — view V6-01 / component B13.
- Full observability / audit-emission view (adapter library, endpoint, dashboards) — view V6-05 / component A18.
- Memory backend adapter internals — component B11 (consumed, not defined here).
- Audit retention / redaction policy — Workstream F (F1).
- `XAgentDatabase` per-agent DB provisioning detail — view V6-01 (initial MCP services) / ADR 0020.

## 3. Context & Dependencies

Realizing components and what each contributes to the slice:
- **A10 Letta memory backend** — the memory service; persists to Postgres (state) and OpenSearch / object storage as appropriate (ADR 0005).
- **A11 OpenSearch** — vectors + hybrid search + advisory audit fanout; never a system of record (ADR 0009).
- **B4 Crossplane v2 Compositions** — owns the substrate XRDs and per-substrate Compositions; writes the uniform connection secret (ADR 0041).
- **B11 Memory backend adapter** — adapts the Platform SDK `memory.*` surface to Letta.

ADR decisions honored:
- **ADR 0005** — Letta is the memory backend.
- **ADR 0009** — OpenSearch is the search/vector store and advisory audit fanout, not a system of record.
- **ADR 0014** — Postgres is primary storage; OpenSearch is retrieval optimization only.
- **ADR 0025** — memory access modes (private / namespace-shared / RBAC-OPA) live on `MemoryStore`.
- **ADR 0033** — initial targets AWS (EKS) and GitHub; dual-mode hosting (kind in-cluster / AWS managed).
- **ADR 0034** — audit system of record is Postgres + S3; OpenSearch advisory only (wiring to stores).
- **ADR 0041** — substrate abstraction via Crossplane Compositions; uniform connection-secret contract; admission-validated substrate selection.

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
Owned by Crossplane Compositions (B4):
- `MemoryStore` (XR) — `accessMode` (private / namespace-shared / RBAC-OPA), `backendType` (ADR 0025).
- `XPostgres` (XRD) — `version`, `size`, `storage`, `connectionSecretRef`, `substrateClass`. kind: CloudNativePG; AWS: RDS.
- `XSearchIndex` (XRD) — `version`, `nodeCount`, `storage`, `connectionSecretRef`, `substrateClass`. kind: in-cluster OpenSearch; AWS: managed OpenSearch.
- `XObjectStore` (XRD) — `bucketName`, `lifecycle`, `connectionSecretRef`, `substrateClass`. kind: MinIO or no-op; AWS: S3.
- `XMongoDocStore` (XRD) — `version`, `size`, `storage`, `connectionSecretRef`, `substrateClass`. kind: Bitnami MongoDB; AWS: DocumentDB or self-managed.

Consumed (owned by ARK, A5):
- `Memory` — `memoryStoreRef` (binds an agent to a `MemoryStore`; access mode resolved from the store, ADR 0025).

All namespaced; versioning per ADR 0030/0041 (conversion webhooks + deprecation windows on claim-shape changes); owner of versioning lifecycle = B4.

### 4.2 APIs / SDK surfaces
- **Platform SDK (B6)** `memory.*` and `rag.*` — the only agent-facing surface for memory and retrieval; `memory.*` adapts to Letta via B11; `rag.*` goes through LiteLLM (defined in V6-01/V6-04; consumed here).
- Letta service API — internal target of the memory adapter (B11); method surface not specified in source.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
Emitted by the data slice (per-event-type names deferred to B12 registry):
- `platform.lifecycle.*` — `MemoryStore` lifecycle (created/started/paused/resumed/completed/failed/deleted).
- `platform.audit.*` — audit events landing in the Postgres + S3 system of record with OpenSearch advisory fanout (ADR 0034), via the audit adapter.

### 4.4 Data schemas / connection-secret contracts
- **Uniform connection-secret contract (architectural invariant, tested):** every Composition writes `host`, `port`, `user`, `password`, `dbname` (or the equivalent per primitive). Per-primitive field lists beyond these five are not enumerated in source — `[PROPOSED — not in source]` for any addition.
- **XR status is substrate-agnostic:** `ready`, `endpoint`, `version`; substrate-specific fields (RDS ARN, in-cluster Service paths) are deliberately absent.
- **Substrate selection:** every cluster carries `platform.io/environment=kind|aws`; a Gatekeeper admission policy rejects any claim with no matching Composition for the cluster's label.
- **Capability-parity is not promised:** claim shape is consistent; runtime behaviour may differ per substrate (e.g. kind `XObjectStore` may produce "no archive").
- Audit `audit_events` table is the in-flight Postgres row store; on AWS a ~5-min batch CronJob aggregates rows → immutable S3 object (verified) then deletes Postgres rows; on kind, Postgres alone is the system of record (ADR 0034).

## 5. OSS-vs-Custom Decision
N/A — VIEW (OSS-vs-custom for Letta, OpenSearch, Crossplane Compositions, and the memory adapter lives in component SPECs A10, A11, B4, B11).

## 6. Functional Requirements
Each requirement is an **invariant/constraint this view imposes** on every participating component.

- **REQ-V6-03-01:** Postgres MUST be the system of record for state and checkpoints; OpenSearch MUST be a retrieval-optimization layer only and MUST NOT be a system of record (ADR 0014, ADR 0009). (§6.3, lines ~296, ~325, ~337)
- **REQ-V6-03-02:** Anything stored in OpenSearch MUST be reproducible from a primary source — vectors/embeddings re-derivable from object-storage documents, audit indexes re-derivable from S3 (AWS) or Postgres (kind). (§6.3, line ~325)
- **REQ-V6-03-03:** Every `MemoryStore` MUST declare exactly one access mode — private / namespace-shared / RBAC-OPA — and the platform MUST enforce visibility from that declaration; RBAC-OPA mode follows RBAC-as-floor / OPA-as-restrictor (ADR 0025). (§6.3, lines ~327–333)
- **REQ-V6-03-04:** Agents MUST reach memory and RAG only through the Platform SDK `memory.*` / `rag.*` surface (memory via Letta, RAG via LiteLLM); no direct backend access. (§6.3, lines ~300–319)
- **REQ-V6-03-05:** Each substrate-asymmetric primitive MUST be wrapped in one XRD with one Composition per supported substrate (kind + AWS), and both Compositions MUST write the same connection-secret shape (`host`, `port`, `user`, `password`, `dbname` or equivalent) (ADR 0041). (§6.3, line ~339; interface-contract §4)
- **REQ-V6-03-06:** XR status MUST be substrate-agnostic (`ready`, `endpoint`, `version`); substrate-specific status fields MUST NOT appear in user-visible status (ADR 0041). (§6.3, line ~339; interface-contract §4)
- **REQ-V6-03-07:** Substrate selection MUST be driven by the `platform.io/environment=kind|aws` label, and Gatekeeper admission MUST reject any claim with no matching Composition for the cluster's label (ADR 0041 + ADR 0002). (§6.3, line ~339)
- **REQ-V6-03-08:** The v1.0 committed substrate XRDs MUST be `XPostgres`, `XSearchIndex`, `XObjectStore`, `XMongoDocStore`; Postgres and OpenSearch MUST be dual-mode hosted (kind in-cluster / AWS managed) (ADR 0033, ADR 0041). (§6.3, lines ~335–339)
- **REQ-V6-03-09:** Capability-parity MUST NOT be promised — reduced-capability substrate gaps (e.g. kind no-archive object store) MUST be documented explicitly in the Composition docs, not silent. (§6.3, line ~339)

## 7. Non-Functional Requirements
- **Security/multi-tenancy (§6.9):** `MemoryStore` access modes are the per-store isolation boundary; RBAC-OPA mode enforces cross-namespace sharing under the platform-wide enforcement model; all stores are namespaced.
- **Observability (§6.5):** `MemoryStore` lifecycle emits `platform.lifecycle.*`; data-store audit flows via the adapter into the Postgres + S3 system of record with OpenSearch advisory fanout.
- **Scale / DR:** AWS RDS provides backup/PITR/replication; DR procedures tested in Workstream F; kind path is functionally complete without cloud services.
- **Versioning (ADR 0030/0041):** XRD/claim shape changes go through conversion webhooks + deprecation windows; versioning owned per-component by B4.

## 8. Cross-Cutting Deliverable Checklist
N/A — VIEW (cross-cutting deliverables are owned by realizing components A10, A11, B4, B11).

## 9. Acceptance Criteria
The view holds when:
- **AC-V6-03-01:** A test confirms checkpoints/state read back from Postgres after the OpenSearch index is dropped and rebuilt; no data is lost by losing OpenSearch. (→ REQ-01, REQ-02)
- **AC-V6-03-02:** Dropping the OpenSearch vector index and re-ingesting from object-storage documents reproduces the index; dropping the audit index and rebuilding from S3 (AWS) / Postgres (kind) reproduces it. (→ REQ-02)
- **AC-V6-03-03:** A `MemoryStore` in each access mode enforces the expected read/write visibility (private isolates to the writer; namespace-shared visible in-namespace; RBAC-OPA gated by RBAC + OPA). (→ REQ-03)
- **AC-V6-03-04:** An agent reaching memory/RAG outside the SDK surface has no working path; the only working path is `memory.*` → Letta and `rag.*` → LiteLLM. (→ REQ-04)
- **AC-V6-03-05:** A `Postgres` claim on kind and on AWS both yield a connection secret with the identical field shape; the agent consumes it without substrate branching. (→ REQ-05)
- **AC-V6-03-06:** The XR status surface exposes only `ready`/`endpoint`/`version`; no RDS ARN or in-cluster Service path leaks into user-visible status. (→ REQ-06)
- **AC-V6-03-07:** A claim with no matching Composition for the cluster's `platform.io/environment` label is rejected at admission, not at runtime. (→ REQ-07)
- **AC-V6-03-08:** All four substrate XRDs exist and resolve on both kind and AWS; Postgres and OpenSearch run in-cluster on kind and managed on AWS. (→ REQ-08)
- **AC-V6-03-09:** The kind `XObjectStore` no-archive gap (and any other reduced-capability gap) is documented in the Composition docs. (→ REQ-09)

## 10. Risks & Open Questions
- **OQ-1 (med):** Per-primitive connection-secret field lists beyond `host/port/user/password/dbname` are not in source; resolved in B4 component SPEC (`[PROPOSED — not in source]` for additions).
- **OQ-2 (low):** Audit retention / redaction policy is deferred to F1; this view fixes only topology-to-store wiring.
- **R-1 (high):** If a component writes a search index that has no primary source, the reproducibility invariant breaks and OpenSearch silently becomes a system of record; mitigated by REQ-02 conformance tests and review gates.
- **R-2 (med):** Capability-parity caveats (kind no-archive) can surprise operators who assume kind ≡ AWS; mitigated by REQ-09 explicit Composition docs.

## 11. References
- architecture-overview.md §6.3 Memory and data architecture (lines ~294–347); §6.5 (audit pipeline cross-ref); interface-contract.md §1.6 (XRDs), §4 (connection-secret), §5 (audit adapter).
- ADRs: 0005 (Letta), 0009 (OpenSearch), 0014 (Postgres primary / OpenSearch optimization), 0025 (memory access modes), 0033 (AWS+GitHub targets / dual-mode), 0034 (audit pipeline), 0041 (substrate abstraction).
- Realizing components: A10 (Letta), A11 (OpenSearch), B4 (Crossplane Compositions), B11 (memory adapter).
- Related views: V6-01, V6-04, V6-05, V6-06.
