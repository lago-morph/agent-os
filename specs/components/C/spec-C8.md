# SPEC C8 — Knowledge Base RAG indexing pipeline  [PROPOSED]

> kind: COMPONENT · workstream: C · tier: T1
> upstream: [C1, C6, C7] · downstream: [F3, F6] · adrs: [0022, 0024, 0014, 0009, 0008, 0013, 0030, 0031, 0034, 0044] · views: [6.4]
> canon-glossary: see _meta/glossary.md · canon-interface: see _meta/interface-contract.md

## 1. Purpose & Problem Statement

The Knowledge Base is a first-class `RAGStore` named `platform-knowledge-base` (§6.4, ADR 0022) —
a shared primitive any Platform Agent may include in its CapabilitySet, reached through the SDK
`rag.*` API on the LiteLLM-mediated call path. Its content must come from somewhere: for **our
authored docs and runbooks** (the MkDocs portal, per-product docs, runbooks), indexing is C8's job
(§14.3 C8). The contract is human-judgment-driven: a contributor flags a commit as a **major or
minor** release change to trigger re-indexing; **patch-level changes do not** trigger it (§6.4),
deliberately avoiding indexing churn from typos and clarifications.

C8 is the **indexing pipeline**: it takes the authored-doc Markdown corpus (from C1, plus C6/C7
content), and on a major/minor-flagged commit, (re-)indexes it into the `platform-knowledge-base`
`RAGStore`, whose backing index is OpenSearch (ADR 0009). Per ADR 0014, the **Git corpus is the
primary/source of truth** and the OpenSearch index is a derived, rebuildable artifact. C8 owns
**only** the authored-doc side: **vendor-doc acquisition and indexing is a separate companion
project** (ADR 0024) that writes into the same store using the same conventions; C8 does not crawl,
pin, or index vendor docs. Because C8 defines the trigger contract and the indexing interface that
both authored docs and the companion project conform to, the trigger and interface contracts must
be specified carefully.

## 2. Scope

### 2.1 In scope
- The **indexing pipeline** that ingests the **authored** Markdown corpus (C1 portal output;
  Diataxis content C2–C5; cross-cutting runbooks C6; maintainer docs C7; per-product docs/runbooks
  from Workstream A) into the `platform-knowledge-base` `RAGStore` (§6.4, §14.3 C8).
- The **trigger contract**: re-indexing fires on a contributor flagging a commit as a **major or
  minor** release change; **patch-level commits do not trigger** (§6.4). The flag is
  human-judgment-driven by design.
- The **(re-)index operation** against the `platform-knowledge-base` `RAGStore` backed by
  OpenSearch (ADR 0009), honoring the primary-source-of-truth invariant: the Git corpus is
  authoritative; the OpenSearch index is derived and **fully rebuildable** from it (ADR 0014).
- The **shared indexing conventions** (chunking/metadata/store mapping) the authored side uses,
  which the **vendor-doc companion project conforms to** when writing into the same store
  (ADR 0024) — C8 owns the conventions for the authored side and publishes them as the contract.
- Wiring the indexing pipeline into the GitHub Actions docs path (ADR 0010 / 0033) and/or a
  Knative-triggered flow (ADR 0031) as the major/minor-flag delivery mechanism — selection is a
  design-time choice flagged below.
- Audit emission of index runs via the audit adapter (ADR 0034) and trigger/result events under
  the Canon CloudEvent taxonomy (ADR 0031).

### 2.2 Out of scope
- **Vendor documentation acquisition and indexing** — a **separate companion project** (ADR 0024);
  it scans vendor versions and triggers its own re-indexing on vendor major/minor releases, writing
  into `platform-knowledge-base` using C8's conventions. Integration-point coordination is **F3**.
- **The `RAGStore` CRD definition / reconciliation** — `RAGStore` is reconciled by the **B13** kopf
  operator into LiteLLM (ADR 0013); C8 indexes content into the store, it does not define/reconcile
  the CRD.
- **The OpenSearch install** — **A11** (ADR 0009). C8 consumes it as the index backend.
- **The `rag.*` consumption API** — owned by the **Platform SDK (B6)**; C8 populates what `rag.*`
  reads, it does not implement the read API.
- **Portal infrastructure / Markdown output** — **C1**. **Content authoring** — C2–C7 / Workstream A.
- **Agent-pod access pattern (SDK API vs filesystem mount)** — design-time per agent, deferred
  (ADR 0022, backlog §1.7); not C8.
- **Audit retention/redaction** — **F1** (ADR 0034 defers it).
- **Final exercise/verification of the pipeline** — **F6**; companion-project handoff — **F3**.

## 3. Context & Dependencies

Upstream consumed (HARD): **C1** (the re-indexable Markdown corpus output + conventions);
**C6** and **C7** (cross-cutting runbooks and maintainer docs — co-own the corpus shape so
re-indexing is preprocessing-free). Soft/foundation: **A11** OpenSearch (the index backend, ADR
0009) and **B13** (`RAGStore` reconciled into LiteLLM, ADR 0013) — faked until they land.

Continuous (non-blocking) inputs: C2–C5 Diataxis content and Workstream A per-product docs/runbooks
continuously enlarge the authored corpus; B12 schema registry for any CloudEvent types C8 emits;
B14 test framework; B22 threat-model standards.

Downstream consumers: **F3** (vendor-doc companion-project handoff — verifies the shared-conventions
integration point); **F6** (final pipeline exercise/verification); and indirectly **every Platform
Agent** that includes `platform-knowledge-base` in its CapabilitySet (HolmesGPT, Interactive Access
Agent, Coach) reading via `rag.*`.

ADR decisions honored:
- **ADR 0022** — `platform-knowledge-base` is a first-class `RAGStore`, separate primitive; C8
  populates it for authored docs; consumption is via `rag.*` on the LiteLLM-mediated path.
- **ADR 0024** — vendor-doc acquisition/indexing is a separate companion project; C8 owns only the
  authored side and publishes the shared indexing conventions the companion project conforms to.
- **ADR 0014** — Git corpus is the primary/source of truth; the OpenSearch index is derived and
  fully rebuildable; re-indexing on major/minor is the rebuild path. C8 never treats the index as
  authoritative.
- **ADR 0009** — OpenSearch is the search/vector backend for the index.
- **ADR 0008** — Material for MkDocs / Markdown-in-repo is the authored-doc source surface.
- **ADR 0013** — `RAGStore` (and its `indexes[]`, `contentSourceRefs[]`, `ingestionPipelineRef`)
  is reconciled by B13; C8's pipeline is referenced as the `ingestionPipelineRef` target.
- **ADR 0030** — major/minor/patch semantics align with the platform versioning policy; the
  pipeline's own HTTP/trigger surface (if any) is versioned.
- **ADR 0031** — any trigger/result events fall under exactly one `platform.*` namespace.
- **ADR 0034** — index runs emit audit via the audit adapter, not directly to a store.
- **ADR 0044** — if the pipeline needs object storage / a search-index backend, it consumes them
  via the substrate XRDs (`ObjectStore`, `SearchIndex`) and the uniform connection-secret shape.

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
C8 defines **no** CRD/XRD. It **binds to** the Canon `RAGStore` CRD (owner: B13 kopf operator —
`_meta/interface-contract.md` §1.4), specifically:
- `RAGStore.backend` — OpenSearch (ADR 0009).
- `RAGStore.indexes[]` — the index(es) C8 populates for `platform-knowledge-base`.
- `RAGStore.contentSourceRefs[]` — references the authored-doc corpus source(s).
- `RAGStore.ingestionPipelineRef` — **points at the C8 indexing pipeline** as the ingestion
  pipeline for authored docs.

The named `platform-knowledge-base` `RAGStore` instance is Canon (§6.4, glossary). C8 owns no
fields beyond what B13's CRD provides; it consumes them by Canon name.

### 4.2 APIs / SDK surfaces
- **Consumption side (not owned):** the SDK `rag.*` API (Platform SDK, B6) is how agents read the
  store; C8 populates what `rag.*` returns and does not implement it.
- **Indexing interface (C8-owned, `[PROPOSED — not in source]` for the concrete shape):** the
  operation "(re-)index the authored corpus into `platform-knowledge-base`". Source states the
  *trigger* (major/minor flag) and the *store* and *source-of-truth* invariant, but **not** a
  concrete pipeline API signature, chunking strategy, or metadata schema — these are
  `[PROPOSED — not in source]` and co-owned with C1/C6/C7 (corpus shape) and with the companion
  project (shared conventions). If exposed as a custom HTTP API it follows URL-path versioning
  (ADR 0030 §3.3).
- **Shared-conventions contract (C8-owned):** the chunking + metadata + store-mapping conventions
  that BOTH the authored side and the vendor-doc companion project use to write into the same
  store (ADR 0024). `[PROPOSED — not in source]` for the concrete field set.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
- **Consumed (trigger):** the major/minor-flag signal. Concrete delivery mechanism is a design-time
  choice — either a GitHub Actions docs-pipeline step (ADR 0010/0033) or a Knative-triggered flow
  reading a commit flag. If event-driven, the trigger event falls under exactly one Canon namespace;
  the closest-fit is `platform.capability.*` (capability-registry/RAGStore content change) — but the
  **specific event-type name and schema are B12-deferred** (interface-contract §2/§6). Tagged
  `[PROPOSED — not in source]`; do not invent an event-type name here.
- **Emitted (result):** index-run started/completed/failed. Closest-fit namespace
  `platform.capability.*` (or `platform.observability.*` for lag/threshold signals such as
  ingestion lag); per-event-type names are B12-owned. `[PROPOSED — not in source]` for names.
- C8 introduces **no new top-level namespace** (that would be a breaking ADR change, ADR 0031).

### 4.4 Data schemas / connection-secret contracts
- The **OpenSearch index** backing `platform-knowledge-base` is the derived artifact; the **Git
  Markdown corpus is the primary** (ADR 0014). Every index entry MUST be rebuildable from the
  corpus; the re-index-on-major/minor path IS the documented rebuild path.
- If C8 stages corpus snapshots in object storage or uses a dedicated search index, it consumes
  them via `ObjectStore` / `SearchIndex` with the uniform connection-secret shape (`host`,
  `port`, `user`, `password`, `dbname` or equivalent; ADR 0044). C8 defines no new secret shape.
- The chunk/document metadata schema written into the index is `[PROPOSED — not in source]`
  (co-owned with the shared-conventions contract, §4.2).

## 5. OSS-vs-Custom Decision
**Build-new glue (custom) on top of OSS backends — no fork.** The index backend is OpenSearch
(ADR 0009, reused as-is); the source surface is Material for MkDocs Markdown (ADR 0008); the store
is the B13-reconciled `RAGStore`. C8 builds the **custom ingestion pipeline** (chunk → embed/index
→ write to `platform-knowledge-base`) and the **major/minor trigger wiring**, because no OSS
component provides the platform-specific "human-flagged major/minor re-index of our authored
corpus into a named RAGStore with a primary-source rebuild guarantee" behavior. Rationale:
ADR 0022 (named primitive), ADR 0024 (authored-only scope), ADR 0014 (rebuildable-from-primary).

## 6. Functional Requirements
- REQ-C8-01: C8 SHALL (re-)index the authored Markdown corpus (C1 portal output incl. C2–C7 and
  Workstream A docs/runbooks) into the `platform-knowledge-base` `RAGStore` (§6.4, §14.3 C8).
- REQ-C8-02: Re-indexing SHALL be triggered when a contributor flags a commit as a **major or
  minor** release change (§6.4).
- REQ-C8-03: **Patch-level** commits SHALL NOT trigger re-indexing (§6.4).
- REQ-C8-04: The OpenSearch-backed index SHALL be **fully rebuildable from the Git corpus**; the
  Git corpus is the primary/source of truth and the index is derived (ADR 0014).
- REQ-C8-05: C8 SHALL index content into the `RAGStore` whose `backend` is OpenSearch (ADR 0009),
  binding via the B13-reconciled `RAGStore` fields (`indexes[]`, `contentSourceRefs[]`,
  `ingestionPipelineRef`), inventing no new CRD/field.
- REQ-C8-06: C8 SHALL publish the **shared indexing conventions** (chunking/metadata/store mapping)
  for the authored side; the conventions SHALL be the contract the **vendor-doc companion project**
  conforms to when writing into the same store (ADR 0024). `[PROPOSED — not in source]` shape.
- REQ-C8-07: C8 SHALL NOT acquire, pin, crawl, or index **vendor** documentation; that is the
  separate companion project's responsibility (ADR 0024). Coordination is F3.
- REQ-C8-08: Each index run SHALL emit an audit record via the audit adapter (ADR 0034) and a
  result event under exactly one Canon `platform.*` CloudEvent namespace (ADR 0031), without
  introducing a new top-level namespace. Audit emission is gated on the audit-adapter
  freeze-gate (D-05): C8 emits no audit events until the adapter interface and `audit_events`
  schema are frozen.
- REQ-C8-09: The trigger SHALL be delivered through the v1.0 mechanism (GitHub Actions docs path,
  ADR 0010/0033, and/or a Knative-triggered flow, ADR 0031); the chosen mechanism is documented.
- REQ-C8-10: Re-indexing SHALL be **idempotent** — re-running the same major/minor-flagged corpus
  state SHALL converge the index to the same content without duplicate entries (derived-artifact
  consistency, ADR 0014).
- REQ-C8-11: A full **rebuild-from-primary** path SHALL exist and be runnable on demand
  (independent of the major/minor trigger) per the OpenSearch-rebuildable invariant (ADR 0014).
- REQ-C8-12: Any object-storage / search-index backing the pipeline SHALL be consumed via the
  substrate XRDs and uniform connection-secret shape (ADR 0044), defining no new secret schema.

## 7. Non-Functional Requirements
- Security: index runs SHALL NOT embed or log secrets; backend access uses the ESO/connection-secret
  path. Trigger wiring uses the SHA-pinned GitHub Actions posture (ADR 0010). Indexed content is
  platform documentation, not tenant data.
- Multi-tenancy (§6.9): `platform-knowledge-base` is a single platform-wide store of platform docs;
  access is governed by CapabilitySet inclusion + the LiteLLM-mediated `rag.*` path (ADR 0022), not
  by C8. C8 indexes no tenant-scoped content.
- Observability (§6.5): index-run start/complete/fail and **ingestion lag** SHALL be observable
  (audit + a result/observability event), so the "ingestion lag" runbook (§10.7) has signals.
- Scale: indexing cost scales with authored-corpus size and major/minor-flag frequency; patch
  commits are deliberately excluded to bound churn (§6.4). Idempotent re-index avoids unbounded
  growth.
- Versioning (ADR 0030): major/minor/patch semantics align with the platform versioning policy; any
  C8 HTTP/trigger surface is URL-path-versioned.

## 8. Cross-Cutting Deliverable Checklist
- Helm/manifests: **Applicable — the indexing pipeline's deploy/trigger wiring (GitOps).**
- Per-product docs (10.5): **N/A — Workstream A owns per-product docs; C8 indexes them.**
- Runbook (10.7): **Applicable — an "ingestion lag" runbook for the pipeline (also referenced by C6/Workstream A); C8 supplies the pipeline-specific runbook content.**
- Alerts: **Applicable — ingestion-lag / index-run-failure alert signals.**
- Grafana dashboard (Crossplane XR): **Applicable (light) — KB RAG indexing health feeds the "Knowledge base RAG effectiveness" developer dashboard (D2, §11.2); C8 emits the metrics, D2 builds the dashboard.**
- Headlamp plugin: **N/A — no dedicated UI; store visibility is via standard RAGStore/capability surfaces.**
- OPA/Rego integration: **N/A — C8 writes index content; access policy is on the `rag.*` consumption path (LiteLLM/OPA), not the indexer.**
- Audit emission (ADR 0034): **Applicable — index runs emit audit via the adapter.**
- Knative trigger flow: **Applicable (design-time) — the major/minor-flag re-index may be a Knative-triggered flow (ADR 0031); alternative is the GitHub Actions docs path (ADR 0010).**
- HolmesGPT toolset: **N/A — HolmesGPT consumes the populated store via `platform-knowledge-base`/`rag.*`, not a C8 toolset.**
- 3-layer tests (Chainsaw/Playwright/PyTest): **Applicable — PyTest for pipeline logic (chunking, idempotency, rebuild, patch-no-trigger); Chainsaw for the `RAGStore` binding/trigger reconcile path; Playwright N/A unless a portal-side flag UI exists.**
- Tutorials & how-tos: **Partial — supports the §10.1 "Use the knowledge base RAG from a custom agent" tutorial and §10.8 "how docs get indexed" docs-on-docs (C9); C8 supplies the indexing behavior they document.**

## 9. Acceptance Criteria
- AC-C8-01 (REQ-C8-01): A major/minor-flagged commit to the authored corpus results in that
  content being queryable from `platform-knowledge-base` via `rag.*`.
- AC-C8-02 (REQ-C8-02): A commit flagged major or minor triggers an index run.
- AC-C8-03 (REQ-C8-03): A commit flagged patch (or unflagged) does NOT trigger an index run.
- AC-C8-04 (REQ-C8-04 / REQ-C8-11): Deleting and rebuilding the OpenSearch index from the Git
  corpus reproduces the same queryable content (rebuild-from-primary verified).
- AC-C8-05 (REQ-C8-05): The pipeline writes into the `RAGStore` named `platform-knowledge-base`
  with `backend` OpenSearch, via the B13 `RAGStore` fields, defining no new CRD/field.
- AC-C8-06 (REQ-C8-06): The published shared indexing conventions exist and a sample document
  indexed by the authored path and one written per the conventions land in a consistent shape
  (verified jointly with F3 against the companion-project contract).
- AC-C8-07 (REQ-C8-07): No C8 code path acquires/crawls/indexes vendor docs (negative test).
- AC-C8-08 (REQ-C8-08): An index run produces an audit record (via the adapter) and a result event
  under exactly one Canon `platform.*` namespace.
- AC-C8-09 (REQ-C8-10): Re-running the same flagged corpus state yields no duplicate index entries.
- AC-C8-10 (REQ-C8-09): The chosen trigger mechanism (GitHub Actions and/or Knative) fires the
  re-index on a major/minor flag end-to-end.
- AC-C8-11 (REQ-C8-12): Any object-store/search-index used is bound via the substrate XRD
  connection-secret shape, with no bespoke secret schema.

## 10. Risks & Open Questions
- R1 (high): The **concrete indexing interface** — chunking strategy, metadata schema, pipeline API
  shape — is `[PROPOSED — not in source]`. It is load-bearing because the vendor-doc companion
  project (ADR 0024) must conform to it. Reconciliation: C8 publishes the conventions as the
  contract; F3 validates the integration point; co-own the corpus shape with C1/C6/C7.
- R2 (high): The **trigger delivery mechanism** is a design-time choice (GitHub Actions docs path
  vs Knative-triggered flow). The major/minor *flag semantics* are source-stated (§6.4); the
  *transport* and any CloudEvent **event-type name** are B12-deferred (`[PROPOSED — not in
  source]`). Open question: where the flag lives on the commit (label, trailer, path convention).
- R3 (med): The CloudEvent **namespace** for trigger/result events — closest fit
  `platform.capability.*` (RAGStore content change) / `platform.observability.*` (lag) — but
  per-event-type names/schemas are B12-owned (interface-contract §2/§6). C8 must not mint names.
- R4 (med): **Companion-project drift** (ADR 0024) — until it lands, KB has authored content only;
  vendor coverage is absent (acceptable for v1.0). Conventions drift is owned at F3.
- R5 (low): Upstream `RAGStore` (B13) and OpenSearch (A11) may land after C8 authoring; build and
  test against fakes (a stub RAGStore + a local OpenSearch fixture) until they land, then re-run.
- R6 (low): Agent-pod access pattern (SDK vs filesystem mount) is deferred (ADR 0022, backlog §1.7)
  — does not block C8; C8 populates the store regardless of read pattern.

## 11. References
- architecture-overview.md §6.4 Knowledge Base as a separate primitive — content sources,
  re-indexing rules (major/minor flag; patch excluded), access patterns (~L349–368).
- architecture-overview.md §14.3 Workstream C, C8 row (vendor-doc companion project carve-out)
  (~L1731); §14.6 F3 (companion handoff) + F6 (~L1757–1763).
- architecture-overview.md §10 (MkDocs portal indexed as `platform-knowledge-base` RAGStore,
  ~L1429); §10.7 ingestion-lag runbook; §11.2 D2 KB RAG effectiveness dashboard (~L1573).
- ADR 0022 (Knowledge Base separate primitive), ADR 0024 (vendor doc separate companion project),
  ADR 0014 (Postgres/Git primary; OpenSearch derived/rebuildable), ADR 0009 (OpenSearch),
  ADR 0008 (Material for MkDocs), ADR 0013 (`RAGStore` CRD via B13), ADR 0030 (versioning),
  ADR 0031 (CloudEvent taxonomy), ADR 0034 (audit pipeline/adapter), ADR 0044 (substrate XRDs).
- _meta/glossary.md (Knowledge Base, `platform-knowledge-base`, `RAGStore`, OpenSearch).
- _meta/interface-contract.md §1.4 (`RAGStore` fields, B13 owner), §2 (CloudEvent taxonomy + B12
  deferral), §4 (connection-secret), §6 (gaps left to component design).
