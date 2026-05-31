# SPEC F3 — Vendor documentation companion-project handoff

> kind: COMPONENT · workstream: F · tier: T2
> upstream: [C8, B4] · downstream: [] · adrs: [0024, 0022, 0009, 0014, 0030] · views: [6.4]
> canon-glossary: b0edae10a2e6 · canon-interface: 0ce201d5d5af

## 1. Purpose & Problem Statement

ADR 0024 decides that **vendor documentation acquisition is a separate companion project**, not part of this architecture's scope. The platform ships the *integration point* — the `platform-knowledge-base` RAGStore (the Knowledge Base primitive, ADR 0022) and C8's RAG indexing pipeline — but the scanning of upstream vendor releases for version differences and the acquisition/triggering of vendor-doc indexing lives in the companion project (referenced in §14.3 C8).

F3 is the production-readiness **handoff**: at the end of v1.0 it verifies that the contract between the companion project and the platform's Knowledge Base works as designed — that pinned vendor docs land in the `platform-knowledge-base` RAGStore via the agreed integration point, that the companion project's major/minor-release trigger reaches C8's pipeline, and that the boundary (what the platform owns vs. what the companion owns) is documented for handoff to the companion team. F3 builds nothing; it confirms and documents the seam.

## 2. Scope

### 2.1 In scope
- **Integration-point verification**: confirm the companion project can deliver vendor docs into the `platform-knowledge-base` RAGStore through C8's indexing pipeline (the same `RAGStore` ingestion path platform docs use).
- **Trigger-contract verification**: confirm the companion project's "vendor major/minor release detected" signal drives indexing the same way a contributor flagging a commit major/minor does (patch does not trigger — §14.3 C8).
- **Boundary documentation**: a handoff document delineating platform-owned surface (RAGStore, ingestion pipeline contract, trigger interface) vs. companion-owned surface (acquisition, version-diff scanning, vendor-doc curation), per ADR 0024.
- **Pinned-vendor-doc presence check**: verify pinned vendor docs are included in the Knowledge Base as a query-able source (the Knowledge Base contains "platform docs, runbooks, pinned vendor docs" per Canon §Knowledge Base).
- **Handoff acceptance**: a sign-off that the integration works end-to-end against at least one real vendor-doc source.

### 2.2 Out of scope (and where it lives instead)
- **The companion project itself** (acquisition, scanning, version-diff, curation) → the **separate companion project** (ADR 0024) — explicitly NOT in this architecture.
- **The Knowledge Base RAG indexing pipeline** → **C8** (platform owns the ingestion side; F3 verifies the seam, does not build the pipeline).
- **The `RAGStore` CRD / Knowledge Base primitive** → **B13** (reconciler) / **A11+A10** (backends) / ADR 0022; F3 sets no schema.
- **RAG-effectiveness dashboards** → **D2**.
- **Firecrawl / commercial scraping** → NOT in v1.0 (Canon: Firecrawl excluded; future-enhancements.md §7) — vendor-doc acquisition is documents, not live scraping.
- **Letta/OpenSearch backend internals** → A10/A11.

## 3. Context & Dependencies

**Upstream consumed:**
- **C8 (Knowledge Base RAG indexing pipeline)** — the ingestion pipeline that indexes authored docs/runbooks into `platform-knowledge-base`; F3 verifies the companion project's vendor docs ride this same pipeline and trigger model. F3 consumes C8's pipeline contract, does not change it.
- **B4 (Crossplane Compositions)** — provides the `SearchIndex`/`ObjectStore` substrate backing the RAGStore that the companion's docs are indexed into.

**Downstream consumers:** none (terminal handoff piece).

**ADRs honored:**
- **ADR 0024** — vendor doc acquisition is a *separate companion project*; F3's job is the handoff/integration verification, not building acquisition. The platform exposes the integration point only.
- **ADR 0022** — the Knowledge Base is a separate primitive (single `platform-knowledge-base` RAGStore included via CapabilitySet, not built into any agent); vendor docs are pinned content within it, not a new primitive.
- **ADR 0009 / 0014** — OpenSearch is the retrieval/vector store backing RAG; remains advisory/retrieval-only, not a system of record.
- **ADR 0030** — the ingestion/trigger interface the companion calls is a versioned surface (HTTP/path versioning per Canon §3.3); F3 records which version the handoff is against.

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
F3 defines **no new CRD/XRD**. It verifies content lands in the existing `RAGStore` (Canon §1.4: `backend`, `indexes[]`, `contentSourceRefs[]`, `ingestionPipelineRef`) — specifically the `platform-knowledge-base` RAGStore. The companion's vendor-doc source would appear as a `contentSourceRefs[]` entry / `ingestionPipelineRef` input. The exact registration shape of an external content source is C8-owned; anything beyond the stated `RAGStore` fields is `[PROPOSED — not in source]`.

### 4.2 APIs / SDK surfaces
The companion project calls C8's **ingestion pipeline trigger** (the same major/minor-release trigger contributors use). Per Canon §3.3, custom-service HTTP APIs use URL-path versioning (`/v1/...`). F3 verifies the companion targets the published trigger version. The platform SDK `rag.*` surface (Canon §3.1) is how agents *query* the Knowledge Base; F3 confirms vendor docs are retrievable through it. Concrete trigger/ingest signatures are **not specified in source** — `[PROPOSED — not in source]`.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
A "vendor docs updated / indexed" notification, if emitted, falls under `platform.capability.*` (a RAGStore content change is a capability-registry change; `platform.capability.changed` per ADR 0013) or `platform.observability.*` for indexing-job telemetry — both `[PROPOSED — not in source]`. F3 mints no event types.

### 4.4 Data schemas / connection-secret contracts
Vendor docs are indexed into the RAGStore backends (`SearchIndex` vector/index + object storage) using the standard connection-secret contract (Canon §4); F3 adds no fields. The doc payload schema (chunking, embedding) is C8-owned.

## 5. OSS-vs-Custom Decision
**Integration verification + handoff documentation — no build.** The platform side already exists (C8 pipeline, ADR 0022 RAGStore, A11/A10 backends). F3 is **config/verify + docs**: wire a real vendor-doc source through the existing trigger, confirm retrieval, and write the boundary/handoff doc. No fork, no new service. Rationale: ADR 0024 deliberately scopes acquisition *out*; F3 must not build it, only prove and document the seam the architecture already committed to.

## 6. Functional Requirements
- **REQ-F3-01:** F3 MUST verify that a vendor-doc source delivered by the companion project is indexed into the `platform-knowledge-base` RAGStore via C8's existing ingestion pipeline (no bespoke ingestion path).
- **REQ-F3-02:** F3 MUST verify the companion's major/minor-release trigger drives indexing identically to a contributor major/minor flag, and that patch-level changes do **not** trigger (§14.3 C8).
- **REQ-F3-03:** F3 MUST confirm pinned vendor docs are retrievable from the Knowledge Base via the platform SDK `rag.*` surface (queryable by a Platform Agent through its CapabilitySet).
- **REQ-F3-04:** F3 MUST produce a **handoff document** delineating platform-owned vs. companion-owned surfaces per ADR 0024, including the versioned trigger/ingest interface the companion targets.
- **REQ-F3-05:** F3 MUST record the integration as verified against at least one real vendor-doc source end-to-end (acquisition-stub or real companion output → index → retrieve).
- **REQ-F3-06:** F3 MUST NOT introduce any acquisition / version-scanning logic into the platform (ADR 0024 boundary); any such need is logged as companion-project scope.
- **REQ-F3-07:** F3 MUST confirm the Knowledge Base remains a single shared RAGStore included via CapabilitySet (ADR 0022) — vendor docs do not create a second primitive.

## 7. Non-Functional Requirements
- **Security:** vendor docs entering the Knowledge Base MUST pass the same ingestion controls as platform docs (no privileged bypass); the companion's trigger MUST authenticate to the C8 pipeline like any other caller. `[PROPOSED]` — auth mode is C8-owned.
- **Multi-tenancy (§6.9):** the Knowledge Base is platform-shared; F3 confirms vendor-doc visibility honors the existing RAGStore visibility/RBAC, not a new tenant model.
- **Observability (§6.5):** vendor-doc indexing runs MUST be observable (success/failure, doc count) through C8's existing indexing telemetry; F3 verifies, does not add a pipeline.
- **Scale:** vendor-doc corpus size is informational for F5; F3 confirms ingestion handles a representative vendor-doc batch without manual intervention.
- **Versioning (ADR 0030):** F3 pins the trigger/ingest API version the handoff is certified against; a later companion-side change targeting a deprecated version is a handoff-doc-flagged risk.

## 8. Cross-Cutting Deliverable Checklist
- Helm/manifests — **N/A — F3 deploys nothing; the RAGStore + pipeline already exist (C8/A11/A10).**
- Per-product docs (10.5) — **applicable** (handoff/boundary document; "how vendor docs reach the Knowledge Base").
- Runbook (10.7) — **applicable** (verify/repair the companion→Knowledge-Base integration). Feeds **F6**.
- Alerts — **N/A — indexing-failure alerting is C8's; F3 verifies it fires, adds none.** `[PROPOSED]` reuse C8 alert.
- Grafana dashboard (Crossplane XR) — **N/A — RAG-effectiveness dashboards are D2.**
- Headlamp plugin — **N/A — no interactive CRD surface introduced.**
- OPA/Rego integration — **N/A — F3 verifies ingestion controls; adds no policy.**
- Audit emission (ADR 0034) — **applicable** — `[PROPOSED]` a `platform.capability.*` record when vendor docs change the Knowledge Base content (ADR 0013); emission is C8/B13-owned.
- Knative trigger flow — **applicable** — F3 verifies the companion's major/minor trigger reaches C8 (the existing trigger model), introduces no new flow.
- HolmesGPT toolset — **N/A — handoff verification, not an operational toolset surface.**
- 3-layer tests — **applicable** (PyTest for ingest+retrieve of a fixture vendor doc; Chainsaw for RAGStore content-source registration; Playwright N/A or minimal for LibreChat retrieval check).
- Tutorials & how-tos — **applicable** (how-to: register a companion vendor-doc source; reference: the handoff boundary).

## 9. Acceptance Criteria
- **AC-F3-01:** A vendor doc supplied via the companion path is searchable in `platform-knowledge-base` after indexing. (→ REQ-F3-01)
- **AC-F3-02:** A simulated major/minor release triggers indexing; a patch-level change does not. (→ REQ-F3-02)
- **AC-F3-03:** A Platform Agent retrieves a known vendor-doc fact via `rag.*` through its CapabilitySet. (→ REQ-F3-03)
- **AC-F3-04:** The handoff document exists and lists platform-owned vs. companion-owned surfaces and the pinned trigger/ingest API version. (→ REQ-F3-04, REQ-F3-07)
- **AC-F3-05:** End-to-end verification (source → index → retrieve) is recorded against at least one real vendor-doc source. (→ REQ-F3-05)
- **AC-F3-06:** Repository/code review confirms no acquisition or version-scanning logic was added to the platform. (→ REQ-F3-06)

## 10. Risks & Open Questions
- **R1 (med):** The companion project may not exist / be ready at v1.0 GA, leaving the seam unverified against a real producer. Mitigation: verify with a faithful acquisition-stub that targets the published trigger/ingest contract; flag real-companion verification as a follow-up. Blast radius: med.
- **R2 (med):** Boundary drift — the companion may assume platform-side behavior the platform doesn't provide (e.g., dedup, version pinning). Mitigation: the handoff doc is the contract; ambiguous responsibilities are assigned explicitly. Blast radius: med.
- **R3 (low):** Vendor docs could bloat the Knowledge Base and degrade retrieval relevance. Mitigation: coordinate corpus sizing with D2/F5; out of F3's build scope. Blast radius: low.
- **OQ1:** Does the companion push into the pipeline, or does C8 pull from a companion-published location? `[PROPOSED]` companion triggers C8's existing pull-style ingestion (mirrors the contributor flag model); confirm with C8 owner.
- **OQ2:** Are vendor docs versioned/pinned per platform release inside the Knowledge Base? `[PROPOSED]` pinned per Canon's "pinned vendor docs" wording; pin policy is companion-curated.

## 11. References
- architecture-overview.md §6.4 / V6-04 (Knowledge Base as a separate primitive), §14.3 C8 line 1731 (vendor doc acquisition is a separate companion project; major/minor trigger; patch does not trigger), §14.6 line 1759 (F3 scope).
- Canon glossary (Knowledge Base = single `platform-knowledge-base` RAGStore of platform docs, runbooks, pinned vendor docs; included via CapabilitySet). Canon interface-contract §1.4 (`RAGStore`), §3.1 (`rag.*`), §3.3 (URL-path-versioned custom-service APIs), §2 (`platform.capability.*`).
- ADR 0024 (vendor doc acquisition separate companion project), ADR 0022 (Knowledge Base separate primitive), ADR 0009/0014 (OpenSearch retrieval-only), ADR 0030 (versioning).
- Related pieces: C8, B4, A10, A11, B13, D2.
