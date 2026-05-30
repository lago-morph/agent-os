# SPEC ADR-0009 — OpenSearch as the search and vector store [PROPOSED]

> kind: ADR · workstream: — · tier: T1
> upstream: [] · downstream: [A10;A18] · adrs: [0009] · views: [6.3;6.4;6.5]
> canon-glossary: 6aadcc2a4f38 · canon-interface: 54f5ede58e5f

## 1. Purpose & Problem Statement
ADR 0009 fixes OpenSearch as the single retrieval-optimization tier (vector + hybrid search, the `platform-knowledge-base` RAGStore, advisory audit/test indexes). This SPEC states what honoring that decision requires: OpenSearch is advisory fanout, never a system of record — everything in it is reproducible from a primary source (Postgres, S3, object storage, or an external system); all writes go through component adapters (audit, RAG, test-results streamer), never a direct write endpoint; it is provisioned dual-mode via the `XSearchIndex` XRD with a uniform connection secret; and it authenticates to Keycloak via its OSS Security plugin (no auth-proxy fronting). The decision is settled; this SPEC captures the obligations it imposes and the acceptance criteria that prove it is honored.

## 2. Scope
### 2.1 In scope
- OpenSearch as component A11: vector + hybrid search, KB RAG indexes, advisory audit/test-result indexes.
- The advisory-not-system-of-record invariant; reproducibility of everything in OpenSearch from primary sources.
- Adapter-only writes (audit adapter, RAG adapter, test-results streamer); no direct write endpoint.
- Dual-mode hosting via `XSearchIndex` XRD (in-cluster on kind, AWS-managed on AWS) with a uniform connection secret.
- OIDC/SAML via the OSS Security plugin to Keycloak (excluded from oauth2-proxy fronting).

### 2.2 Out of scope (and where it lives instead)
- OpenSearch install + deliverables — component A11 SPEC.
- Audit pipeline (Postgres + S3 system of record) — ADR 0034 / component A18; Postgres-primary invariant — ADR 0014.
- `XSearchIndex` Composition mechanics + connection-secret contract — B4 / ADR 0041.
- Memory backend that lands indexes here — ADR 0005 (Letta); KB primitive — ADR 0022.
- Three-layer test-result fanout — ADR 0011.

## 3. Context & Dependencies
Upstream consumed: none (A11 is a foundation component). Downstream consumers: A10 (Letta retrieval indexes), A18 (audit fanout).
ADR decisions honored: **0009** — OpenSearch is advisory retrieval, adapter-only writes, dual-mode via `XSearchIndex`, OSS-Security-plugin SSO; **0014** — Postgres primary, OpenSearch retrieval-only; **0034** — audit system of record is Postgres + S3, OpenSearch is fanout; **0033** — dual-mode kind + AWS-managed; **0041** — substrate abstraction pattern; **0011** — non-unit test results fan out here.

## 4. Interfaces & Contracts
### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
- `XSearchIndex` (XRD, namespaced) — `version`, `nodeCount`, `storage`, `connectionSecretRef`, `substrateClass`. One Composition per substrate (kind in-cluster OpenSearch; AWS managed OpenSearch). Owner: B4. XR status fields substrate-agnostic (`ready`, `endpoint`, `version`).
- `RAGStore` (B13-reconciled, namespaced) — `backend`, `indexes[]`, `contentSourceRefs[]`, `ingestionPipelineRef`; OpenSearch is the `backend`. `platform-knowledge-base` is the single KB RAGStore.

### 4.2 APIs / SDK surfaces
The SDK `rag.*` API (B6) reads OpenSearch for dense-vector, BM25 lexical, and hybrid retrieval — a single backend serves all three modalities. Service accounts reach OpenSearch via API. OpenSearch Dashboards is the operator UI, SSO'd through Keycloak.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
OpenSearch is a downstream sink of `platform.audit.*` events (via the audit adapter fanout), never a direct emitter into the taxonomy. Per-event-type names deferred to B12.

### 4.4 Data schemas / connection-secret contracts
Both `XSearchIndex` Compositions write a uniform connection secret (endpoint, user, password, optional TLS material) so consumers don't know which substrate is active — per the ADR 0041 contract; consumed by Kargo (ADR 0040) during promotion. OpenSearch holds no system-of-record data: vectors/embeddings re-derive from object storage, audit indexes from Postgres + S3, checkpoint-derived indexes from Postgres.

## 5. OSS-vs-Custom Decision
Upstream project: **OpenSearch**, installed dual-mode (in-cluster on kind, AWS-managed via Crossplane `XSearchIndex` on AWS), component A11. Mode: install + config + Crossplane-compose. Rationale per ADR 0009: OpenSearch's OSS Security plugin includes OIDC/SAML (removes a separate auth proxy in front of search) and its vector/hybrid search is at parity with Elasticsearch for the platform's use cases. Rejected: Elasticsearch (licensing + Security-tier gating, no offsetting benefit).

## 6. Functional Requirements
- REQ-ADR-0009-01: OpenSearch MUST be advisory fanout only — never a system of record for audit, memory, RAG, or test results.
- REQ-ADR-0009-02: Everything in OpenSearch MUST be reproducible from a primary source (Postgres, S3, object storage, or external system).
- REQ-ADR-0009-03: All writes MUST go through component adapters (audit adapter, RAG adapter, test-results streamer); OpenSearch MUST NOT be addressed as a primary write endpoint.
- REQ-ADR-0009-04: OpenSearch MUST be provisioned dual-mode via the `XSearchIndex` XRD (in-cluster on kind, AWS-managed on AWS) with a uniform connection secret.
- REQ-ADR-0009-05: A lag or outage on OpenSearch MUST NOT block the system of record (audit ingestion still succeeds; recovery is a reindex).
- REQ-ADR-0009-06: OpenSearch MUST authenticate to Keycloak via its OSS Security plugin and be excluded from oauth2-proxy fronting.
- REQ-ADR-0009-07: A single OpenSearch backend MUST serve dense-vector, BM25 lexical, and hybrid retrieval for the `rag.*` API and the Knowledge Base.

## 7. Non-Functional Requirements
- Resilience: reindex paths are first-class operational procedures, exercised in DR drills (backlog F2).
- Security/tenancy: SSO via OSS Security plugin to Keycloak; service accounts via API.
- Foundational ordering: A11 is scheduled early so HolmesGPT, the KB, Letta, the audit pipeline, and the test-result streamer have a retrieval tier to build against.
- Versioning: `XSearchIndex` XRD versioned per ADR 0030/0041 (conversion webhooks + deprecation windows on claim-shape changes).

## 8. Cross-Cutting Deliverable Checklist
N/A — ADR. The §14.1 deliverable set is owned by component A11; conformance items appear in §9 / the PLAN.

## 9. Acceptance Criteria
Decision honored when:
- AC-ADR-0009-01: Dropping any OpenSearch index and rebuilding it from its primary source restores it fully (audit from Postgres+S3, vectors from object storage, checkpoints from Postgres). (REQ-01, REQ-02)
- AC-ADR-0009-02: There is no code path that writes to OpenSearch except via an adapter (audit/RAG/test-results). (REQ-03)
- AC-ADR-0009-03: Applying an `XSearchIndex` claim on kind and on AWS yields the same connection-secret shape; consumers are substrate-agnostic. (REQ-04)
- AC-ADR-0009-04: With OpenSearch down, audit ingestion to Postgres+S3 still succeeds; recovery is a reindex, not a restore. (REQ-05)
- AC-ADR-0009-05: OpenSearch authenticates against Keycloak via the OSS Security plugin and has no oauth2-proxy in front of it. (REQ-06)
- AC-ADR-0009-06: The `rag.*` API serves dense-vector, BM25, and hybrid queries from the single OpenSearch backend. (REQ-07)

## 10. Risks & Open Questions
- Capability-parity caveat across substrates (blast radius: med) — claim shape consistent; runtime behaviour may differ per substrate; reindex paths bridge gaps.
- Migration off OpenSearch (low) — bounded: re-point `rag.*`, Letta index adapter, audit fanout, test-result streamer; data rebuildable from primary sources.
- Reindex performance under load (low) — exercised in DR drills (F2); settle sizing in A11 plan.

## 11. References
- ADR 0009 (`adr/0009-opensearch-search-vector-store.md`) — the decision.
- Enforcing components: A11 (OpenSearch install, owner), A18 (audit adapter fanout), A10 (Letta indexes), B4 (`XSearchIndex` Composition), B6 (`rag.*` API).
- architecture-overview.md §6.1, §6.3, §6.4, §6.5; architecture-backlog.md §2.15, §6, §14, F2.
- Related: ADR 0011, 0014, 0022, 0033, 0034, 0040, 0041.
