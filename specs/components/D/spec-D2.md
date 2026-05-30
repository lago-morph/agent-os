# SPEC D2 — Developer dashboards

> kind: COMPONENT · workstream: D · tier: T1
> upstream: [A1, A2, A5, A7, A10, A11, A13, B4, B6, B10, B12] · downstream: [] · adrs: [0021, 0015, 0018, 0009, 0014, 0011, 0030] · views: [6.5]
> canon-glossary: <hash> · canon-interface: <hash>

## 1. Purpose & Problem Statement

Developers building Platform Agents need feedback views that operators do not — prompt-version performance, A/B comparisons, evaluation trends, cost-per-success economics, failure-mode clustering, and the effectiveness of the tools, memory, skills, and Knowledge Base RAG an agent uses. Per-component vendor dashboards and the integrated operator views (D1) are framed around platform health, not agent-iteration feedback. This component (D2) owns the high-leverage **developer-facing dashboards** described in §11.2.

Each dashboard is delivered as a namespaced `GrafanaDashboard` Crossplane XR (ADR 0021), composed by Component B4's Composition and reconciled into Grafana via the same declarative GitOps path as every other platform resource. Visibility is governed by Kubernetes RBAC and OPA per the RBAC-as-floor / OPA-as-restrictor model (ADR 0018), so a developer sees their own tenant/agent data. Dashboards deep-link into Langfuse traces, Headlamp resource views, and Git commits (§11.2). `[PROPOSED]` tags mark any panel data source, metric name, or index field not fixed by Canon.

## 2. Scope

### 2.1 In scope
- The nine §11.2 developer dashboards: Prompt performance; A/B test comparison; Eval trend; Cost per success; Failure mode explorer; Tool usage heatmap; Memory effectiveness; Skill performance; Knowledge Base RAG effectiveness.
- Authoring each as a `GrafanaDashboard` XR manifest (`dashboardJson`, `folder`, `visibility`) submitted via GitOps.
- Deep-link wiring into Langfuse traces, Headlamp resource views, and Git commits (§11.2).
- Panel query definitions against the platform metric/trace/eval/index backends (Mimir, Tempo, Langfuse, OpenSearch advisory index).

### 2.2 Out of scope (and where it lives instead)
- The `GrafanaDashboard` XRD, its Composition, and the Grafana provider reconciliation path — **Component B4** (§11, ADR 0021).
- Per-component dashboards — each **Workstream A component** (§14.1, §11).
- Operator integrated / cross-cutting dashboards — **Component D1** (§11.1).
- Test-framework pass/fail + flake dashboards — **Component D3** (§14.4); D2's Eval trend covers evaluation runs, not the Chainsaw/Playwright/PyTest test layers.
- The evaluation engine, promptfoo runs, and A/B execution mechanics — owned by ARK `Evaluation` (A5) and the agent/eval tooling; D2 only visualizes their outputs (ADR 0011).
- Alert rules — owned per-component (§14.1); D2 visualizes firing/regression state only.
- Dashboards a Platform Agent publishes about itself (e.g. Coach-published "agent X performance trend") — those ship as the agent's own `GrafanaDashboard` XR (§11); D2 provides the reusable developer-facing templates.

## 3. Context & Dependencies

Upstream consumed:
- **B4** — the `GrafanaDashboard` XRD + Composition + Grafana provider path; D2 manifests are claims against it.
- **A2 (Langfuse)** — prompt/version performance, eval scores, cost, and trace deep-link target; primary data source for prompt performance, A/B, eval trend, cost-per-success, failure-mode dashboards.
- **A13 (Tempo + Mimir)** — metrics (Mimir) and traces (Tempo) backends queried by panels; latency/token/cost series.
- **A1 (LiteLLM)** — gateway token/cost/model-routing metrics for cost-per-success and prompt performance.
- **A5 (ARK)** — `Evaluation` / `AgentRun` outcomes feeding eval-trend, A/B, and failure-mode clustering.
- **A10 (Letta)** — memory read/write/recall signals for the Memory effectiveness dashboard.
- **A11 (OpenSearch)** — RAG retrieval + advisory eval-result index for KB RAG effectiveness and eval-trend panels (advisory only per ADR 0009/0014).
- **A7 (OPA)** — policy-decision signals where they bear on failure modes (e.g. denied tool calls).
- **B6 (Platform SDK)** — `rag.*`, `memory.*`, and OTel emission surfaces produce the telemetry the tool/memory/skill/RAG dashboards aggregate.
- **B10 (Coach Component)** — consumes/overlaps these views for self-improvement; D2 templates are the basis for Coach-published agent dashboards.
- **B12 (CloudEvent schema registry)** — defines `platform.evaluation.*` and related event shapes panels aggregate.

ADRs honored:
- **ADR 0021** — dashboards are namespaced `GrafanaDashboard` XRs; visibility RBAC + OPA controlled.
- **ADR 0018** — RBAC-as-floor / OPA-as-restrictor governs dashboard visibility (developer sees own data).
- **ADR 0015** — Tempo + Langfuse correlated by `trace_id`; failure-mode and prompt-performance deep-links rely on this correlation.
- **ADR 0009 / ADR 0014** — OpenSearch is retrieval-optimization / advisory only; Postgres (and Langfuse for eval) are authoritative — RAG-effectiveness and eval panels must treat OpenSearch as non-authoritative.
- **ADR 0011** — promptfoo is an evaluation tool, not a test layer; eval-trend and A/B dashboards visualize evaluation output, distinct from D3's test-layer dashboards.
- **ADR 0030** — each XR tracks the B4-owned XRD API version.

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
- `GrafanaDashboard` (XR, namespaced; XR form `XGrafanaDashboard`) — fields used: `dashboardJson`, `folder`, `visibility` (RBAC + OPA-controlled). One XR per dashboard. Version tracks the XRD owned by B4 (per ADR 0030/0041 conversion-webhook policy).
- `Evaluation` (ARK CRD, A5) — read-only reference: D2 visualizes outcomes of `Evaluation` runs (`agentRef`, `datasetRef`, `evaluators[]`); D2 does not define or reconcile it.

### 4.2 APIs / SDK surfaces
- Grafana data-source query APIs against **Mimir** (PromQL), **Tempo** (trace queries), **OpenSearch** (advisory eval/RAG index), and **Langfuse** (prompt/eval/cost + deep-link target). Concrete PromQL metric names, Langfuse score/field names, and OpenSearch index fields are **[PROPOSED — not in source]**; source names the dashboards and their intent, not metric identifiers.
- **B6 Platform SDK** `rag.*` / `memory.*` / OTel emission produce the underlying telemetry; specific method signatures are not specified in source (interface-contract §3.1), so panel queries reference **[PROPOSED — not in source]** metric/attribute names.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
- Emitted: none. D2 is a visualization-only consumer.
- Consumed (indirectly, as aggregated backend data): `platform.evaluation.*` (eval run started/completed, A/B results, red-team findings — eval trend, A/B, failure mode), `platform.gateway.*` (token/cost/routing — prompt performance, cost per success), `platform.capability.*` (capability use bearing on tool/skill effectiveness), `platform.lifecycle.*` (`AgentRun` outcomes for cost-per-success and failure clustering). Per-event-type names are deferred to B12.

### 4.4 Data schemas / connection-secret contracts
- N/A — D2 provisions no substrate primitives and writes no connection secrets; it reads existing Grafana data sources provisioned by A13 / A11 / A2. Per-primitive field lists not applicable.

## 5. OSS-vs-Custom Decision

Upstream project: **Grafana** (dashboards) reconciled via the Crossplane Grafana provider path owned by B4. Decision: **config/author**, not build-new — D2 authors dashboard JSON and `GrafanaDashboard` XR manifests, reusing Langfuse/Grafana panels where they exist and composing developer-facing views on top. Rationale: §11 states the high-leverage work is integrated and developer-facing views authored as Compositions; the reconciliation engine is B4 and the eval/trace data come from A2/A5. ADR linkage: ADR 0021, ADR 0011.

## 6. Functional Requirements

- REQ-D2-01: Each of the nine §11.2 dashboards SHALL be delivered as a distinct namespaced `GrafanaDashboard` XR with `dashboardJson`, `folder`, and `visibility` set.
- REQ-D2-02: The Prompt performance dashboard SHALL present latency, tokens, cost, and eval scores broken down by prompt version.
- REQ-D2-03: The A/B test comparison dashboard SHALL render side-by-side prompt or agent versions with a statistical-significance indicator.
- REQ-D2-04: The Eval trend dashboard SHALL show rolling pass rate, failure clustering, and regression alert state across evaluation runs (ADR 0011 — evaluation, not test layers).
- REQ-D2-05: The Cost per success dashboard SHALL compute and display cost normalized by successful outcome, attributable per agent/model/prompt version.
- REQ-D2-06: The Failure mode explorer dashboard SHALL present clusters of failures, each deep-linking to the corresponding Langfuse trace(s) (ADR 0015).
- REQ-D2-07: The Tool usage heatmap dashboard SHALL visualize tool-invocation frequency/effectiveness across tools and agents.
- REQ-D2-08: The Memory effectiveness dashboard SHALL surface memory recall/usage signals from the Letta backend (A10).
- REQ-D2-09: The Skill performance dashboard SHALL surface per-skill usage and effectiveness signals.
- REQ-D2-10: The Knowledge Base RAG effectiveness dashboard SHALL present retrieval success, latency, and hit rate by query type against the `platform-knowledge-base` RAGStore, treating the OpenSearch index as advisory (ADR 0009/0014).
- REQ-D2-11: Dashboards SHALL deep-link into Langfuse traces, Headlamp resource views, and Git commits (§11.2).
- REQ-D2-12: Dashboard `visibility` SHALL be enforced via Kubernetes RBAC + OPA so a developer sees only their tenant's/agent's data unless explicitly shared (ADR 0018, ADR 0021).
- REQ-D2-13: D2 dashboard JSON SHALL be reusable as templates for Platform-Agent-published dashboards (e.g. Coach "agent X performance trend") via standard Crossplane composition selectors (§11).

## 7. Non-Functional Requirements

- Security / multi-tenancy (§6.9): visibility is RBAC-floored and OPA-restricted; no dashboard exposes cross-tenant prompt/eval/cost data absent explicit share (ADR 0018).
- Observability (§6.5): dashboards consume Mimir/Tempo/Langfuse/OpenSearch and add no parallel telemetry stack; eval/RAG/test data share production pipelines (ADR 0011 reporting stance).
- Scale: panel queries SHALL bound time-range and cardinality (per prompt version / per agent) to remain responsive; [PROPOSED — not in source] specific cardinality budgets.
- Versioning (ADR 0030): each XR tracks the B4-owned XRD API version; dashboard JSON revisions flow through GitOps with normal review.
- Reliability: with OpenSearch (advisory) unreachable, RAG-effectiveness and eval panels degrade to "data unavailable" rather than block the rest of the dashboard (ADR 0009/0014 advisory stance).
- Data authority: eval/cost figures cite their authoritative source (Langfuse / Postgres) and label OpenSearch-sourced panels as advisory.

## 8. Cross-Cutting Deliverable Checklist

- Helm/manifests: Applicable — `GrafanaDashboard` XR manifests in Git (GitOps).
- Per-product docs (10.5): N/A — D2 is not a Workstream A product; developer usage docs live in Workstream C / E (§12.2).
- Runbook (10.7): N/A — operator runbooks owned by Workstream C; D2 is a developer-facing surface, not an operated product.
- Alerts: N/A — regression/alert rules owned per-component (§14.1); D2 visualizes regression/firing state only.
- Grafana dashboard (Crossplane XR): Applicable — this is the core deliverable.
- Headlamp plugin: N/A — D2 deep-links into native Headlamp resource views; it authors no plugin.
- OPA/Rego integration: Applicable — dashboard `visibility` enforced via OPA (ADR 0018/0021).
- Audit emission (ADR 0034): N/A — read-only visualization emits no audit-relevant actions of its own.
- Knative trigger flow: N/A — D2 emits/consumes no CloudEvents directly (it reads aggregated backends).
- HolmesGPT toolset: N/A — toolset contributions are per-component (Workstream A); D2 ships no metrics/log tools.
- 3-layer tests (Chainsaw/Playwright/PyTest): Applicable — Chainsaw asserts XR reconciliation into Grafana; Playwright asserts dashboard render, deep-links, and visibility gating; PyTest N/A unless cost-per-success / significance query-helpers are added.
- Tutorials & how-tos: N/A — developer training/how-tos owned by Workstream E/C (§12.2).

## 9. Acceptance Criteria

- AC-D2-01: Applying the D2 manifests creates nine `GrafanaDashboard` XRs that reconcile to nine visible Grafana dashboards in the correct folders. (REQ-D2-01)
- AC-D2-02: Prompt performance shows latency, tokens, cost, and eval-score panels selectable by prompt version. (REQ-D2-02)
- AC-D2-03: A/B comparison renders two versions side-by-side with a significance indicator. (REQ-D2-03)
- AC-D2-04: Eval trend shows a rolling pass-rate line, a failure-cluster view, and a regression indicator over ≥2 eval runs. (REQ-D2-04)
- AC-D2-05: Cost per success renders cost-normalized-by-success attributable per agent/model/prompt version. (REQ-D2-05)
- AC-D2-06: Failure mode explorer groups failures into clusters, each row deep-linking to a Langfuse trace. (REQ-D2-06)
- AC-D2-07: Tool usage heatmap renders a tool-by-agent intensity grid. (REQ-D2-07)
- AC-D2-08: Memory effectiveness renders recall/usage panels sourced from Letta. (REQ-D2-08)
- AC-D2-09: Skill performance renders per-skill usage/effectiveness panels. (REQ-D2-09)
- AC-D2-10: KB RAG effectiveness renders retrieval-success, latency, and hit-rate-by-query-type panels and labels OpenSearch-sourced panels as advisory. (REQ-D2-10)
- AC-D2-11: Each dashboard exposes working deep-links to Langfuse traces, Headlamp resource views, and Git commits. (REQ-D2-11)
- AC-D2-12: A developer without RBAC/OPA grant for another tenant cannot load that tenant's D2 dashboards. (REQ-D2-12)
- AC-D2-13: A D2 dashboard JSON can be instantiated as an agent-scoped XR via a composition selector without modification. (REQ-D2-13)

## 10. Risks & Open Questions

- RISK (med): Concrete metric/eval-score/index field names are not in source; dashboards depend on per-component metric naming and B6 SDK emission contracts landing first. Reconciliation: panel queries authored against [PROPOSED — not in source] names, finalized as A-components and B6 publish contracts. [PROPOSED]
- RISK (med): "Cost per success" requires a defined notion of "success" per agent run; source does not fix this. Proposed: derive from `AgentRun.state` / `Evaluation` pass, configurable per dashboard variable. [PROPOSED — not in source]
- RISK (low): A/B "significance" computation has no source-defined method; proposed as a Langfuse/eval-provided statistic or a [PROPOSED] PyTest/query helper, not bespoke stats in the panel. [PROPOSED]
- RISK (low): OpenSearch advisory unavailability degrades RAG/eval panels — accepted per ADR 0009/0014 soft-dependency stance.
- OQ (low): Are the nine dashboards one XR each or grouped into a folder-scoped bundle? Proposed: one XR per dashboard for independent visibility control (REQ-D2-01). [PROPOSED]
- OQ (med): Do D2 templates and Coach-published agent dashboards (B10) share a single source-of-truth JSON, or fork? Proposed: D2 owns the template; B10 instantiates via composition selectors (REQ-D2-13) to avoid divergence. [PROPOSED]

## 11. References

- architecture-overview.md §11 Grafana dashboards (~L1537), §11.2 Developer dashboards (~L1563), §11 deep-link/self-publish notes (~L1545, ~L1575), §14.1 standard deliverables (~L1649), §14.4 Workstream D (~L1734), §6.5 observability, §13 testing/reporting (~L1633).
- ADR 0021 (GrafanaDashboard XRs), ADR 0018 (RBAC/OPA), ADR 0015 (Tempo+Langfuse trace_id), ADR 0009 (OpenSearch retrieval), ADR 0014 (Postgres primary; OpenSearch advisory), ADR 0011 (promptfoo evaluation vs test layers), ADR 0030 (versioning).
- Related pieces: B4 (Composition), A2 (Langfuse), A5 (ARK/Evaluation), A10 (Letta), A11 (OpenSearch), A13 (Tempo+Mimir), B6 (SDK), B10 (Coach), D1, D3.
- _meta/interface-contract.md §1.6 (GrafanaDashboard XR), §1.2 (`Evaluation`), §2 (CloudEvent taxonomy), §3.1 (Platform SDK surfaces); _meta/glossary.md (GrafanaDashboard, Knowledge Base, Coach Agent, deep-link products).
