# SPEC C6 — Cross-cutting / integrated runbooks  [PROPOSED]

> kind: COMPONENT · workstream: C · tier: T1
> upstream: [C1] · downstream: [C8] · adrs: [0008, 0012, 0034, 0035, 0040] · views: [6.5]
> canon-glossary: see _meta/glossary.md · canon-interface: see _meta/interface-contract.md

## 1. Purpose & Problem Statement

Per-product runbooks live alongside the per-product docs and are owned by the Workstream A
component that owns each product (§10.7, §14.3). But the hardest operational failures are
**cross-cutting**: they span products and cannot be diagnosed or remediated from any single
product's runbook — "agent runs slow end-to-end", "cost spike traced back through virtual keys",
"audit volume drop", "eval failures spiking", "agent stuck mid-run", "tenant-specific failure"
(§10.7). These live as a separate portal section precisely because they cross product boundaries.

C6 authors that cross-cutting runbook section: for each cross-cutting symptom, a runbook giving
the **diagnostic flow → remediation → escalation** path, threading through the gateway (LiteLLM),
the sandbox/agent runtime, the observability stack (Langfuse, Tempo, Mimir), the audit pipeline,
the capability registries, and the promotion fabric. The content is Markdown in the C1 portal,
re-indexed into `platform-knowledge-base` by C8 so HolmesGPT and the Interactive Access Agent can
retrieve it. C6 is content authoring; it ships no code, no CRD, and no runtime service.

## 2. Scope

### 2.1 In scope
- The **cross-cutting runbook section** in the C1 portal (§10.7) — one runbook per cross-cutting
  symptom enumerated in §10.7: agent runs slow end-to-end; cost spike traced back through virtual
  keys; audit volume drop; eval failures spiking; agent stuck mid-run; tenant-specific failure.
- For each runbook: a **diagnostic flow** (which signals/dashboards/traces to inspect, in order),
  a **remediation** path, and an **escalation** path (§10.7).
- Cross-references into per-product runbooks (§10.7, Workstream A) where a cross-cutting flow
  hands off to a single-product remediation.
- Authoring runbooks so they are usable as a **HolmesGPT** retrieval source via
  `platform-knowledge-base` (ADR 0012, ADR 0022) — structured for RAG ingestion by C8.
- Alignment to the observability surfaces that exist: trace correlation by `trace_id` across
  Langfuse + Tempo (ADR 0015), audit volume via the audit pipeline (ADR 0034), `LogLevel`
  raising for deeper diagnosis (ADR 0035).

### 2.2 Out of scope
- **Per-product runbooks** (gateway latency spike, virtual key exhaustion, callback failure,
  ingestion lag, broker backlog, sandbox creation failures, secret sync failures, controller
  errors, archive backlog) — owned by the **Workstream A** component for each product (§10.7).
- **Portal infrastructure / runbook section slot** — **C1** (§14.3); C6 fills the slot.
- **Indexing runbooks into `platform-knowledge-base`** — **C8** (§14.3); C6 emits Markdown only.
- **Maintainer / extender docs** for custom code — **C7** (§10.6).
- **Operator dashboards** the runbooks link to — **D1** (§11.1); C6 references, does not build.
- **Final production runbook compilation / exercise-at-least-once verification** — **F6** (§14.6).
- **Audit retention/redaction policy** referenced by the "audit volume drop" runbook — **F1**
  (ADR 0034 defers it).

## 3. Context & Dependencies

Upstream consumed (HARD): **C1** — the portal navigation skeleton and the reserved §10.7 runbook
section, plus the MkDocs/Markdown authoring conventions C6 writes against.

Continuous (non-blocking) inputs: every product the cross-cutting flows traverse (LiteLLM, ARK,
agent-sandbox, Langfuse, Tempo, Mimir, the audit pipeline, Kargo) continuously stabilizes the
signals C6 references; per-product runbook authors (Workstream A) provide the single-product
hand-off targets. C6 may author partial runbooks early and finalize when the relevant flow's
components land (§10 doc-timing). **F6** later verifies every runbook has been exercised.

Downstream consumers: **C8** (re-indexes the runbook Markdown into `platform-knowledge-base`);
operators and **HolmesGPT** / the **Interactive Access Agent** retrieve them at diagnosis time.

ADR decisions honored:
- **ADR 0008** — Material for MkDocs; runbooks are Markdown-in-repo, re-indexable by C8.
- **ADR 0012 / 0022** — runbooks are retrievable through `platform-knowledge-base` so HolmesGPT
  (and any consuming Platform Agent) can use them; the KB is a separate primitive, not built into
  any agent.
- **ADR 0034** — the "audit volume drop" runbook reasons over the Postgres+S3 system of record and
  the OpenSearch advisory index; it must not treat OpenSearch as authoritative.
- **ADR 0035** — diagnostic flows may instruct raising `LogLevel` / trace granularity per
  component/tenant/event-class to deepen a live diagnosis, then lowering it.
- **ADR 0040** — the runbooks reference Kargo Stage promotion state where a cross-cutting failure
  is promotion-related (e.g. a regression introduced at a Stage).

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
N/A — C6 defines no CRD/XRD. Runbooks **reference** Canon kinds by name (`VirtualKey`,
`BudgetPolicy`, `AgentRun`, `LogLevel`, `Approval`) but own none.

### 4.2 APIs / SDK surfaces
N/A — C6 exposes no API/SDK surface. Its consumable interface is Markdown runbook pages and the
RAG-friendly structure C8 ingests. `[PROPOSED — not in source]` for any specific runbook page
template/front-matter shape — the diagnostic/remediation/escalation *structure* is source-stated
(§10.7), but a concrete page template is a C6 authoring convention (co-owned with C8's ingestion
contract).

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
N/A — C6 emits/consumes no CloudEvents. Runbooks **read** signals derived from the taxonomy
(e.g. `platform.observability.*` threshold crossings, `platform.audit.*` volume, `platform.gateway.*`
routing) as diagnostic inputs, but C6 introduces no event types (those are B12-owned).

### 4.4 Data schemas / connection-secret contracts
N/A — C6 is content and holds no data backend / writes no connection secret. The only contract is
the **Markdown corpus shape** the C8 indexer ingests; that schema is owned by C8.

## 5. OSS-vs-Custom Decision
**Build-new content on top of configured OSS (Material for MkDocs, ADR 0008); no fork.** C6 is
authored Markdown runbooks, not software. It reuses the C1 portal and the C8 indexing path;
rationale (ADR 0008): docs-as-code in-repo, re-indexable into the Knowledge Base. No alternative
tooling decision is owned here.

## 6. Functional Requirements
- REQ-C6-01: C6 SHALL author a cross-cutting runbook for each symptom enumerated in §10.7: agent
  runs slow end-to-end; cost spike traced back through virtual keys; audit volume drop; eval
  failures spiking; agent stuck mid-run; tenant-specific failure.
- REQ-C6-02: Each runbook SHALL contain a **diagnostic flow**, a **remediation** path, and an
  **escalation** path (§10.7).
- REQ-C6-03: Each runbook's diagnostic flow SHALL name the concrete signals/surfaces to inspect
  and the order — e.g. `trace_id`-correlated Langfuse + Tempo traces (ADR 0015), Mimir metrics,
  audit volume from the audit pipeline (ADR 0034), and the relevant operator dashboard (D1).
- REQ-C6-04: Each runbook SHALL live in the C1 portal's reserved cross-cutting runbook section as
  Markdown-in-repo (§10.7, ADR 0008), distinct from per-product runbooks.
- REQ-C6-05: Cross-cutting runbooks SHALL cross-reference the per-product runbook they hand off to
  (Workstream A) rather than duplicate single-product remediation (§10.7).
- REQ-C6-06: Runbooks SHALL be authored in a structure that the C8 pipeline can re-index into
  `platform-knowledge-base` without runbook-specific preprocessing (§14.3 C8).
- REQ-C6-07: The "audit volume drop" runbook SHALL treat Postgres+S3 as the system of record and
  OpenSearch as advisory/rebuildable, per ADR 0034 / ADR 0014.
- REQ-C6-08: Where deeper live diagnosis is needed, a runbook SHALL document raising and then
  restoring `LogLevel` / trace granularity per the dynamic toggle (ADR 0035).
- REQ-C6-09: The "cost spike traced back through virtual keys" runbook SHALL reference `VirtualKey`
  and `BudgetPolicy` by their Canon names and the budget-exceeded observability flow (§6.7).
- REQ-C6-10: Runbooks SHALL be authored to be usable as a HolmesGPT retrieval source via
  `platform-knowledge-base` (ADR 0012 / 0022).

## 7. Non-Functional Requirements
- Security: runbooks SHALL NOT embed secrets, tokens, or tenant-identifying data; remediation
  steps that touch credentials reference the standard ESO/secret path, not literal values.
- Multi-tenancy (§6.9): the "tenant-specific failure" runbook SHALL respect tenant isolation —
  diagnosis steps scope to a tenant's namespaces and `platform_tenants`/`platform_namespaces`
  claims without exposing other tenants' data.
- Observability (§6.5): runbooks are consumers of the observability stack; they SHALL reference
  only existing signals (Langfuse, Tempo, Mimir, audit pipeline) and the D1 dashboards.
- Scale: content scales with the number of cross-cutting symptoms; additive, no runtime cost.
- Versioning (ADR 0030): runbooks track the documented behavior of the components they traverse;
  major/minor doc changes trigger C8 re-index, patch changes do not (§6.4).

## 8. Cross-Cutting Deliverable Checklist
- Helm/manifests: **N/A — C6 is Markdown content; portal build/deploy is C1.**
- Per-product docs (10.5): **N/A — owned by Workstream A product owners.**
- Runbook (10.7): **Applicable — C6 IS the cross-cutting runbook deliverable.**
- Alerts: **N/A — C6 references alert-driven signals but defines no alert rules (those ship with the products / D1).**
- Grafana dashboard (Crossplane XR): **N/A — runbooks link to D1 dashboards; C6 builds none.**
- Headlamp plugin: **N/A — no UI surface; portal-vs-Headlamp linking deferred (ADR 0008, backlog).**
- OPA/Rego integration: **N/A — content has no admission/runtime policy surface.**
- Audit emission (ADR 0034): **N/A — C6 reads audit signals; it emits no audit records.**
- Knative trigger flow: **N/A — re-index triggering on flagged commits is owned by C8.**
- HolmesGPT toolset: **N/A — HolmesGPT consumes runbooks via `platform-knowledge-base` (C8), not a C6 toolset; C6 authors the retrievable content.**
- 3-layer tests (Chainsaw/Playwright/PyTest): **Partial — PyTest/link-lint for build + cross-reference validity (via C1's check); Playwright for portal nav/search of the section; Chainsaw N/A — no CRD.**
- Tutorials & how-tos: **N/A — that is C2/C3; C6 is runbooks.**

## 9. Acceptance Criteria
- AC-C6-01 (REQ-C6-01): A runbook page exists for each of the six §10.7 cross-cutting symptoms.
- AC-C6-02 (REQ-C6-02): Each runbook page contains distinct diagnostic-flow, remediation, and
  escalation sections.
- AC-C6-03 (REQ-C6-03): The "agent runs slow end-to-end" runbook names an ordered set of signals
  including `trace_id`-correlated Langfuse + Tempo inspection.
- AC-C6-04 (REQ-C6-04): All six runbooks render in the portal's cross-cutting runbook section,
  separate from any per-product runbook section, with zero build errors.
- AC-C6-05 (REQ-C6-05): At least one cross-cutting runbook links to a named per-product runbook
  hand-off rather than restating its steps.
- AC-C6-06 (REQ-C6-06): The C8 pipeline ingests the runbook Markdown into `platform-knowledge-base`
  with no runbook-specific preprocessing (verified jointly with C8).
- AC-C6-07 (REQ-C6-07): The "audit volume drop" runbook's diagnostic steps reference Postgres+S3 as
  authoritative and OpenSearch as advisory/rebuildable.
- AC-C6-08 (REQ-C6-08): At least one runbook documents raising `LogLevel` for diagnosis and
  restoring it afterward.
- AC-C6-09 (REQ-C6-09): The "cost spike" runbook references `VirtualKey` and `BudgetPolicy` by
  Canon name and the budget-exceeded flow.
- AC-C6-10 (REQ-C6-10): A retrieval query against `platform-knowledge-base` for a cross-cutting
  symptom returns the corresponding C6 runbook (verified jointly with C8).

## 10. Risks & Open Questions
- R1 (med): The runbook **page template / front-matter** is not Canon — `[PROPOSED — not in
  source]`. The diagnostic/remediation/escalation *structure* is source-stated (§10.7); the
  concrete template is a C6 convention co-owned with C8's ingestion contract. Reconcile jointly.
- R2 (med): Many cross-cutting flows traverse not-yet-landed components; runbooks authored early
  are **partial** (§10 doc-timing). Open question: completeness gate — resolved at **F6** (final
  compilation + exercise-at-least-once verification). Blast radius bounded to doc freshness.
- R3 (low): The "tenant-specific failure" runbook must not leak cross-tenant data in worked
  examples. Reconciliation: use redacted/synthetic tenant identifiers; review against §6.9.
- R4 (low): Signals referenced (specific event-type names under `platform.observability.*` etc.)
  are B12-deferred; runbooks reference the namespace/dashboard, not invented event-type names.

## 11. References
- architecture-overview.md §10.7 Operator runbooks — cross-cutting list (~L1527–1531).
- architecture-overview.md §14.3 Workstream C, C6 + C8 rows (~L1718–1732).
- architecture-overview.md §6.4 Knowledge Base (`platform-knowledge-base` RAGStore) (~L349–368);
  §6.5 Observability (~L370); §6.7 eventing trigger flows (budget-exceeded → email).
- architecture-overview.md §11.1 Operator integrated dashboards (D1 targets) (~L1552).
- ADR 0008 (Material for MkDocs), ADR 0012 (HolmesGPT first-class agent), ADR 0022 (Knowledge Base
  separate primitive), ADR 0014 / 0034 (Postgres+S3 system of record, OpenSearch advisory),
  ADR 0015 (Tempo+Langfuse trace_id correlation), ADR 0035 (dynamic LogLevel toggle), ADR 0040 (Kargo).
- _meta/glossary.md (Knowledge Base, `platform-knowledge-base`, HolmesGPT, `VirtualKey`,
  `BudgetPolicy`, `LogLevel`, `AgentRun`).
- _meta/interface-contract.md §2 (CloudEvent taxonomy), §5 (audit adapter interface).
