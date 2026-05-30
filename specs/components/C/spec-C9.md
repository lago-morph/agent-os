# SPEC C9 — Docs-on-docs  [PROPOSED]

> kind: COMPONENT · workstream: C · tier: T2
> upstream: [C1, C8] · downstream: [] · adrs: [0008, 0022, 0024, 0012] · views: [6.4]
> canon-glossary: see _meta/glossary.md · canon-interface: see _meta/interface-contract.md

## 1. Purpose & Problem Statement

A small but load-bearing section explaining how to create, maintain, review, and find
documentation (§10.8): the PR-based contribution workflow, MkDocs conventions, where each Diataxis
quadrant lives, how docs get indexed into the Knowledge Base, and how to query the Knowledge Base
from LibreChat or HolmesGPT. Without it, contributors lack a single entry point for "how does
documentation work here", and the docs-as-code commitment (ADR 0008) and the C8 indexing
behavior stay tacit. C9 is concise narrative content authored into the C1 portal; it ships no code.

## 2. Scope

### 2.1 In scope
- The **docs-on-docs** portal section (§10.8): contribution workflow (PR-based); MkDocs
  conventions; the map of where each Diataxis quadrant lives (§10.1–§10.4) plus per-product docs
  (§10.5), maintainer docs (§10.6), runbooks (§10.7); how docs get indexed into the Knowledge Base
  (pointing at C8's major/minor trigger, §6.4); and how to query `platform-knowledge-base` from
  **LibreChat** (via the Interactive Access Agent) or **HolmesGPT** (ADR 0022 / 0012).

### 2.2 Out of scope
- **Portal infrastructure / contribution-workflow CI** — **C1** (C9 narrates it).
- **The indexing pipeline itself** — **C8** (C9 documents the behavior, not the mechanism).
- Diataxis content (C2–C5), runbooks (C6 / Workstream A), maintainer docs (C7).
- Vendor-doc acquisition — separate companion project (ADR 0024).

## 3. Context & Dependencies
Upstream consumed (HARD): **C1** (portal + §10.8 slot + contribution-workflow conventions C9
describes); **C8** (the indexing behavior + major/minor trigger C9 explains). Downstream: none —
C9 is a leaf. Continuous input: all other C pieces define the quadrant map C9 points at.

ADR decisions honored:
- **ADR 0008** — Material for MkDocs; PR-based docs-as-code; C9 is Markdown-in-repo.
- **ADR 0022 / 0012** — KB query access is via `platform-knowledge-base` through the Interactive
  Access Agent (LibreChat) or HolmesGPT on the `rag.*` LiteLLM-mediated path; C9 documents this.
- **ADR 0024** — vendor docs are a separate companion project; C9 notes the boundary.

## 4. Interfaces & Contracts
### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
N/A — C9 defines no CRD/XRD; it references `platform-knowledge-base` (`RAGStore`) by Canon name.
### 4.2 APIs / SDK surfaces
N/A — C9 owns no API. It documents how to reach the KB (LibreChat / HolmesGPT, `rag.*`); the
`rag.*` API is B6-owned. `[PROPOSED — not in source]` for any concrete page template convention.
### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
N/A — C9 emits/consumes no CloudEvents (the re-index trigger is C8's).
### 4.4 Data schemas / connection-secret contracts
N/A — content only; the corpus shape it ships under is C1/C8-owned.

## 5. OSS-vs-Custom Decision
**Build-new content on configured OSS (Material for MkDocs, ADR 0008); no fork.** C9 is authored
Markdown, not software; it reuses the C1 portal and C8 indexing path.

## 6. Functional Requirements
- REQ-C9-01: C9 SHALL document the PR-based documentation contribution workflow and MkDocs
  conventions (§10.8, ADR 0008).
- REQ-C9-02: C9 SHALL map where each Diataxis quadrant and the per-product/maintainer/runbook
  sections live (§10.1–§10.7).
- REQ-C9-03: C9 SHALL explain how docs get indexed into `platform-knowledge-base`, referencing the
  C8 major/minor-flag trigger and patch-exclusion (§6.4).
- REQ-C9-04: C9 SHALL explain how to query `platform-knowledge-base` from LibreChat (via the
  Interactive Access Agent) and from HolmesGPT on the `rag.*` path (ADR 0022 / 0012).
- REQ-C9-05: C9 SHALL be Markdown-in-repo in the C1 §10.8 section and re-indexable by C8.

## 7. Non-Functional Requirements
- Security: no secrets in content. Multi-tenancy (§6.9): platform-wide reference, no tenant data.
- Observability (§6.5): N/A — static content. Scale: small section, additive.
- Versioning (ADR 0030): major/minor doc changes trigger C8 re-index; patch does not (§6.4).

## 8. Cross-Cutting Deliverable Checklist
- Helm/manifests: **N/A — content; build/deploy is C1.**
- Per-product docs (10.5): **N/A — Workstream A.**
- Runbook (10.7): **N/A — C6 / Workstream A.**
- Alerts: **N/A — static content.**
- Grafana dashboard (Crossplane XR): **N/A — no metrics.**
- Headlamp plugin: **N/A — no UI surface.**
- OPA/Rego integration: **N/A — no policy surface.**
- Audit emission (ADR 0034): **N/A — emits nothing.**
- Knative trigger flow: **N/A — re-index trigger is C8.**
- HolmesGPT toolset: **N/A — C9 documents KB query access; ships no toolset.**
- 3-layer tests (Chainsaw/Playwright/PyTest): **Partial — PyTest/link-lint for build + link validity (via C1); Playwright for nav/search of the §10.8 section; Chainsaw N/A.**
- Tutorials & how-tos: **N/A — C2/C3.**

## 9. Acceptance Criteria
- AC-C9-01 (REQ-C9-01): The docs-on-docs section documents the PR-based workflow and MkDocs
  conventions, and builds with zero errors.
- AC-C9-02 (REQ-C9-02): The section contains a quadrant/section map covering §10.1–§10.7.
- AC-C9-03 (REQ-C9-03): The section explains the major/minor re-index trigger and that patch
  commits do not trigger (§6.4), referencing C8.
- AC-C9-04 (REQ-C9-04): The section explains querying `platform-knowledge-base` from LibreChat
  (Interactive Access Agent) and HolmesGPT.
- AC-C9-05 (REQ-C9-05): The section renders in the C1 §10.8 slot and is ingested by C8 with no
  preprocessing.

## 10. Risks & Open Questions
- R1 (low): Page template convention is `[PROPOSED — not in source]`; trivial for a small section.
- R2 (low): Must stay in sync with C8's trigger semantics and C1's workflow; on change, re-flag
  major/minor to re-index. Blast radius low (T2, leaf, no downstream).

## 11. References
- architecture-overview.md §10.8 Docs-on-docs (~L1533–1535); §10 contribution workflow / docs-as-code
  (~L1429–1431); §6.4 Knowledge Base access patterns (~L364–368).
- architecture-overview.md §14.3 Workstream C, C9 row (~L1732).
- ADR 0008 (Material for MkDocs), ADR 0022 (Knowledge Base separate primitive), ADR 0012 (HolmesGPT),
  ADR 0024 (vendor docs separate companion project).
- _meta/glossary.md (Knowledge Base, `platform-knowledge-base`, LibreChat, HolmesGPT, Interactive
  Access Agent). _meta/interface-contract.md §1.4 (`RAGStore`), §3 (`rag.*` via B6).
