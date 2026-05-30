# SPEC V6-04 — The Knowledge Base as a separate primitive `[PROPOSED]`

> kind: VIEW · workstream: — · tier: T1
> upstream: [] · downstream: [] · adrs: [0008, 0022, 0024] · views: [6.4]
> canon-glossary: 6aadcc2a4f38 · canon-interface: 54f5ede58e5f

## 1. Purpose & Problem Statement

This view defines the **integration contract** for the Knowledge Base slice: the architectural commitment that the Knowledge Base is a **first-class `RAGStore` named `platform-knowledge-base`** — a shared primitive any Platform Agent may include in its CapabilitySet — and is **independent of HolmesGPT, Coach, and any other agent** (ADR 0022). It is not a buildable component; it is realized by the Knowledge Base RAG indexing pipeline (C8) and the OpenSearch store (A11) it indexes into.

The problem it solves: if platform knowledge were baked into a specific agent (e.g. HolmesGPT), every other agent would need a copy and the knowledge would fragment. By making the Knowledge Base a single named `RAGStore` reached through the standard SDK `rag.*` path, every agent — interactive, self-management, self-improvement — reaches the same knowledge the same way, with the same governance. This view fixes the singleton-primitive invariant, the content-source set, the human-judgment re-indexing rule, and the access-path invariant.

## 2. Scope
### 2.1 In scope
- The singleton-primitive invariant: the Knowledge Base is the `platform-knowledge-base` `RAGStore`, included via CapabilitySet, not built into any agent (ADR 0022).
- The content-source set: platform docs (the MkDocs portal), per-product architecture docs + runbooks, pinned vendor docs.
- The re-indexing rule: human-judgment-driven; major/minor release-flagged commits trigger re-indexing; patch-level does not.
- Vendor-doc acquisition/re-indexing ownership: a separate companion project (referenced, not in v1.0 scope — ADR 0024).
- The access-path invariant: agents reach the KB through the SDK `rag.*` API over the standard LiteLLM call path; LibreChat reaches it via the Interactive Access Agent.
- The agent-implementation choice (SDK-API vs filesystem mount) is design-time per agent.

### 2.2 Out of scope (and where it lives instead)
- General `RAGStore` reconciliation into the gateway — view V6-01 / component B13.
- Storage roles / OpenSearch retrieval-optimization invariant — view V6-03 / ADR 0009 / ADR 0014.
- The MkDocs documentation portal itself — component C1 / ADR 0008.
- The Interactive Access Agent implementation — component A16 (consumed, not defined here).
- Vendor-documentation companion project internals — ADR 0024 (referenced, out of v1.0 scope).
- The indexing-pipeline implementation detail — component C8 (consumed, not defined here).

## 3. Context & Dependencies

Realizing components and what each contributes to the slice:
- **C8 Knowledge Base RAG indexing pipeline** — ingests the content sources into the `platform-knowledge-base` `RAGStore` and applies the major/minor re-indexing rule.
- **A11 OpenSearch** — the vector + hybrid retrieval backend the `RAGStore` indexes into (per view V6-03 storage roles; OpenSearch remains retrieval-optimization only).

ADR decisions honored:
- **ADR 0022** — Knowledge Base is a separate primitive: a `RAGStore` independent of any agent.
- **ADR 0008** — Material for MkDocs is the documentation portal (a content source).
- **ADR 0024** — vendor-documentation acquisition is a separate companion project, referenced but not part of v1.0 scope.

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
Consumed (owned by B13):
- `RAGStore` — `backend`, `indexes[]`, `contentSourceRefs[]`, `ingestionPipelineRef`. The Knowledge Base is the specific instance named `platform-knowledge-base`.
- `CapabilitySet` — `ragStores[]` (the field through which an agent includes the Knowledge Base).

All namespaced; versioning per ADR 0030; owner of versioning lifecycle = B13.

### 4.2 APIs / SDK surfaces
- **Platform SDK (B6)** `rag.*` — the agent-facing retrieval surface; KB queries terminate at LiteLLM on the standard call path (defined in V6-01; consumed here).
- LibreChat → Interactive Access Agent (A16) → `rag.*` — the developer access path to the KB.
- Optional filesystem mount of KB content (for structured grep) is a per-agent design-time alternative to the SDK API; both are governed paths.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
Emitted by the KB slice (per-event-type names deferred to B12 registry):
- `platform.capability.*` — `RAGStore` registry changes for `platform-knowledge-base` (add/update/delete), per the capability-registry taxonomy.
- `platform.audit.*` — KB queries ride the LiteLLM call path and are audited there via the adapter (no separate emission point).

### 4.4 Data schemas / connection-secret contracts
- KB content is indexed into OpenSearch (vectors + hybrid) and re-derivable from primary content sources per the view V6-03 reproducibility invariant — `[PROPOSED — not in source]` for any KB-specific index schema beyond `RAGStore.indexes[]`.
- Re-indexing trigger is a commit-level major/minor release flag (human-judgment); the flag mechanism is design-time in C8.

## 5. OSS-vs-Custom Decision
N/A — VIEW (OSS-vs-custom for the indexing pipeline and OpenSearch lives in component SPECs C8, A11).

## 6. Functional Requirements
Each requirement is an **invariant/constraint this view imposes** on every participating component.

- **REQ-V6-04-01:** The Knowledge Base MUST be a single first-class `RAGStore` named `platform-knowledge-base`, independent of HolmesGPT, Coach, and any other agent (ADR 0022). (§6.4, line ~351)
- **REQ-V6-04-02:** Any Platform Agent MUST be able to include the Knowledge Base by referencing it in its CapabilitySet (`ragStores[]`); it MUST NOT be built into an agent image. (§6.4, lines ~351, ~367)
- **REQ-V6-04-03:** The KB content sources MUST be: the platform's own documentation (the MkDocs portal), per-product architecture docs + runbooks, and pinned vendor-documentation versions. (§6.4, lines ~353–357)
- **REQ-V6-04-04:** Re-indexing of platform-authored docs MUST be human-judgment-driven: commits flagged major or minor trigger re-indexing; patch-level changes MUST NOT trigger re-indexing. (§6.4, line ~361)
- **REQ-V6-04-05:** Vendor-documentation acquisition and re-indexing MUST be owned by a separate companion project (referenced, not v1.0 scope per ADR 0024); the platform only consumes pinned vendor-doc versions. (§6.4, line ~362)
- **REQ-V6-04-06:** Every agent MUST reach the Knowledge Base through the SDK `rag.*` API over the standard LiteLLM call path; LibreChat reaches it via the Interactive Access Agent (A16). (§6.4, lines ~366–367)
- **REQ-V6-04-07:** The implementation choice between SDK-API and filesystem mount MUST be a per-agent design-time decision; both MUST remain governed paths. (§6.4, line ~368)

## 7. Non-Functional Requirements
- **Security/multi-tenancy (§6.9):** KB access is governed by CapabilitySet inclusion and the standard LiteLLM/OPA/audit path; an agent without the KB in its CapabilitySet cannot reach it.
- **Observability (§6.5):** KB queries are audited via the adapter on the LiteLLM path; `RAGStore` registry changes emit `platform.capability.*`.
- **Scale / freshness:** the major/minor re-indexing rule bounds indexing churn (no re-index on typo/patch); KB index is re-derivable from primary sources (view V6-03).
- **Versioning (ADR 0030):** `RAGStore`/`CapabilitySet` versioning owned per-component by B13; vendor docs are version-pinned.

## 8. Cross-Cutting Deliverable Checklist
N/A — VIEW (cross-cutting deliverables are owned by realizing components C8, A11).

## 9. Acceptance Criteria
The view holds when:
- **AC-V6-04-01:** Exactly one `RAGStore` named `platform-knowledge-base` exists and is referenced by no agent image build (only by CapabilitySets). (→ REQ-01, REQ-02)
- **AC-V6-04-02:** Two different agents (e.g. the Interactive Access Agent and HolmesGPT) reach the same KB content via the same `rag.*` path; neither carries a private copy. (→ REQ-02, REQ-06)
- **AC-V6-04-03:** The KB index contains content from all three source classes (platform docs, per-product docs/runbooks, pinned vendor docs). (→ REQ-03)
- **AC-V6-04-04:** A commit flagged patch does NOT trigger re-indexing; a commit flagged major or minor DOES. (→ REQ-04)
- **AC-V6-04-05:** Vendor-doc re-indexing is driven by the companion project's signal, not by the v1.0 platform pipeline. (→ REQ-05)
- **AC-V6-04-06:** A developer in LibreChat reaches the KB only through the Interactive Access Agent; an agent without the KB in its CapabilitySet cannot query it. (→ REQ-06, REQ-02)
- **AC-V6-04-07:** An agent configured for filesystem-mount access and one configured for SDK-API access both reach the KB through governed paths. (→ REQ-07)

## 10. Risks & Open Questions
- **OQ-1 (med):** The companion project for vendor docs (ADR 0024) has no v1.0 ownership/links; this view only references it. Resolved when the companion project is scoped (Workstream F handoff, F3).
- **OQ-2 (low):** KB-specific index schema beyond `RAGStore.indexes[]` is `[PROPOSED — not in source]`; resolved in C8.
- **R-1 (med):** Human-judgment release flagging can be inconsistent (a major change mis-flagged patch leaves the KB stale); mitigated by making the flag a reviewable commit attribute and surfacing index freshness.
- **R-2 (low):** Filesystem-mount access could drift from the SDK-API governance model; mitigated by REQ-07 requiring both to remain governed.

## 11. References
- architecture-overview.md §6.4 The Knowledge Base as a separate primitive (lines ~349–368); §6.3 (OpenSearch retrieval-optimization role); §6.1 (LiteLLM `rag.*` call path).
- ADRs: 0022 (Knowledge Base as a separate primitive), 0008 (Material for MkDocs portal), 0024 (vendor-documentation companion project).
- Realizing components: C8 (KB RAG indexing pipeline), A11 (OpenSearch); access via A16 (Interactive Access Agent), B6 (Platform SDK `rag.*`), B13 (`RAGStore`/`CapabilitySet`).
- Related views: V6-01, V6-03, V6-08.
