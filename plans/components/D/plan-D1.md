# PLAN D1 — Operator integrated dashboards

> spec: SPEC-D1 · kind: COMPONENT · tier: T1
> wave: consumer (authoring-parallel; realizes after B4 + A-component metric contracts) · estimate: M
> upstream-pieces: [A1, A2, A4, A6, A7, A10, A13, A18, B4, B12, B14] · downstream-pieces: []

## 1. Implementation Strategy

Author each of the six §11.1 integrated dashboards as a namespaced `GrafanaDashboard` XR (ADR 0021) submitted via GitOps against the Composition owned by B4. Build in dependency order: stand up the XR scaffold and visibility wiring first (proves the B4 path end-to-end), then layer panels per dashboard as each upstream A-component's metric/trace/index contract lands. Where a metric or index field name is not yet published, author the panel against a [PROPOSED] placeholder name and a fixture data source so the dashboard is testable before the real backend exists, then swap to the real query.

## 2. Ordered Task List

- TASK-01: Define the six dashboard XR manifests (skeleton: `dashboardJson` stub, `folder`, `visibility`) — produces: 6 `GrafanaDashboard` XR manifests — depends-on: []
- TASK-02: Wire RBAC + OPA visibility refs for per-tenant scoping — produces: visibility policy refs + manifest fields — depends-on: [TASK-01]
- TASK-03: Author Platform overview panels + component drill-down links — produces: overview dashboardJson — depends-on: [TASK-01]
- TASK-04: Author End-to-end request-flow panels (Tempo↔Langfuse trace_id join) — produces: flow dashboardJson — depends-on: [TASK-01]
- TASK-05: Author Cost panels (team/agent/model, budget vs actual, burn rate) — produces: cost dashboardJson — depends-on: [TASK-01]
- TASK-06: Author Audit & security panels (advisory-labeled OpenSearch + policy/secret/egress) — produces: audit dashboardJson — depends-on: [TASK-01]
- TASK-07: Author Capacity panels (warm-pool, cold-start, memory headroom, broker backlog, queue depth) — produces: capacity dashboardJson — depends-on: [TASK-01]
- TASK-08: Author Test framework health panels from B14 index — produces: test-health dashboardJson — depends-on: [TASK-01]
- TASK-09: Wire §11 deep-links (Langfuse/Argo/ArgoCD/Kargo/OpenSearch/LiteLLM admin), Kargo one-per-Stage — produces: deep-link panels — depends-on: [TASK-03..08]
- TASK-10: Chainsaw + Playwright tests (reconcile, render, visibility gating, Stage-count) — produces: test suites — depends-on: [TASK-02, TASK-09]

## 3. Dependency Map

### 3.1 Upstream pieces that must ship first (HARD)
- B4 — `GrafanaDashboard` XRD + Composition + Grafana provider path (without it nothing reconciles).

### 3.2 Downstream pieces blocked on this
- None (D1 is a leaf consumer).

### 3.3 Continuous (non-blocking) inputs
- A1/A2/A4/A6/A7/A10/A13/A18 metric, trace, and index contracts (panels finalize as each lands; placeholder names until then).
- B12 CloudEvent schemas (shape of aggregated event data).
- B14 advisory test-result index (Test framework health panel).
- B22 threat model (visibility-policy review).

## 4. Parallelizable Subtasks

- After TASK-01: TASK-03 through TASK-08 (the six dashboards) are independent and fan out concurrently.
- TASK-02 (visibility) runs in parallel with panel authoring.
- TASK-09 and TASK-10 converge the fan-out.

## 5. Test Strategy

- Chainsaw: AC-D1-01 (XRs reconcile into Grafana in correct folders), AC-D1-10 (Stage-count change without error). Fixture: fake Grafana provider response if B4 path not yet live.
- Playwright: AC-D1-02..07 (each dashboard renders required panels), AC-D1-08 (cross-tenant visibility denied), AC-D1-09 (deep-links resolve). Fixtures/fakes: stub Mimir/Tempo/OpenSearch/Langfuse data sources for not-yet-landed upstreams.
- PyTest: N/A — no custom logic (add only if query-builder helpers are introduced).

## 6. PR / Branch Mapping

### 6.1 Stack position — base branch = `wave/consumer` (contains B4 spec + A-component specs)
### 6.2 PR — `piece/D1-operator-integrated-dashboards` → base `wave/consumer`; carries spec-D1 + plan-D1
### 6.3 Merge order — independent of D2/D3 siblings; rolls up to main after B4 lands

## 7. Effort Estimate

- TASK-01 S · TASK-02 S · TASK-03 M · TASK-04 M · TASK-05 M · TASK-06 M · TASK-07 M · TASK-08 S · TASK-09 M · TASK-10 M
- Rollup: M (six independent dashboards fan out; low integration complexity once B4 path exists).
- Critical path: TASK-01 → (any one dashboard, e.g. TASK-04) → TASK-09 → TASK-10.

## 8. Rollback / Reversibility

Delete the `GrafanaDashboard` XRs via GitOps; Crossplane garbage-collects the Grafana dashboards. Fully reversible — no state outside Grafana and the Git manifests. No downstream breaks (leaf consumer); operators lose integrated views and fall back to per-component dashboards.
