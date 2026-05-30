# SPEC C1 — Documentation portal infrastructure

> kind: COMPONENT · workstream: C · tier: T1
> upstream: [] · downstream: [C2, C3, C4, C5, C6, C7, C8, C9] · adrs: [0008, 0030, 0024, 0010, 0033] · views: [6.4]
> canon-glossary: see _meta/glossary.md · canon-interface: see _meta/interface-contract.md

## 1. Purpose & Problem Statement

Documentation is a first-class platform deliverable, not an afterthought (§10). Every
component ships Diataxis content, per-product docs, runbooks, and maintainer docs as Markdown
in the same repo as the code it documents, and the whole corpus is re-indexed as the
`platform-knowledge-base` `RAGStore` (§6.4, §10, ADR 0008). Without a single, low-maintenance
portal with consistent structure, search, versioning, and a PR-based contribution workflow,
each component would diverge on layout and tooling, the docs-as-code commitment would erode,
and the C8 indexing pipeline would have no stable Markdown surface to ingest.

C1 is the portal **infrastructure** piece: the **Material for MkDocs** site configuration, the
Diataxis-aligned information architecture and navigation skeleton, in-portal search, per-version
builds, and the PR-based contribution workflow (linting, preview builds, conventions) that every
other Workstream C content piece (C2–C9) and every Workstream A/B per-product doc author plugs
into. C1 ships infrastructure and conventions only — it authors **no** Diataxis content itself;
that is owned by C2–C5 (and C6/C7/C9).

## 2. Scope

### 2.1 In scope
- **Material for MkDocs** site setup (ADR 0008): `mkdocs.yml` configuration, theme/branding,
  Markdown-in-repo docs-as-code layout, static-site build.
- The Diataxis-aligned information architecture and **navigation skeleton** — the four quadrants
  (tutorials §10.1, how-to §10.2, reference §10.3, explanation §10.4) plus the slots for
  per-product docs (§10.5), maintainer/extender docs (§10.6), runbooks (§10.7), and docs-on-docs
  (§10.8). C1 owns the empty structure; C2–C9 fill it.
- **In-portal search** over the published corpus.
- **Per-version doc builds** (§10, ADR 0008) so platform docs and pinned vendor docs are served
  at the versions referenced by deployed components (ADR 0024).
- **PR-based contribution workflow**: the docs-as-code review path, MkDocs/Markdown conventions,
  link/build/lint checks, and local preview, run as a GitHub Actions check (ADR 0010 / 0033).
- A clean, re-indexable Markdown output surface that the C8 pipeline consumes (§14.3 C8).

### 2.2 Out of scope
- Diataxis content authoring — **C2** (tutorials), **C3** (how-to), **C4** (reference),
  **C5** (explanation).
- Cross-cutting / integrated runbooks content — **C6**. Per-product runbooks (§10.7) and
  per-product docs (§10.5) — **Workstream A** components that own each product.
- Maintainer / extender docs content — **C7**. Docs-on-docs content — **C9** (C1 provides the
  section slot; C9 writes the contribution-workflow narrative).
- The Knowledge Base RAG **indexing pipeline** and its `platform-knowledge-base` `RAGStore`
  wiring — **C8** (C1 only emits the Markdown it ingests).
- Vendor documentation acquisition / indexing — a separate companion project (ADR 0024), not in
  this architecture's scope.
- The portal-vs-Headlamp linking strategy — **deferred** (architecture-backlog §1.15; ADR 0008
  explicitly does not preempt it).

## 3. Context & Dependencies

Upstream consumed: none as a hard blocker — C1 is authoring-parallel (`consumer` wave) and
stands up the portal before content lands.

Continuous (non-blocking) inputs: **every Workstream A/B component** continuously contributes
docs (per-product docs §10.5, runbooks §10.7, maintainer docs §10.6) into the slots C1 defines;
recorded as continuous inputs, not blockers.

Downstream consumers: **C2–C5** (Diataxis content), **C6** (runbooks), **C7** (maintainer
docs), **C8** (indexing pipeline — consumes C1's Markdown output), **C9** (docs-on-docs).

ADR decisions honored:
- **ADR 0008** — Material for MkDocs is the portal; Markdown-in-repo docs-as-code; per-version
  builds; PR review covers code and docs together; Markdown output re-indexable by C8.
- **ADR 0024** — vendor doc acquisition is a separate companion project; C1 reserves a slot for
  pinned vendor docs at matching versions but does not acquire or index them.
- **ADR 0010 / 0033** — GitHub Actions is the v1.0 CI; the docs build/lint/preview check runs as
  a GitHub Actions workflow against GitHub as the source surface.
- **ADR 0030** — versioning policy; the portal's per-version builds align doc versions to the
  component/API versions they document.

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
N/A — C1 defines no CRD/XRD. It emits Markdown consumed by the `platform-knowledge-base`
`RAGStore` (Canon, §6.4) via the **C8** pipeline; the `RAGStore` itself is owned by C8 / its
backing component, not C1.

### 4.2 APIs / SDK surfaces
N/A — C1 exposes no programmatic API or SDK surface. Its "interface" to downstream content
pieces is the navigation skeleton, the MkDocs/Markdown authoring conventions, and the
contribution-workflow checks. `[PROPOSED — not in source]` for any specific directory-layout
convention (e.g. one quadrant per top-level docs directory) — the four-quadrant *structure* is
source-stated (§10.1–§10.4), but the concrete folder names are an invented authoring convention.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
N/A — C1 is a static-site portal and emits no CloudEvents. (C8 re-indexing is triggered by
contributor-flagged major/minor commits per §14.3 C8 / ADR 0008, owned by C8, not C1.)

### 4.4 Data schemas / connection-secret contracts
N/A — the portal is a static build with no data backend and writes no connection secret. The
only data contract is the **Markdown corpus shape** the C8 indexer ingests; the indexer/RAGStore
schema is owned by C8.

## 5. OSS-vs-Custom Decision
**Configure (wrap) an OSS project — Material for MkDocs (ADR 0008); do not fork.** C1 provides
`mkdocs.yml`, theme/branding config, the navigation skeleton, search config, the per-version
build wiring, and the GitHub Actions contribution-workflow checks around it. Rationale (ADR
0008): beautiful out of the box, low-maintenance for a small team, native Markdown-in-repo
docs-as-code, good versioning and search, and Markdown output the C8 pipeline can re-index.
Backstage and Confluence were rejected (ADR 0008 alternatives).

## 6. Functional Requirements
- REQ-C1-01: The portal SHALL be built with Material for MkDocs from Markdown sources committed
  in-repo alongside the code they document (ADR 0008, §10).
- REQ-C1-02: The portal SHALL expose a navigation structure with a dedicated top-level section
  for each Diataxis quadrant — tutorials, how-to guides, reference, explanation (§10.1–§10.4) —
  plus reserved sections for per-product docs (§10.5), maintainer/extender docs (§10.6),
  runbooks (§10.7), and docs-on-docs (§10.8).
- REQ-C1-03: The portal SHALL provide in-portal full-text search over the published corpus.
- REQ-C1-04: The portal SHALL support per-version builds so platform docs and pinned vendor docs
  are servable at the versions referenced by deployed components (§10, ADR 0008, ADR 0024).
- REQ-C1-05: The portal SHALL define a PR-based contribution workflow in which documentation
  changes are reviewed in the same PR as related code (docs-as-code, ADR 0008).
- REQ-C1-06: The contribution workflow SHALL run an automated GitHub Actions check that builds
  the site, validates internal links, and lints Markdown/MkDocs conventions, failing the PR on
  build or broken-link errors (ADR 0010 / 0033).
- REQ-C1-07: The portal SHALL emit a clean Markdown corpus that the C8 indexing pipeline can
  re-index into the `platform-knowledge-base` `RAGStore` without portal-specific preprocessing
  (§10, §14.3 C8, ADR 0008).
- REQ-C1-08: C1 SHALL document the MkDocs/Markdown authoring conventions and the local-preview
  workflow so any Workstream A/B/C contributor can add docs without infrastructure changes
  (the docs-on-docs narrative itself is C9; C1 ships the conventions it references).
- REQ-C1-09: The portal SHALL reserve a slot for per-version pinned vendor docs without acquiring
  or indexing them, leaving acquisition to the separate companion project (ADR 0024).
- REQ-C1-10: Doc build versions SHALL be alignable to the component/API versions they document,
  consistent with the platform versioning policy (ADR 0030).

## 7. Non-Functional Requirements
- Security: the contribution workflow runs as a security-first GitHub Actions check (SHA-pinned
  actions, scoped tokens) consistent with the v1.0 pipeline posture (ADR 0010); the portal serves
  static content with no secrets embedded.
- Multi-tenancy (§6.9): the portal is platform-wide reference content, not tenant-scoped; it
  carries no tenant data. (Tenant-visible dashboards/UIs are owned elsewhere.)
- Observability (§6.5): build success/failure is visible via the GitHub Actions check; no runtime
  service metrics, as the portal is a static site.
- Scale: static-site build scales with corpus size only; per-version builds are additive.
- Versioning (ADR 0030): per-version doc builds track documented component/API versions;
  navigation-skeleton changes are additive and backward-compatible for content pieces.

## 8. Cross-Cutting Deliverable Checklist
- Helm/manifests: **Applicable** — static-site build + deploy config (GitOps-published).
- Per-product docs (10.5): **N/A — C1 reserves the §10.5 slot; per-product docs are authored by Workstream A owners.**
- Runbook (10.7): **N/A — C1 reserves the runbook section; content is C6 + Workstream A.**
- Alerts: **N/A — static portal has no runtime failure conditions to alert on.**
- Grafana dashboard (Crossplane XR): **N/A — no per-component runtime metrics.**
- Headlamp plugin: **N/A — portal-vs-Headlamp linking is deferred (backlog §1.15, ADR 0008).**
- OPA/Rego integration: **N/A — static content portal has no admission/runtime policy surface.**
- Audit emission (ADR 0034): **N/A — no audit-relevant runtime actions; the portal is a static build.**
- Knative trigger flow: **N/A — re-index triggering on flagged commits is owned by C8, not C1.**
- HolmesGPT toolset: **N/A — KB query access is via the `platform-knowledge-base` RAGStore (C8), not a C1 toolset.**
- 3-layer tests (Chainsaw/Playwright/PyTest): **Partial** — Playwright for portal nav/search e2e; PyTest for build/link-lint checks; Chainsaw **N/A — no CRD**.
- Tutorials & how-tos: **N/A — C1 provides the slots; the content is C2/C3.**

## 9. Acceptance Criteria
- AC-C1-01 (REQ-C1-01): `mkdocs build` produces a static site from in-repo Markdown with zero
  build errors.
- AC-C1-02 (REQ-C1-02): The built site's navigation contains a distinct top-level section for
  each of the four Diataxis quadrants and reserved sections for per-product docs, maintainer
  docs, runbooks, and docs-on-docs.
- AC-C1-03 (REQ-C1-03): A search query for a known term in published content returns the correct
  page in the in-portal search.
- AC-C1-04 (REQ-C1-04): Two distinct version builds of the same page are servable and selectable
  via the version selector.
- AC-C1-05 (REQ-C1-05): A docs change merges through the standard PR review path with the docs
  diff reviewed alongside code.
- AC-C1-06 (REQ-C1-06): A PR introducing a broken internal link or a build error fails the
  GitHub Actions docs check.
- AC-C1-07 (REQ-C1-07): The C8 pipeline ingests the C1 Markdown output and indexes it into
  `platform-knowledge-base` with no portal-specific preprocessing (verified jointly with C8).
- AC-C1-08 (REQ-C1-08): A contributor following the documented conventions adds a new page that
  builds and appears in nav without editing portal infrastructure.
- AC-C1-09 (REQ-C1-09): A pinned-vendor-docs slot exists and renders a placeholder/version
  marker without C1 acquiring vendor content.
- AC-C1-10 (REQ-C1-10): A doc build can be tagged to a documented component/API version per the
  versioning policy.

## 10. Risks & Open Questions
- R1 (med): Concrete docs **directory layout / folder names** are not Canon — `[PROPOSED — not in
  source]`. The four-quadrant *structure* is source-stated; folder naming is a C1 authoring
  convention C2–C9 must adopt verbatim. Reconciliation note: C1 owns and publishes the layout.
- R2 (med): Portal-vs-Headlamp linking strategy is **deferred** (backlog §1.15, ADR 0008). Open
  question: where the portal is surfaced inside the operator UI. Blast radius bounded — does not
  block content authoring.
- R3 (low): Per-version build mechanics interact with vendor-doc pinning (ADR 0024) whose
  acquisition is out of scope. Open question: how vendor-doc version pins are fed in once the
  companion project exists; C1 reserves the slot only.
- R4 (low): The exact "clean Markdown" contract the C8 indexer needs (front-matter, heading
  rules) is co-owned with C8. Reconcile the corpus shape jointly so re-indexing is preprocessing-free.

## 11. References
- architecture-overview.md §10 Documentation plan (~L1425–1436; Material for MkDocs + RAGStore
  indexing L1429; doc timing L1431; docs-before-implementation L1433).
- architecture-overview.md §10.1–§10.4 Diataxis quadrants (~L1435–1502); §10.5 per-product
  (~L1504); §10.6 maintainer (~L1518); §10.7 runbooks (~L1527); §10.8 docs-on-docs (~L1533).
- architecture-overview.md §14.3 Workstream C table, C1 + C8 rows (~L1718–1732).
- architecture-overview.md §6.4 Knowledge Base as a separate primitive (`platform-knowledge-base`
  RAGStore).
- ADR 0008 (Material for MkDocs), ADR 0024 (vendor docs separate companion project),
  ADR 0010 (GitHub Actions only), ADR 0033 (AWS + GitHub initial targets), ADR 0030 (versioning).
- _meta/glossary.md (Material for MkDocs, Knowledge Base, `platform-knowledge-base`, RAGStore).
- _meta/interface-contract.md §1 (RAGStore — owned downstream), §2 (CloudEvent taxonomy).
