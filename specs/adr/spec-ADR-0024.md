# SPEC ADR-0024 — Vendor documentation acquisition is a separate companion project [PROPOSED]

> kind: ADR · workstream: — · tier: T2
> upstream: [] · downstream: [C8, F3] · adrs: [0024] · views: [6.4]
> canon-glossary: cf2d1a754a58 · canon-interface: 45ee7b798c47

## 1. Purpose & Problem Statement

ADR 0024 is a settled decision: vendor-documentation acquisition and re-indexing live in a **separate companion project**, referenced from this architecture but explicitly out of v1.0 architecture scope. The architecture defines only the consumption contract — the `platform-knowledge-base` RAGStore (ADR 0022) and the `rag.*` SDK API — and treats the companion project as an upstream producer that writes pinned vendor-doc versions into that store. This SPEC states what honoring that scoping obliges: a narrow, stable contract between the two projects; v1.0 not gating on the pipeline existing; architecture-side ownership limited to the integration point (Workstream F, item F3) and keeping the cross-reference current. It does not re-argue why vendor-doc acquisition is out of scope.

The problem the decision solves: acquiring vendor docs (crawling third-party sources, tracking upstream versions, licensing/redistribution, re-index triggers on vendor releases) is qualitatively different from authoring our own docs, has its own lifecycle/ownership/risk, and bundling it into v1.0 would conflate "Knowledge Base as a primitive" with "data plane that fills the Knowledge Base."

## 2. Scope

### 2.1 In scope
- The obligation that the consumption contract (the `platform-knowledge-base` RAGStore + `rag.*` SDK API) is the only architecture-owned surface; the companion project is an external producer.
- The obligation that v1.0 does NOT gate on a vendor-doc pipeline existing — the Knowledge Base ships populated from authored docs/runbooks (C8).
- The obligation that the companion project, when it lands, writes into the named RAGStore using the same indexing conventions as authored docs and re-indexes on upstream major/minor releases.
- The obligation that the integration point and cross-reference currency are owned by Workstream F item F3.

### 2.2 Out of scope (and where it lives instead)
- The vendor-doc crawl/sync/version-tracking/licensing pipeline itself — the **separate companion project** (external; not a v1.0 piece).
- Authoring of platform docs/runbooks that populate the Knowledge Base in v1.0 — Workstream **C** item **C8** (RAG indexing pipeline).
- Companion-project ownership, links, freshness SLAs — filled in later; tracked by **F3**.
- The Knowledge Base primitive itself (RAGStore existence, consumer fan-out) — **ADR 0022**.
- Any future change to the Knowledge Base contract the companion project might require — a separate ADR against ADR 0022, not this scoping decision.

## 3. Context & Dependencies

Upstream consumed: none within v1.0 architecture scope — the companion project is external and explicitly not a v1.0 piece.
Downstream consumers: **C8** (Knowledge Base RAG indexing pipeline) defines the indexing conventions the companion project must match; **F3** (vendor documentation companion project handoff) owns the integration point and cross-reference.

ADR decisions honored:
- **ADR 0024** (this) — vendor-doc acquisition is a separate companion project; narrow contract; no v1.0 gate.
- **ADR 0022** — the `platform-knowledge-base` RAGStore is the consumption-side primitive the companion project writes into.
- **ADR 0008** — Material for MkDocs is the authored-docs portal the companion content sits alongside.
- **ADR 0009** — OpenSearch is the shared retrieval substrate both authored and vendor content index into.

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs
- `RAGStore` (kopf-operator B13, namespaced) — the `platform-knowledge-base` instance is the contract surface. Source-stated fields: `backend`, `indexes[]`, `contentSourceRefs[]`, `ingestionPipelineRef`. The companion project appears as one entry among `contentSourceRefs[]`. No new CRD is introduced by this ADR.

### 4.2 APIs / SDK surfaces
- `rag.*` (Platform SDK, owner B6) — the consumption API the architecture commits to. The companion project is a *producer* into the store, not a `rag.*` caller; its write path/indexing API is companion-project-owned and `[PROPOSED — not in source]`.

### 4.3 CloudEvents emitted / consumed
N/A — ADR 0024 introduces no CloudEvent obligation. Re-index triggers on vendor releases are companion-project-internal; if surfaced into the platform they would ride `platform.capability.*` (RAGStore content change) per ADR 0031, but no such event is committed in source — `[PROPOSED — not in source]`.

### 4.4 Data schemas / connection-secret contracts
- The companion project writes into the same OpenSearch-backed indexes as authored docs (ADR 0009), using the same indexing conventions as C8. Concrete index/document schema is a C8 design-time deliverable; `[PROPOSED — not in source]` beyond "same conventions as authored docs."

## 5. OSS-vs-Custom Decision
N/A — ADR. (Enforcement note: the companion project is explicitly external/out-of-scope for v1.0; the architecture builds only the consumption-side primitive — `platform-knowledge-base` RAGStore on OpenSearch — already owned by ADR 0022/0009.)

## 6. Functional Requirements
- REQ-ADR-0024-01: Vendor-doc acquisition/re-indexing MUST be a separate companion project, NOT a v1.0 architecture component; the platform MUST NOT gate any v1.0 capability on its existence.
- REQ-ADR-0024-02: The architecture's committed surface MUST be limited to the consumption contract — the `platform-knowledge-base` RAGStore and the `rag.*` SDK API.
- REQ-ADR-0024-03: v1.0 MUST ship the Knowledge Base populated from authored docs and runbooks (C8); vendor-doc coverage MUST be able to grow with no architectural change.
- REQ-ADR-0024-04: When the companion project lands, it MUST write into the named `platform-knowledge-base` RAGStore using the same indexing conventions as authored docs and MUST re-index on upstream major/minor releases.
- REQ-ADR-0024-05: The integration point and cross-reference currency MUST be owned by Workstream F item F3; licensing, crawl politeness, upstream version tracking, and freshness SLAs MUST remain the companion project's responsibility.
- REQ-ADR-0024-06: Any companion-project-driven change to the Knowledge Base contract (new metadata fields, new access patterns) MUST go through a new ADR against ADR 0022, not re-litigate this scoping decision.

## 7. Non-Functional Requirements
- Separation of concerns: licensing/redistribution risk stays entirely on the companion-project side of the contract boundary.
- Coverage caveat: until the companion project lands, RAG queries depending on vendor-doc coverage are answered only from authored material — acceptable for v1.0, called out in the documentation plan.
- Versioning (ADR 0030): the `rag.*` SDK surface (B6) and `RAGStore` CRD (B13) version per their owners; the narrow contract pins to those.

## 8. Cross-Cutting Deliverable Checklist
N/A — ADR (verification map lives in the PLAN). The §14.1 set applies to enforcing pieces (C8, F3), not to this scoping record.

## 9. Acceptance Criteria
- AC-ADR-0024-01: Honored when no v1.0 component lists the vendor-doc pipeline as a hard dependency, and the platform installs/operates with it absent. (REQ-01/03)
- AC-ADR-0024-02: Honored when the only architecture-documented vendor-doc surface is the `platform-knowledge-base` RAGStore + `rag.*` API. (REQ-02)
- AC-ADR-0024-03: Honored when v1.0 ships with the Knowledge Base populated from C8 authored docs/runbooks and answers RAG queries. (REQ-03)
- AC-ADR-0024-04: Honored when (companion project present) vendor content appears in the named RAGStore under the same indexing conventions as authored docs and re-indexes on a simulated upstream release. (REQ-04)
- AC-ADR-0024-05: Honored when F3 owns a current cross-reference to the companion project and the companion project (not the platform) owns licensing/freshness. (REQ-05)
- AC-ADR-0024-06: Honored when a hypothetical contract change is routed as a new ADR against ADR 0022 rather than absorbed silently. (REQ-06)

## 10. Risks & Open Questions
- OQ-1 (low): Companion-project ownership and links are filled in later; the contract is narrow but the producer is unspecified until then. `[PROPOSED]`
- R-1 (med): The companion project's choices (which vendors/versions/pin strategy) shape Knowledge Base quality; mitigated by F3 owning coordination so the integration point doesn't drift.
- R-2 (low): Vendor-doc coverage absent in early v1.0 may surprise users expecting third-party answers; mitigated by the documentation-plan call-out.

## 11. References
- ADR 0024 (`adr/0024-vendor-doc-separate-project.md`) — the decision enforced here.
- architecture-overview.md §6.4 (Knowledge Base as a separate primitive), §14.6 (Workstream F; C8, F3).
- README.md "What's NOT in this repository".
- Enforcing/related pieces: C8 (Knowledge Base RAG indexing), F3 (companion-project handoff).
- ADR 0022 (Knowledge Base primitive), 0008 (MkDocs portal), 0009 (OpenSearch).
