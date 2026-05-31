# SPEC C7 — Maintainer / extender documentation for all custom code  [PROPOSED]

> kind: COMPONENT · workstream: C · tier: T1
> upstream: [C1] · downstream: [C8] · adrs: [0008, 0006, 0019, 0030, 0044] · views: [6.4]
> canon-glossary: see _meta/glossary.md · canon-interface: see _meta/interface-contract.md

## 1. Purpose & Problem Statement

Every custom-developed component ships maintainer/extender documentation (§10.6): user-facing
usage docs, maintainer/extender docs (code structure, design rationale, how to add features), an
API reference, and a test guide + contribution guide. Without a consistent maintainer-doc
standard, each custom component (the kopf operator, Headlamp plugins, the platform SDK, agent base
images, LiteLLM callbacks, Coach, custom HolmesGPT toolsets, glue services, Crossplane
Compositions) would document its internals differently or not at all, blocking external extenders
and future maintainers and starving the Knowledge Base of "how our code works" content.

C7 owns the **maintainer/extender documentation deliverable across all custom code** (§10.6,
§14.3 C7) — the section structure, the per-component template (the four §10.6 artifacts), and the
authoring conventions — plus authoring/curation of the maintainer docs themselves into the C1
portal as Markdown, re-indexed into `platform-knowledge-base` by C8. The per-product
*architecture-specific* docs (§10.5) and per-product *runbooks* (§10.7) are NOT C7 — those belong
to Workstream A. C7 is the maintainer/extender slice for **custom** (Workstream B–style) code.

## 2. Scope

### 2.1 In scope
- The portal's **maintainer/extender documentation section** (§10.6) and the per-component
  template enforcing the four §10.6 artifacts: **user-facing usage docs**, **maintainer/extender
  docs** (code structure, design rationale, how to add features), **API reference**, and **test
  guide + contribution guide**.
- Coverage of **every custom-developed component** named in §10.6: the kopf operator (B13),
  Headlamp plugins (B5 + framework A22), the platform SDK (B6), agent base images (B7), LiteLLM
  callbacks (B2), Coach (B10), custom HolmesGPT toolsets, glue services, Crossplane Compositions (B4).
- Authoring conventions for documenting custom code's **API reference** consistently with the
  versioning policy (ADR 0030) — CRD/XRD API versions, SDK semver, URL-path-versioned HTTP APIs.
- A docs structure C8 can re-index into `platform-knowledge-base` (§14.3 C8) so HolmesGPT and the
  Interactive Access Agent can answer "how does our custom code work / how do I extend it".

### 2.2 Out of scope
- **Per-product architecture-specific docs** (§10.5) — owned by **Workstream A** product owners
  (deployment shape, conventions, SSO, troubleshooting for installed products).
- **Per-product runbooks** (§10.7) and **cross-cutting runbooks** — Workstream A and **C6**.
- **Diataxis tutorials / how-to / reference / explanation** (§10.1–§10.4) — **C2–C5**. (C7's
  "API reference" is the maintainer-facing internal API of custom code, distinct from C4's
  product/SDK reference; the boundary is reconciled with C4 — see R1.)
- **Portal infrastructure / section slot** — **C1**; C7 fills the §10.6 slot.
- **Indexing into `platform-knowledge-base`** — **C8**; C7 emits Markdown only.
- **Docs-on-docs** (how docs are written/reviewed/found) — **C9**.
- The **substance** of each custom component's design (the authoritative source is each
  component's own spec) — C7 documents and curates; it does not redesign components.

## 3. Context & Dependencies

Upstream consumed (HARD): **C1** — portal skeleton, the reserved §10.6 maintainer-docs section,
and MkDocs/Markdown authoring conventions.

Continuous (non-blocking) inputs: **every custom component** (B-series + A22 framework) feeds its
own maintainer/extender docs; per §10.6 and §10 doc-timing, the docs are constructed **before
implementation** as the source of test cases — so C7's template and section are needed early, and
component-specific maintainer content lands with each component. Partial early authoring is
encouraged to reduce the final-component burden (§10).

Downstream consumers: **C8** (re-indexes maintainer docs); external **extenders/maintainers** and
**HolmesGPT** / **Interactive Access Agent** retrieving them via `platform-knowledge-base`.

ADR decisions honored:
- **ADR 0008** — Material for MkDocs; maintainer docs are Markdown-in-repo alongside the custom
  code they document, re-indexable by C8.
- **ADR 0006** — the kopf operator is a subchart of LiteLLM; its maintainer docs document that
  packaging/lifecycle binding.
- **ADR 0019** — agent base images cover LangGraph (`langgraph`) and Langchain Deep Agents
  (`deep-agents`); maintainer docs for B7 use these Canon SDK names.
- **ADR 0030** — API reference for custom code follows the versioning policy: per-component CRD/XRD
  API versions, SDK semver + compatibility matrix, URL-path-versioned custom HTTP APIs.
- **ADR 0044** — Crossplane Compositions' maintainer docs document the XRD contract (uniform
  connection-secret shape, substrate-agnostic XR status, one Composition per substrate).

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
N/A — C7 defines no CRD/XRD. Its API-reference content **documents** Canon CRDs/XRDs owned by
their reconciler components (e.g. `MCPServer`/`CapabilitySet`/`VirtualKey` for the B13 kopf
operator; `Postgres`/`AuditLog` for B4 Compositions) using verbatim Canon field names from
`_meta/interface-contract.md`; it owns none and invents no fields.

### 4.2 APIs / SDK surfaces
N/A as an *owned* surface — C7 documents existing surfaces: the **Platform SDK** (`memory.*`,
`rag.*`, OTel emission, A2A registration helpers; B6) and the **agent SDK** (`langgraph`,
`deep-agents`; B7), plus custom HTTP APIs (kopf admin, Knative adapters, audit endpoint) under
URL-path versioning (ADR 0030 §3.3). `[PROPOSED — not in source]` for any maintainer-doc page
template / API-reference layout convention; specific SDK method signatures are **not specified in
source** (interface-contract §3, §6) and C7 documents only the named surface groups, flagging gaps.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
N/A — C7 emits/consumes no CloudEvents. Maintainer docs for event-emitting custom components
reference the `platform.*` namespaces and point to **B12**'s schema registry for per-event-type
schemas; C7 invents no event types.

### 4.4 Data schemas / connection-secret contracts
N/A — C7 holds no data backend / writes no connection secret. Crossplane-Composition maintainer
docs **document** the uniform connection-secret contract (`host`, `port`, `user`, `password`,
`dbname`; ADR 0044) as authored by B4; C7 reproduces it, does not define it. The only C7-owned
contract is the Markdown corpus shape C8 ingests (co-owned with C8).

## 5. OSS-vs-Custom Decision
**Build-new content on top of configured OSS (Material for MkDocs, ADR 0008); no fork.** C7 is
authored Markdown maintainer/extender documentation and a per-component template, not software. It
reuses the C1 portal and C8 indexing path. Rationale (ADR 0008): docs-as-code in-repo next to the
custom code, re-indexable into the Knowledge Base. No tooling decision is owned here.

## 6. Functional Requirements
- REQ-C7-01: C7 SHALL define a per-component maintainer-doc template enforcing the four §10.6
  artifacts: user-facing usage docs; maintainer/extender docs (code structure, design rationale,
  how to add features); API reference; test guide + contribution guide.
- REQ-C7-02: C7 SHALL ensure maintainer/extender docs exist for every custom-developed component
  enumerated in §10.6 (kopf operator, Headlamp plugins, platform SDK, agent base images,
  callbacks, Coach, custom HolmesGPT toolsets, glue services, Crossplane Compositions).
- REQ-C7-03: Each maintainer doc SHALL include code-structure, design-rationale, and how-to-add-a-
  feature sections (§10.6) so an external extender can extend the component.
- REQ-C7-04: Each component's API reference SHALL state its versioning per ADR 0030 — CRD/XRD API
  versions, SDK semver + compatibility matrix, or URL-path version, as applicable to that component.
- REQ-C7-05: Maintainer docs SHALL be Markdown-in-repo alongside the custom code they document, in
  the portal's reserved §10.6 section (ADR 0008).
- REQ-C7-06: C7 SHALL use only Canon names for the CRDs/XRDs/SDK surfaces it documents
  (`_meta/glossary.md`, `_meta/interface-contract.md`); any maintainer-doc element not in Canon
  SHALL be tagged `[PROPOSED — not in source]`.
- REQ-C7-07: Maintainer docs SHALL be structured so the C8 pipeline can re-index them into
  `platform-knowledge-base` without component-specific preprocessing (§14.3 C8).
- REQ-C7-08: The kopf-operator maintainer doc SHALL document its LiteLLM-subchart packaging /
  lifecycle binding (ADR 0006).
- REQ-C7-09: The agent-base-image maintainer doc SHALL cover both `langgraph` and `deep-agents`
  SDK values (ADR 0019).
- REQ-C7-10: The Crossplane-Composition maintainer doc SHALL document the uniform connection-secret
  contract and one-Composition-per-substrate pattern (ADR 0044), referencing B4 as owner.

## 7. Non-Functional Requirements
- Security: maintainer docs SHALL NOT embed secrets/credentials; secret-handling guidance points
  to the ESO path. Contribution-guide content SHALL reflect the SHA-pinned GitHub Actions posture
  (ADR 0010).
- Multi-tenancy (§6.9): maintainer docs are platform-wide reference, not tenant-scoped; carry no
  tenant data.
- Observability (§6.5): no runtime service; docs reference the components' own observability where
  relevant (e.g. how to add a trace span in the SDK).
- Scale: content scales with the number of custom components; additive.
- Versioning (ADR 0030): each API reference tracks its component's version; major/minor doc
  changes trigger C8 re-index, patch changes do not (§6.4).

## 8. Cross-Cutting Deliverable Checklist
- Helm/manifests: **N/A — C7 is Markdown content; portal build/deploy is C1.**
- Per-product docs (10.5): **N/A — §10.5 per-product docs are Workstream A; C7 is §10.6 maintainer docs.**
- Runbook (10.7): **N/A — runbooks are Workstream A (per-product) and C6 (cross-cutting).**
- Alerts: **N/A — content, no runtime failure conditions.**
- Grafana dashboard (Crossplane XR): **N/A — no per-component runtime metrics.**
- Headlamp plugin: **N/A — C7 documents the Headlamp-plugin custom code (B5/A22); it builds no plugin.**
- OPA/Rego integration: **N/A — content has no admission/runtime policy surface.**
- Audit emission (ADR 0034): **N/A — emits no audit records.**
- Knative trigger flow: **N/A — re-index triggering on flagged commits is owned by C8.**
- HolmesGPT toolset: **N/A — HolmesGPT retrieves maintainer docs via `platform-knowledge-base` (C8); C7 documents custom HolmesGPT toolsets but ships none.**
- 3-layer tests (Chainsaw/Playwright/PyTest): **Partial — PyTest/link-lint for build + template-conformance + Canon-name checks (via C1); Playwright for portal nav/search of the §10.6 section; Chainsaw N/A — no CRD.**
- Tutorials & how-tos: **N/A — C2/C3 own those; C7 is maintainer/extender docs.**

## 9. Acceptance Criteria
- AC-C7-01 (REQ-C7-01): A maintainer-doc template exists with the four §10.6 artifact sections.
- AC-C7-02 (REQ-C7-02): Each custom component enumerated in §10.6 has a maintainer doc present in
  the portal (tracked by a coverage checklist; partial pages allowed pre-implementation per §10).
- AC-C7-03 (REQ-C7-03): A sampled maintainer doc contains code-structure, design-rationale, and
  add-a-feature sections sufficient for an extender to add a feature.
- AC-C7-04 (REQ-C7-04): Each component's API reference states its versioning per ADR 0030 (CRD API
  version / SDK semver / URL-path version, as applicable).
- AC-C7-05 (REQ-C7-05): All maintainer docs render in the portal's §10.6 section with zero build
  errors.
- AC-C7-06 (REQ-C7-06): A Canon-name lint over C7 pages finds no non-Canon CRD/XRD/SDK identifier
  untagged by `[PROPOSED — not in source]`.
- AC-C7-07 (REQ-C7-07): The C8 pipeline ingests maintainer docs into `platform-knowledge-base`
  with no component-specific preprocessing (verified jointly with C8).
- AC-C7-08 (REQ-C7-08): The kopf-operator maintainer doc documents the LiteLLM-subchart packaging.
- AC-C7-09 (REQ-C7-09): The agent-base-image maintainer doc covers `langgraph` and `deep-agents`.
- AC-C7-10 (REQ-C7-10): The Crossplane-Composition maintainer doc documents the uniform
  connection-secret contract and per-substrate Composition pattern.

## 10. Risks & Open Questions
- R1 (med): **Boundary with C4** (Diataxis reference) — C4 owns product/SDK reference; C7 owns
  maintainer-facing internal API reference. Open question: where the platform-SDK reference lives
  vs. the SDK *internals* reference. Reconciliation: C4 = consumer-facing surface, C7 = extender
  internals; cross-link both. Blast radius bounded.
- R2 (med): Maintainer-doc **page template / API-reference layout** is not Canon — `[PROPOSED —
  not in source]`. The four §10.6 artifacts *are* source-stated; the template is a C7 convention
  co-owned with C8's ingestion contract.
- R3 (med): Docs-before-implementation (§10) means many maintainer docs are authored against
  not-yet-built code and may drift; per-component re-review is required when the component lands.
  Open question: drift-detection cadence — partly resolved by major/minor re-index flagging (§6.4).
- R4 (low): SDK method signatures are **not specified in source** (interface-contract §3/§6); C7
  documents named surface groups only and flags signature-level gaps rather than inventing them.

## 11. References
- architecture-overview.md §10.6 Maintainer/extender documentation for custom code (~L1518–1525).
- architecture-overview.md §10 doc-timing + docs-before-implementation (~L1431–1433).
- architecture-overview.md §14.3 Workstream C, C7 + C8 rows (~L1718–1732); §14.2 custom-component
  list (B-series, ~L1700–1716).
- architecture-overview.md §6.4 Knowledge Base (`platform-knowledge-base` RAGStore).
- ADR 0008 (Material for MkDocs), ADR 0006 (kopf operator as LiteLLM subchart), ADR 0019 (LangGraph
  + Deep Agents SDK values), ADR 0030 (versioning policy), ADR 0044 (substrate abstraction /
  connection-secret contract).
- _meta/glossary.md (custom components, SDK Canon names, CRD/XRD kinds).
- _meta/interface-contract.md §1 (CRDs/XRDs), §3 (SDK surfaces), §4 (connection-secret), §6 (gaps).
