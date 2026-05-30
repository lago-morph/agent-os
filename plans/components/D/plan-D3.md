# PLAN D3 — Test framework dashboards

> spec: SPEC-D3 · kind: COMPONENT · tier: T2
> wave: consumer (authoring-parallel; realizes after B4 + B14 index contract) · estimate: S
> upstream-pieces: [A11, A13, B4, B14] · downstream-pieces: []

## 1. Implementation Strategy

Author two `GrafanaDashboard` XRs (Pass/fail trends, Flake detection) via GitOps against B4's Composition, querying B14's OpenSearch advisory test-result index and OTel/Mimir metrics. Stand up the XR scaffold + visibility first, then build panels against B14's index field contract; where field names are not yet published, query [PROPOSED] placeholders over a fixture index and swap to real fields once B14 lands. Treat OpenSearch as advisory throughout (graceful "data unavailable" degradation).

## 2. Ordered Task List

- TASK-01: Define two dashboard XR manifests (skeleton `dashboardJson`, `folder`, `visibility`) — produces: 2 `GrafanaDashboard` XR manifests — depends-on: []
- TASK-02: Wire RBAC + OPA visibility refs — produces: visibility policy refs + fields — depends-on: [TASK-01]
- TASK-03: Author Pass/fail trends panels (per-layer trend over time) — produces: pass-fail dashboardJson — depends-on: [TASK-01]
- TASK-04: Author Flake detection panels (non-deterministic-outcome detection over window) — produces: flake dashboardJson — depends-on: [TASK-01]
- TASK-05: Wire deep-links (CI run/commit; D1 Test-framework-health panel) — produces: deep-link panels — depends-on: [TASK-03, TASK-04]
- TASK-06: Chainsaw + Playwright tests (reconcile, render, advisory degradation, visibility, deep-links) — produces: test suites — depends-on: [TASK-02, TASK-05]

## 3. Dependency Map

### 3.1 Upstream pieces that must ship first (HARD)
- B4 — `GrafanaDashboard` XRD + Composition + Grafana provider path.
- B14 — the test framework and its OpenSearch advisory test-result index (D3's data source; field contract gates real queries).

### 3.2 Downstream pieces blocked on this
- None (D3 is a leaf consumer). D1 deep-links to D3 but is not blocked by it.

### 3.3 Continuous (non-blocking) inputs
- A11 (OpenSearch) index availability; A13 (Mimir) OTel test metrics — panels finalize as B14 publishes the index/metric contract; placeholders until then.
- B22 threat model (visibility-policy review).

## 4. Parallelizable Subtasks

- After TASK-01: TASK-03 and TASK-04 are independent and fan out concurrently; TASK-02 runs in parallel.
- TASK-05 and TASK-06 converge them.

## 5. Test Strategy

- Chainsaw: AC-D3-01 (two XRs reconcile into Grafana in the correct folder). Fixture: fake Grafana provider response if B4 path not yet live.
- Playwright: AC-D3-02/03 (per-layer trend + flake list render), AC-D3-04 (advisory label + graceful degradation when OpenSearch unreachable), AC-D3-05 (deep-links resolve), AC-D3-06 (cross-tenant visibility denied). Fixtures: synthetic B14 index documents (passing/failing/flaky) for not-yet-landed B14.
- PyTest: N/A — no custom logic (flake definition lives as a dashboard variable, not code).

## 6. PR / Branch Mapping

### 6.1 Stack position — base branch = `wave/consumer` (contains B4 + B14 specs)
### 6.2 PR — `piece/D3-test-framework-dashboards` → base `wave/consumer`; carries spec-D3 + plan-D3
### 6.3 Merge order — independent of D1/D2 siblings; rolls up to main after B4 + B14 land

## 7. Effort Estimate

- TASK-01 S · TASK-02 S · TASK-03 S · TASK-04 M · TASK-05 S · TASK-06 S
- Rollup: S (two dashboards; flake detection is the only non-trivial query).
- Critical path: TASK-01 → TASK-04 → TASK-05 → TASK-06.

## 8. Rollback / Reversibility

Delete the two `GrafanaDashboard` XRs via GitOps; Crossplane garbage-collects the dashboards. Fully reversible — no state outside Grafana and Git. No downstream breaks (leaf consumer); maintainers fall back to D1's cross-layer health panel and raw CI output.
