# PLAN D2 — Developer dashboards

> spec: SPEC-D2 · kind: COMPONENT · tier: T1
> wave: consumer (authoring-parallel; realizes after B4 + A2/A5/B6 telemetry contracts) · estimate: L
> upstream-pieces: [A1, A2, A5, A7, A10, A11, A13, B4, B6, B10, B12] · downstream-pieces: []

## 1. Implementation Strategy

Author each of the nine §11.2 developer dashboards as a namespaced `GrafanaDashboard` XR (ADR 0021) submitted via GitOps against the Composition owned by B4. Stand up the XR scaffold + visibility wiring first (proves the B4 path and per-developer scoping), then fan out the nine dashboards as each upstream telemetry contract lands — Langfuse (A2) eval/prompt/cost data, ARK `Evaluation` (A5) outcomes, Letta (A10) memory signals, OpenSearch (A11) RAG/eval index, and B6 SDK emission. Where a metric, eval-score, or index field name is not yet published, author the panel against a [PROPOSED] placeholder name and a fixture data source so the dashboard is testable before the real backend exists, then swap to the real query. Build the JSON so it doubles as a reusable template for Coach (B10) / agent-published dashboards via composition selectors.

## 2. Ordered Task List

- TASK-01: Define the nine dashboard XR manifests (skeleton: `dashboardJson` stub, `folder`, `visibility`) — produces: 9 `GrafanaDashboard` XR manifests — depends-on: []
- TASK-02: Wire RBAC + OPA visibility refs for per-developer / per-tenant scoping — produces: visibility policy refs + manifest fields — depends-on: [TASK-01]
- TASK-03: Author Prompt performance panels (latency/tokens/cost/eval score by prompt version) — produces: prompt-perf dashboardJson — depends-on: [TASK-01]
- TASK-04: Author A/B comparison panels (side-by-side versions + significance) — produces: ab dashboardJson — depends-on: [TASK-01]
- TASK-05: Author Eval trend panels (rolling pass rate, failure clustering, regression) — produces: eval-trend dashboardJson — depends-on: [TASK-01]
- TASK-06: Author Cost per success panels (cost normalized by success, per agent/model/prompt) — produces: cost-per-success dashboardJson — depends-on: [TASK-01]
- TASK-07: Author Failure mode explorer panels (clusters + Langfuse trace deep-link) — produces: failure-mode dashboardJson — depends-on: [TASK-01]
- TASK-08: Author Tool usage heatmap panels — produces: tool-heatmap dashboardJson — depends-on: [TASK-01]
- TASK-09: Author Memory effectiveness panels (Letta signals) — produces: memory dashboardJson — depends-on: [TASK-01]
- TASK-10: Author Skill performance panels — produces: skill dashboardJson — depends-on: [TASK-01]
- TASK-11: Author KB RAG effectiveness panels (retrieval success/latency/hit-rate by query type; advisory-labeled) — produces: rag dashboardJson — depends-on: [TASK-01]
- TASK-12: Wire deep-links (Langfuse traces, Headlamp resource views, Git commits) across all nine — produces: deep-link panels — depends-on: [TASK-03..11]
- TASK-13: Make JSON template-reusable for B10/agent-published XRs via composition selectors — produces: parameterized templates — depends-on: [TASK-03..11]
- TASK-14: Chainsaw + Playwright tests (reconcile, render, deep-links, visibility gating, template instantiation) — produces: test suites — depends-on: [TASK-02, TASK-12, TASK-13]

## 3. Dependency Map

### 3.1 Upstream pieces that must ship first (HARD)
- B4 — `GrafanaDashboard` XRD + Composition + Grafana provider path (without it nothing reconciles).

### 3.2 Downstream pieces blocked on this
- None (D2 is a leaf consumer). B10 (Coach) optionally reuses D2 templates but is not blocked.

### 3.3 Continuous (non-blocking) inputs
- A2 (Langfuse) prompt/eval/cost contracts; A5 (ARK `Evaluation`) outcomes; A10 (Letta) memory signals; A11 (OpenSearch) RAG/eval index; A1/A13 metric contracts; B6 SDK emission attribute names — panels finalize as each lands; placeholder names until then.
- B12 CloudEvent schemas (`platform.evaluation.*` etc.).
- B10 Coach (consumes templates; non-blocking).
- B22 threat model (visibility-policy review).

## 4. Parallelizable Subtasks

- After TASK-01: TASK-03 through TASK-11 (the nine dashboards) are independent and fan out concurrently.
- TASK-02 (visibility) runs in parallel with panel authoring.
- TASK-12, TASK-13, TASK-14 converge the fan-out.

## 5. Test Strategy

- Chainsaw: AC-D2-01 (XRs reconcile into Grafana in correct folders), AC-D2-13 (template instantiable as agent-scoped XR via selector). Fixture: fake Grafana provider response if B4 path not yet live.
- Playwright: AC-D2-02..11 (each dashboard renders required panels + deep-links resolve to Langfuse/Headlamp/Git), AC-D2-12 (cross-tenant visibility denied). Fixtures/fakes: stub Langfuse/Mimir/Tempo/OpenSearch data sources and synthetic eval/run data for not-yet-landed upstreams.
- PyTest: only if cost-per-success normalization or A/B significance is implemented as a query/transform helper (AC-D2-05, AC-D2-03) — otherwise N/A.

## 6. PR / Branch Mapping

### 6.1 Stack position — base branch = `wave/consumer` (contains B4 spec + A2/A5/B6 specs)
### 6.2 PR — `piece/D2-developer-dashboards` → base `wave/consumer`; carries spec-D2 + plan-D2
### 6.3 Merge order — independent of D1/D3 siblings; rolls up to main after B4 lands

## 7. Effort Estimate

- TASK-01 S · TASK-02 S · TASK-03 M · TASK-04 M · TASK-05 M · TASK-06 M · TASK-07 M · TASK-08 S · TASK-09 M · TASK-10 S · TASK-11 M · TASK-12 M · TASK-13 M · TASK-14 M
- Rollup: L (nine dashboards plus template-reuse and significance/cost-normalization logic raise it above D1's M).
- Critical path: TASK-01 → (eval-dependent dashboard, e.g. TASK-05) → TASK-12 → TASK-14.

## 8. Rollback / Reversibility

Delete the `GrafanaDashboard` XRs via GitOps; Crossplane garbage-collects the Grafana dashboards. Fully reversible — no state outside Grafana and the Git manifests. No downstream breaks (leaf consumer); developers lose iteration-feedback views and fall back to raw Langfuse/Grafana. If B10 instantiated D2 templates, those agent-published XRs are independent copies and are unaffected by D2 removal.
