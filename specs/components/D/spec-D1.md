# SPEC D1 — Operator integrated dashboards

> kind: COMPONENT · workstream: D · tier: T1
> upstream: [A1, A2, A4, A6, A7, A10, A13, A18, B4, B12, B14] · downstream: [] · adrs: [0021, 0015, 0018, 0034, 0035, 0040] · views: [6.5]
> canon-glossary: <hash> · canon-interface: <hash>

## 1. Purpose & Problem Statement

Operators need cross-cutting, integrated views that span products to run the platform — single-screen platform health, end-to-end agent request flow, cost, audit/security, and capacity. Per-component dashboards are mostly vendor-provided and ship with each Workstream A component (§14.1, §11); they do not give an operator the integrated picture. This component (D1) owns the high-leverage **integrated / cross-cutting operator dashboards** described in §11.1.

Each dashboard is delivered as a namespaced `GrafanaDashboard` Crossplane XR (ADR 0021), composed by Component B4's Composition and reconciled into Grafana via the same declarative GitOps path as every other platform resource. Visibility is governed by Kubernetes RBAC and OPA per the RBAC-as-floor / OPA-as-restrictor model (ADR 0018). [PROPOSED] tags mark any panel data source, metric name, or field not fixed by Canon.

## 2. Scope

### 2.1 In scope
- The six §11.1 integrated operator dashboards: Platform overview; End-to-end agent request flow; Cost; Audit and security; Capacity; Test framework health.
- Authoring each as a `GrafanaDashboard` XR manifest (the `dashboardJson`, `folder`, `visibility`) submitted via GitOps.
- Drill-down / deep-link wiring into the Headlamp deep-link inventory targets (Langfuse, OpenSearch Dashboards, Argo Workflows, ArgoCD, LiteLLM admin, Kargo) per §11.
- Panel query definitions against the platform metric/trace/index backends (Mimir, Tempo, OpenSearch advisory index, Langfuse).

### 2.2 Out of scope (and where it lives instead)
- The `GrafanaDashboard` XRD, its Composition, and the Grafana provider reconciliation path — **Component B4** (§11, ADR 0021).
- Per-component dashboards — each **Workstream A component** (§14.1, §11).
- Developer-facing dashboards — **Component D2** (§11.2).
- The Test framework health dashboard's flake-detection deep-dive — D1 ships the §11.1 "pass/fail trends" cross-layer panel; **Component D3** owns the dedicated test-framework dashboards including flake detection (§14.4).
- SLO dashboards, error-budget tracking, operations-metrics summaries — deferred to `future-enhancements.md` (§11.1).
- Alert rules themselves — owned per-component (§14.1); D1 only visualizes their firing state.

## 3. Context & Dependencies

Upstream consumed:
- **B4** — the `GrafanaDashboard` XRD + Composition + Grafana provider path; D1 manifests are namespace-scoped XRs against it (Crossplane v2, ADR 0044).
- **A13 (Tempo + Mimir)** — metrics (Mimir) and traces (Tempo) backends queried by panels.
- **A2 (Langfuse)** — LLM trace/cost data and deep-link target for request-flow and cost dashboards.
- **A1 (LiteLLM)** — gateway routing/cost/failover metrics and `platform.gateway.*` events.
- **A6 (agent-sandbox + Envoy)** — sandbox warm-pool / cold-start / egress-denial signals (Capacity, Audit & security).
- **A7 (OPA)** — policy-violation rate signals (Audit & security).
- **A4 (Knative + NATS)** — broker backlog / queue depth (Capacity).
- **A10 (Letta)** — memory-backend headroom (Capacity).
- **A18 (audit endpoint/adapter)** — audit volume and anomaly signals (Audit & security).
- **B12 (CloudEvent schema registry)** — defines `platform.*` event shapes panels aggregate.
- **B14 (test framework)** — OpenSearch advisory test-result index for the Test framework health panel.

ADRs honored:
- **ADR 0021** — dashboards are namespaced `GrafanaDashboard` XRs; visibility RBAC + OPA controlled.
- **ADR 0018** — RBAC-as-floor / OPA-as-restrictor governs dashboard visibility.
- **ADR 0015** — Tempo + Langfuse correlated by `trace_id`; the request-flow dashboard relies on this correlation.
- **ADR 0034** — audit data originates from the Postgres+S3 system of record via the audit pipeline; OpenSearch is advisory only — the Audit dashboard must treat OpenSearch as a non-authoritative index.
- **ADR 0035** — dashboards may surface `LogLevel` / trace-granularity state but do not own the toggle.
- **ADR 0040** — Kargo deep-link is one-per-Stage; the Stage list is evolving (single Stage in v1.0).

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
- `GrafanaDashboard` (namespace-scoped XR, Crossplane v2) — fields used: `dashboardJson`, `folder`, `visibility` (RBAC + OPA-controlled). One XR per dashboard. Version tracks the XRD owned by B4 (per ADR 0030/0044 conversion-webhook policy).

### 4.2 APIs / SDK surfaces
- Grafana data-source query APIs against **Mimir** (PromQL), **Tempo** (trace queries), **OpenSearch** (advisory index), and **Langfuse** as panel/deep-link target. Concrete PromQL metric names and OpenSearch index fields are **[PROPOSED — not in source]**; source names the dashboards and their intent, not metric identifiers.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
- Emitted: none. D1 is a visualization-only consumer.
- Consumed (indirectly, as aggregated backend data): `platform.gateway.*` (routing/cost), `platform.policy.*` (violations), `platform.audit.*` (audit volume), `platform.security.*` (egress denials, security signals), `platform.observability.*` (threshold crossings), `platform.lifecycle.*` (run/sandbox state for flow + capacity). Per-event-type names are deferred to B12.

### 4.4 Data schemas / connection-secret contracts
- N/A — D1 provisions no substrate primitives and writes no connection secrets; it reads existing Grafana data sources provisioned by A13 / A11. Per-primitive field lists not applicable.

## 5. OSS-vs-Custom Decision

Upstream project: **Grafana** (dashboards) reconciled via the Crossplane Grafana provider path owned by B4. Decision: **config/author**, not build-new — D1 authors dashboard JSON and `GrafanaDashboard` XR manifests; it reuses vendor panels where they exist and composes integrated views on top. Rationale: §11 states the high-leverage work is integrated/developer views authored as Compositions; the reconciliation engine is B4. ADR linkage: ADR 0021.

## 6. Functional Requirements

- REQ-D1-01: Each of the six §11.1 dashboards SHALL be delivered as a distinct namespaced `GrafanaDashboard` XR with `dashboardJson`, `folder`, and `visibility` set.
- REQ-D1-02: The Platform overview dashboard SHALL present single-screen health of every platform component with drill-down links to component-specific views.
- REQ-D1-03: The End-to-end agent request flow dashboard SHALL render one trace pattern broken into per-stage latency, correlated by `trace_id` across Tempo and Langfuse (ADR 0015).
- REQ-D1-04: The Cost dashboard SHALL break cost down per team, per agent, and per model, showing budget vs. actual and burn rate.
- REQ-D1-05: The Audit and security dashboard SHALL surface anomaly indicators, policy-violation rate, secret-rotation lag, and egress denials, treating the OpenSearch index as advisory and not authoritative (ADR 0034).
- REQ-D1-06: The Capacity dashboard SHALL show sandbox warm-pool utilization, cold-start rate, memory-backend headroom, broker backlog, and queue depth.
- REQ-D1-07: The Test framework health dashboard SHALL show pass/fail trends across Chainsaw, Playwright, and PyTest layers sourced from the B14 advisory test-result index.
- REQ-D1-08: Dashboard `visibility` SHALL be enforced via Kubernetes RBAC + OPA so a tenant's dashboards are not visible to another tenant unless explicitly shared (ADR 0018, ADR 0021).
- REQ-D1-09: Dashboards SHALL deep-link into the §11 Headlamp deep-link inventory targets relevant to each view (e.g. Langfuse from request-flow/cost; Argo Workflows/ArgoCD/Kargo from capacity/promotion; OpenSearch Dashboards from audit).
- REQ-D1-10: The Kargo deep-link, where present, SHALL render one link per Stage and tolerate an evolving Stage list (single Stage in v1.0) (ADR 0040).

## 7. Non-Functional Requirements

- Security / multi-tenancy (§6.9): visibility is RBAC-floored and OPA-restricted; no dashboard exposes cross-tenant data absent explicit share (ADR 0018).
- Observability (§6.5): dashboards are the operator observability surface; they consume Mimir/Tempo/Langfuse/OpenSearch and add no parallel telemetry stack.
- Scale: panel queries SHALL bound time-range and cardinality to remain responsive at v1.0 component counts; [PROPOSED — not in source] specific cardinality budgets.
- Versioning (ADR 0030): each XR tracks the B4-owned XRD API version; dashboard JSON revisions flow through GitOps with normal review.
- Reliability: with OpenSearch (advisory) unreachable, audit/test panels degrade to "data unavailable" rather than block the rest of the dashboard (mirrors ADR 0034 / §13 soft-dependency stance).

## 8. Cross-Cutting Deliverable Checklist

- Helm/manifests: Applicable — `GrafanaDashboard` XR manifests in Git (GitOps).
- Per-product docs (10.5): N/A — D1 is not a Workstream A product; operator usage docs live in Workstream C / E.
- Runbook (10.7): N/A — owned by Workstream C cross-cutting runbooks (§10.7); D1 dashboards are referenced by them.
- Alerts: N/A — alert rules owned per-component (§14.1); D1 visualizes firing state only.
- Grafana dashboard (Crossplane XR): Applicable — this is the core deliverable.
- Headlamp plugin: N/A — D1 consumes the §11 deep-link inventory provided by A9/A22; it authors no plugin.
- OPA/Rego integration: Applicable — dashboard `visibility` enforced via OPA (ADR 0018/0021); D1 contributes/uses visibility policy refs.
- Audit emission (ADR 0034): N/A — read-only visualization emits no audit-relevant actions of its own.
- Knative trigger flow: N/A — D1 emits/consumes no CloudEvents directly (it reads aggregated backends).
- HolmesGPT toolset: N/A — toolset contributions are per-component (Workstream A); D1 ships no metrics/log tools.
- 3-layer tests (Chainsaw/Playwright/PyTest): Applicable — Chainsaw asserts XR reconciliation into Grafana; Playwright asserts dashboard render + visibility gating; PyTest N/A (no custom logic) unless query-builder helpers are added.
- Tutorials & how-tos: N/A — operator training/how-tos owned by Workstream E/C (§12.1, §14.4 leaves docs to C/E).

## 9. Acceptance Criteria

- AC-D1-01: Applying the D1 manifests creates six `GrafanaDashboard` XRs that reconcile to six visible Grafana dashboards in the correct folders. (REQ-D1-01)
- AC-D1-02: Platform overview shows a status tile per platform component and each tile drill-downs to its component view. (REQ-D1-02)
- AC-D1-03: For a sample agent run, the request-flow dashboard shows ≥2 stages with per-stage latency joined on a single `trace_id` across Tempo and Langfuse. (REQ-D1-03, ADR 0015)
- AC-D1-04: Cost dashboard renders budget-vs-actual and burn-rate panels filterable by team, agent, and model. (REQ-D1-04)
- AC-D1-05: Audit & security dashboard renders policy-violation-rate, secret-rotation-lag, and egress-denial panels and labels OpenSearch-sourced panels as advisory. (REQ-D1-05)
- AC-D1-06: Capacity dashboard renders warm-pool utilization, cold-start rate, memory headroom, broker backlog, and queue-depth panels. (REQ-D1-06)
- AC-D1-07: Test framework health dashboard shows Chainsaw/Playwright/PyTest pass/fail trend lines from the B14 index. (REQ-D1-07)
- AC-D1-08: A user without RBAC/OPA grant for tenant B cannot load tenant B's D1 dashboards. (REQ-D1-08)
- AC-D1-09: Each dashboard exposes working deep-links to its relevant §11 inventory targets. (REQ-D1-09)
- AC-D1-10: With two Stages configured, the Kargo deep-link panel renders two links; with one, exactly one — no error on count change. (REQ-D1-10)

## 10. Risks & Open Questions

- RISK (med): Concrete metric/index field names are not in source; dashboards depend on per-component metric naming landing first. Reconciliation: panel queries authored against [PROPOSED — not in source] names, finalized as A-components publish metric contracts. [PROPOSED]
- RISK (low): OpenSearch advisory unavailability degrades audit/test panels — accepted per ADR 0034 soft-dependency stance.
- OQ (med): Does the Test framework health panel in D1 duplicate D3, or should D1 embed/deep-link D3's dashboard? Proposed: D1 ships the §11.1 cross-layer trend panel and deep-links to D3 for flake-detail, avoiding divergence. [PROPOSED]
- OQ (low): Are the six dashboards one XR each or grouped into a folder-scoped bundle? Proposed: one XR per dashboard for independent visibility control (REQ-D1-01). [PROPOSED]
- OQ (low): Is per-tenant scoping done by templating one XR set per namespace, or shared XRs with OPA row-level filters? Source fixes namespaced XR + OPA visibility but not the templating mechanism. [PROPOSED — not in source]

## 11. References

- architecture-overview.md §11 Grafana dashboards (~L1537), §11.1 Operator dashboards (~L1552), §11 deep-link inventory (~L1550), §14.1 standard deliverables (~L1649), §14.4 Workstream D (~L1734), §6.5 observability, §13 testing soft-dependencies (~L1629).
- ADR 0021 (GrafanaDashboard XRs), ADR 0018 (RBAC/OPA), ADR 0015 (Tempo+Langfuse trace_id), ADR 0034 (audit pipeline), ADR 0035 (log-level toggle), ADR 0040 (Kargo Stages).
- Related pieces: B4 (Composition), A13 (Tempo+Mimir), A2 (Langfuse), D2, D3, B14.
- _meta/interface-contract.md §1.6 (GrafanaDashboard XR), §2 (CloudEvent taxonomy); _meta/glossary.md (GrafanaDashboard, deep-link products).
