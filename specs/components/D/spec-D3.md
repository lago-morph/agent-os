# SPEC D3 — Test framework dashboards

> kind: COMPONENT · workstream: D · tier: T2
> upstream: [A11, A13, B4, B14] · downstream: [] · adrs: [0021, 0018, 0011, 0009, 0034, 0030] · views: [6.5]
> canon-glossary: <hash> · canon-interface: <hash>

## 1. Purpose & Problem Statement

The `agent-platform` test framework (B14) runs three test layers — Chainsaw, Playwright, PyTest — and streams non-unit results to the OpenSearch advisory index with metrics via OpenTelemetry (§13, ADR 0011). Maintainers need trend visibility over that data: pass/fail trends per layer and flake detection (intermittent pass/fail on unchanged tests). The §11.1 operator "Test framework health" panel (owned by D1) gives a cross-layer health glance; D3 owns the dedicated, deeper **test-framework dashboards** — pass/fail trends and flake detection (§14.4).

Each dashboard is delivered as a namespaced `GrafanaDashboard` Crossplane XR (ADR 0021), composed by Component B4 and reconciled via GitOps; visibility is RBAC + OPA controlled (ADR 0018). `[PROPOSED]` tags mark any index field or metric name not fixed by Canon.

## 2. Scope

### 2.1 In scope
- Two test-framework dashboards delivered as `GrafanaDashboard` XRs: Pass/fail trends (per Chainsaw/Playwright/PyTest layer, over time) and Flake detection (tests with non-deterministic outcomes on unchanged code).
- Panel queries against the B14 OpenSearch advisory test-result index and OTel metrics in Mimir (§13, ADR 0011).
- Deep-links to CI run output / commit and to D1's operator Test-framework-health panel.

### 2.2 Out of scope (and where it lives instead)
- The `GrafanaDashboard` XRD, Composition, and Grafana provider path — **Component B4** (ADR 0021).
- The test framework, runners, result indexing, and OTel publishing — **Component B14** (§13).
- The §11.1 operator cross-layer health panel — **Component D1** (§11.1); D3 deep-links to/from it.
- Developer eval/A-B/eval-trend dashboards — **Component D2** (§11.2); D3 covers the test layers, not promptfoo evaluation (ADR 0011 distinguishes them).
- Alert rules on flake/failure — owned per-component (§14.1); D3 visualizes state only.

## 3. Context & Dependencies

Upstream consumed:
- **B4** — `GrafanaDashboard` XRD + Composition + Grafana provider path; D3 manifests are claims against it.
- **B14** — the `agent-platform test` framework: the OpenSearch advisory test-result index and OTel metrics D3 queries; pass/fail/per-layer results and run metadata.
- **A11 (OpenSearch)** — backend hosting the advisory test-result index.
- **A13 (Tempo + Mimir)** — Mimir hosts OTel test metrics; Grafana data source.

ADRs honored:
- **ADR 0021** — namespaced `GrafanaDashboard` XRs; RBAC + OPA visibility.
- **ADR 0018** — RBAC-as-floor / OPA-as-restrictor visibility.
- **ADR 0011** — three test layers are the structural contract; promptfoo is evaluation, not a layer — D3 dashboards cover Chainsaw/Playwright/PyTest only. Non-unit results stream to OpenSearch; trend dashboards over that index serve the role earlier slated for Mimir-only metrics.
- **ADR 0009 / 0034** — OpenSearch is advisory/rebuildable, never system of record; D3 panels treat it as non-authoritative and degrade gracefully if unreachable.
- **ADR 0030** — each XR tracks the B4-owned XRD API version.

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
- `GrafanaDashboard` (XR, namespaced; XR form `XGrafanaDashboard`) — fields used: `dashboardJson`, `folder`, `visibility`. One XR per dashboard (two total). Version tracks the B4-owned XRD.

### 4.2 APIs / SDK surfaces
- Grafana data-source query APIs against **OpenSearch** (B14 advisory test-result index) and **Mimir** (OTel test metrics). Concrete index field names (test id, layer, outcome, run id, commit, duration) and metric names are **[PROPOSED — not in source]**; source names the index and the dashboards' intent (pass/fail trends, flake detection), not the field identifiers.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
- Emitted: none. D3 is visualization-only.
- Consumed (indirectly): test-run audit/result data originates under `platform.audit.*` (test runs are first-class auditable activity, §13) and is surfaced via the B14 advisory index, not consumed by D3 as raw events. Per-event-type names deferred to B12.

### 4.4 Data schemas / connection-secret contracts
- N/A — D3 provisions no substrate primitives and writes no connection secrets; it reads Grafana data sources provisioned by A11 / A13.

## 5. OSS-vs-Custom Decision

Upstream project: **Grafana**, reconciled via the Crossplane Grafana provider path owned by B4. Decision: **config/author** — D3 authors two dashboard JSONs + `GrafanaDashboard` XR manifests over B14's existing index/metrics; no new engine. Rationale: §13 (ADR 0011 update) states trend dashboards over the OpenSearch index serve trend analysis; flake/defect grouping tools (Allure/ReportPortal) are explicitly deferred until felt pain. ADR linkage: ADR 0021, ADR 0011.

## 6. Functional Requirements

- REQ-D3-01: Each dashboard SHALL be delivered as a distinct namespaced `GrafanaDashboard` XR with `dashboardJson`, `folder`, and `visibility` set.
- REQ-D3-02: The Pass/fail trends dashboard SHALL show pass/fail rate over time broken down by test layer (Chainsaw, Playwright, PyTest) (§11.1, §13).
- REQ-D3-03: The Flake detection dashboard SHALL identify tests with non-deterministic outcomes (passing and failing on unchanged code) over a configurable window.
- REQ-D3-04: Both dashboards SHALL source from the B14 OpenSearch advisory test-result index (and Mimir OTel metrics where applicable), treating OpenSearch as advisory/non-authoritative (ADR 0009/0011).
- REQ-D3-05: Dashboards SHALL deep-link a flaky/failing test to its CI run output / commit and to D1's operator Test-framework-health panel.
- REQ-D3-06: Dashboard `visibility` SHALL be RBAC + OPA enforced (ADR 0018, ADR 0021).

## 7. Non-Functional Requirements

- Security / multi-tenancy (§6.9): visibility RBAC-floored, OPA-restricted (ADR 0018).
- Observability (§6.5): consumes the shared B14/OTel/OpenSearch pipeline; adds no test-only stack (ADR 0011).
- Reliability: with OpenSearch (advisory) unreachable, panels degrade to "data unavailable" rather than block (ADR 0009/0034; §13 soft-dependency stance).
- Versioning (ADR 0030): each XR tracks the B4-owned XRD API version; JSON revisions flow through GitOps.
- Scale: queries bound time-range/cardinality (per test id / per layer); [PROPOSED — not in source] specific budgets.

## 8. Cross-Cutting Deliverable Checklist

- Helm/manifests: Applicable — two `GrafanaDashboard` XR manifests in Git (GitOps).
- Per-product docs (10.5): N/A — D3 is not a Workstream A product; maintainer docs live in Workstream C (C7).
- Runbook (10.7): N/A — owned by Workstream C; D3 dashboards are referenced, not operated.
- Alerts: N/A — flake/failure alert rules owned per-component (§14.1); D3 visualizes state only.
- Grafana dashboard (Crossplane XR): Applicable — core deliverable.
- Headlamp plugin: N/A — no plugin; deep-links only.
- OPA/Rego integration: Applicable — `visibility` enforced via OPA (ADR 0018/0021).
- Audit emission (ADR 0034): N/A — read-only visualization; the audited test runs emit via B14's audit path, not D3.
- Knative trigger flow: N/A — emits/consumes no CloudEvents directly.
- HolmesGPT toolset: N/A — per-component toolsets are Workstream A.
- 3-layer tests (Chainsaw/Playwright/PyTest): Applicable — Chainsaw asserts XR reconciliation; Playwright asserts render + visibility + deep-links; PyTest N/A (no custom logic).
- Tutorials & how-tos: N/A — owned by Workstream E/C.

## 9. Acceptance Criteria

- AC-D3-01: Applying D3 manifests creates two `GrafanaDashboard` XRs that reconcile to two visible Grafana dashboards in the correct folder. (REQ-D3-01)
- AC-D3-02: Pass/fail trends shows a per-layer (Chainsaw/Playwright/PyTest) pass/fail trend line over a selectable window. (REQ-D3-02)
- AC-D3-03: Flake detection lists tests that both passed and failed on the same commit/unchanged code within the window. (REQ-D3-03)
- AC-D3-04: With the OpenSearch advisory index populated by B14, both dashboards render real data and label it advisory; with it unreachable they show "data unavailable" without erroring the dashboard. (REQ-D3-04)
- AC-D3-05: A flaky/failing test row deep-links to its CI run/commit and to D1's Test-framework-health panel. (REQ-D3-05)
- AC-D3-06: A user without RBAC/OPA grant cannot load the D3 dashboards for another tenant. (REQ-D3-06)

## 10. Risks & Open Questions

- RISK (med): The B14 index field schema (test id, layer, outcome, commit, run id) is not specified in source; queries depend on it landing. Reconciliation: author against [PROPOSED — not in source] field names, finalize when B14 publishes the index contract. [PROPOSED]
- RISK (low): Flake detection definition (what window / how many flips = "flaky") is not in source. Proposed: configurable dashboard variable defaulting to "≥1 pass and ≥1 fail on the same commit in the window". [PROPOSED — not in source]
- RISK (low): OpenSearch advisory unavailability degrades panels — accepted per ADR 0009/0034.
- OQ (low): Two XRs or one bundled dashboard with two tabs? Proposed: two XRs for independent visibility (REQ-D3-01). [PROPOSED]
- OQ (low): Does D3's pass/fail-trends overlap D1's §11.1 panel? Resolved by scope: D1 ships the cross-layer health glance and deep-links to D3 for the deeper per-test trend + flake views; D3 owns the detail (consistent with D1 spec OQ). [PROPOSED]

## 11. References

- architecture-overview.md §11 Grafana dashboards (~L1537), §11.1 Test framework health item (~L1559), §11.2 (~L1563), §13 Testing framework / reporting (~L1613, ~L1633), §14.4 Workstream D (~L1734).
- ADR 0021 (GrafanaDashboard XRs), ADR 0018 (RBAC/OPA), ADR 0011 (three layers vs promptfoo; OpenSearch trend reporting), ADR 0009 (OpenSearch retrieval/advisory), ADR 0034 (audit/advisory), ADR 0030 (versioning).
- Related pieces: B4 (Composition), B14 (test framework + index), A11 (OpenSearch), A13 (Mimir), D1, D2.
- _meta/interface-contract.md §1.6 (GrafanaDashboard XR), §2 (CloudEvent taxonomy); _meta/glossary.md (GrafanaDashboard, agent-platform).
