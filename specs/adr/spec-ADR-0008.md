# SPEC ADR-0008 — Material for MkDocs as the documentation portal [PROPOSED]

> kind: ADR · workstream: — · tier: T2
> upstream: [] · downstream: [C1;C2;C3;C4;C5;C6;C7;C8;C9] · adrs: [0008] · views: []
> canon-glossary: 6aadcc2a4f38 · canon-interface: 54f5ede58e5f

## 1. Purpose & Problem Statement
ADR 0008 fixes Material for MkDocs as the documentation portal. This SPEC states what honoring that decision requires: documentation is docs-as-code — Markdown sources live in the same repositories as the code they document, ship through normal PR review, and build/deploy as a static site with per-version builds; the emitted Markdown is re-indexable by the C8 pipeline into the `platform-knowledge-base` RAGStore, gated on contributor-flagged major/minor commits. The decision is settled; this SPEC captures the obligations it imposes and the acceptance criteria that prove it is honored.

## 2. Scope
### 2.1 In scope
- Material for MkDocs as the portal (component C1: setup, search, versioning, contribution workflow).
- Docs-as-code: Markdown co-located with code, reviewed in the same PRs, built/deployed as a static site.
- Per-version doc builds (so pinned vendor docs + platform docs serve at deployed-component versions).
- Markdown output re-indexable by C8 into `platform-knowledge-base`, gated on flagged major/minor commits.

### 2.2 Out of scope (and where it lives instead)
- Diataxis content — C2–C5; cross-cutting runbooks — C6; maintainer/extender docs — C7; docs-on-docs — C9.
- The RAG indexing pipeline implementation — C8; Knowledge Base primitive — ADR 0022.
- Per-product docs/runbooks — the owning Workstream A component for each product.
- Vendor-doc acquisition/separation — ADR 0024; portal-vs-Headlamp linking strategy — backlog §1.15 (open).

## 3. Context & Dependencies
Upstream consumed: none (C1 is consumer-tier authoring infra). Downstream consumers: C2–C9 build content/pipelines on the portal.
ADR decisions honored: **0008** — Material for MkDocs portal, docs-as-code, per-version builds, C8-indexable output; **0024** — vendor docs served at pinned versions; **0022** — output feeds the `platform-knowledge-base` RAGStore.

## 4. Interfaces & Contracts
### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
N/A — the portal introduces no CRD/XRD. The `platform-knowledge-base` `RAGStore` is the consuming primitive (owned by the capability surface, indexed by C8), not defined here.

### 4.2 APIs / SDK surfaces
N/A — no SDK surface. The portal emits static Markdown/HTML; the C8 pipeline consumes the Markdown.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
N/A — the portal itself emits no platform CloudEvents. Re-index triggering is contributor-flagged at commit time (a C8 concern), not an event under the `platform.*` taxonomy.

### 4.4 Data schemas / connection-secret contracts
N/A — static-site output; no substrate primitive or connection secret. Re-indexed content lands in OpenSearch via the C8 pipeline (advisory retrieval tier per ADR 0009), reproducible from the Markdown sources in Git.

## 5. OSS-vs-Custom Decision
Upstream project: **Material for MkDocs**, adopted as the portal with docs-as-code in component repos (C1). Mode: install + config (static-site build/deploy). Rationale per ADR 0008: docs-as-code in the same repos, GitOps/PR-review friendly, per-version builds, good search, low-maintenance for a small team; emits Markdown the C8 pipeline can re-index. Rejected: Backstage (heavy; only earns weight with its catalog/plugin ecosystem, which the platform covers via Headlamp + Crossplane), Confluence (breaks docs-as-code/GitOps; WYSIWYG outside the engineer workflow).

## 6. Functional Requirements
- REQ-ADR-0008-01: Documentation MUST be authored as Markdown co-located in the repositories of the code it documents.
- REQ-ADR-0008-02: Docs MUST ship through normal PR review (code and docs reviewed together).
- REQ-ADR-0008-03: The portal MUST build and deploy as a static site with per-version builds.
- REQ-ADR-0008-04: Portal output MUST be Markdown re-indexable by the C8 pipeline into `platform-knowledge-base`.
- REQ-ADR-0008-05: Re-indexing MUST be gated on contributor-flagged major/minor commits; patch-level edits MUST NOT trigger churn.
- REQ-ADR-0008-06: Per-version builds MUST allow pinned vendor docs and platform docs to be served at deployed-component versions.

## 7. Non-Functional Requirements
- Maintainability: low-maintenance static-site portal suited to a small team.
- Authoring: Markdown fluency + local MkDocs preview; non-engineering contributors edit via PRs (accepted trade-off; no WYSIWYG).
- Reproducibility: re-indexed RAG content rebuildable from Git Markdown (consistent with the OpenSearch reproducibility invariant, ADR 0009).
- Scope boundary: no software catalog/scorecards/plugin marketplace (Backstage capabilities deliberately not adopted).

## 8. Cross-Cutting Deliverable Checklist
N/A — ADR. The §14.3 deliverable set is owned by Workstream C (C1 portal infra + C2–C9); conformance items appear in §9 / the PLAN.

## 9. Acceptance Criteria
Decision honored when:
- AC-ADR-0008-01: Docs for a component live in that component's repo and are changed in the same PR as the code. (REQ-01, REQ-02)
- AC-ADR-0008-02: The portal builds a static site with selectable per-version doc builds. (REQ-03, REQ-06)
- AC-ADR-0008-03: A flagged major/minor commit triggers a C8 re-index into `platform-knowledge-base`; a patch commit does not. (REQ-04, REQ-05)
- AC-ADR-0008-04: Pinned vendor docs and platform docs are served at the versions referenced by deployed components. (REQ-06)
- AC-ADR-0008-05: No Backstage software-catalog/plugin surface is shipped by the portal. (REQ — scope boundary)

## 10. Risks & Open Questions
- Portal-vs-Headlamp linking strategy unresolved (blast radius: low) — backlog §1.15 open; this ADR does not preempt it.
- Markdown-only authoring excludes WYSIWYG contributors (low) — accepted trade-off; PR-based editing.
- Catalog need may re-emerge later (low) — a separate decision; can sit alongside the portal.

## 11. References
- ADR 0008 (`adr/0008-material-mkdocs-documentation-portal.md`) — the decision.
- Enforcing components: C1 (portal infrastructure, owner), C2–C7 (content), C8 (RAG indexing pipeline), C9 (docs-on-docs); per-product docs owned by each Workstream A component.
- architecture-overview.md §10, §14.3; architecture-backlog.md §2.6, §1.15.
- Related: ADR 0009, 0022, 0024.
