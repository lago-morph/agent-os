# PLAN V6-05 — Observability architecture `[PROPOSED]`

> spec: SPEC-V6-05 · kind: VIEW · tier: T1
> wave: authoring-parallel · estimate: S
> upstream-pieces: [] · downstream-pieces: []

## 1. Implementation Strategy
This view is realized, not built. The realization map follows the wave layering: **A2 (Langfuse)** and **A13 (Tempo + Mimir)** land in W0 as the two correlated telemetry planes; **A18 (audit endpoint + adapter)** lands in W1 (foundation band, depends on A11/A7) as the durable audit plane; **D1 (operator dashboards)** and **D2 (developer dashboards)** are consumer-wave pieces over the shared pipelines. The view holds when the `trace_id`-correlation, shared-OTel-path, adapter/endpoint-audit, audit-durability, dynamic-toggle, zero-off-cost, and test-result-publication invariants all hold end-to-end. Verification pivots Grafana↔Langfuse on one `trace_id`, proves audit ingestion survives an OpenSearch outage, and exercises a `LogLevel` toggle in both in-process and rolling-restart modes.

## 2. Ordered Task List
- **TASK-01:** Confirm A13 Tempo/Mimir + the OTel collector path receive traces/metrics/logs from agents, LiteLLM, Envoy, ARK, UIs — produces: collection-path checklist — depends-on: []
- **TASK-02:** Confirm A2 Langfuse receives prompts/costs/evals + promptfoo red-team and correlates to Tempo by `trace_id` (ADR 0015) — produces: trace-correlation checklist — depends-on: [TASK-01]
- **TASK-03:** Confirm A18 audit adapter/endpoint is the only audit path, Postgres+S3 system of record, OpenSearch advisory, ingestion survives OpenSearch down (ADR 0034) — produces: audit-durability checklist — depends-on: []
- **TASK-04:** Confirm the `LogLevel` dynamic toggle (in-process default vs rolling-restart for LiteLLM) with zero off-mode cost (ADR 0035), and test-result/OTel publication (ADR 0011) — produces: toggle + test-publication checklist — depends-on: [TASK-01]
- **TASK-05:** Define end-to-end view verification: trace pivot, OTel single-path, audit no-direct-write, OpenSearch-down resilience, toggle both modes, dashboards/alerts present — produces: view acceptance suite mapping (AC-01..09) — depends-on: [TASK-02, TASK-03, TASK-04]

## 3. Dependency Map
### 3.1 Upstream pieces that must ship first (HARD)
- A2 — Langfuse (consumed: LLM-grade plane + trace_id correlation).
- A13 — Tempo + Mimir (consumed: traces + long-term metrics + OTel collector).
- D1, D2 — dashboards (consumed: per-component operator/developer dashboards over shared pipelines).
- A18 — audit endpoint/adapter (consumed audit plane; realizes ADR 0034 audit invariants).
### 3.2 Downstream pieces blocked on this
- None directly (view imposes constraints; every emitting component binds to component specs).
### 3.3 Continuous (non-blocking) inputs
- B12 (CloudEvent registry for `platform.observability.*`/`platform.audit.*`); B4 (`GrafanaDashboard`/`AuditLog` XRDs); B14 test framework; B22 threat model (audit-integrity, audit-tampering patterns shape REQ-03/04).

## 4. Parallelizable Subtasks
- TASK-01 (telemetry path) and TASK-03 (audit plane) run concurrently (independent planes). Fan-out group: {TASK-01→TASK-02, TASK-01→TASK-04, TASK-03}. TASK-05 joins all.

## 5. Test Strategy
- **Chainsaw (operator/CRD):** `LogLevel` toggle reconcile in-process vs rolling-restart and platform records which pattern (AC-05); `GrafanaDashboard`/`AuditLog` XR provisioning.
- **Playwright (UI/e2e):** Grafana panel deep-links to Langfuse and back on one `trace_id` (AC-01); per-component dashboards present, no SLI/SLO artifacts required (AC-09).
- **PyTest (logic):** single-OTel-collector confirmation (AC-02); no-direct-audit-write scan (AC-03); audit ingestion succeeds with OpenSearch down + advisory rebuild (AC-04); rolling-restart toggle drops no stream (AC-06); zero off-mode cost (AC-07); test results to OpenSearch advisory + OTel (AC-08).
- Fixtures/fakes: fake OpenSearch with kill switch; synthetic `trace_id`-tagged spans; fake audit endpoint; LiteLLM deployment-shape fixture (replicas>=2, preStop, grace period).

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/auth` (views authoring band)
### 6.2 PR — `piece/V6-05-observability-architecture` → base `wave/auth`; carries spec-V6-05.md + plan-V6-05.md
### 6.3 Merge order — independent of sibling view PRs; rolls up to main with the other V6-0x views.

## 7. Effort Estimate
- TASK-01 S · TASK-02 S · TASK-03 S · TASK-04 S · TASK-05 M. Rollup: S (authoring). Critical path: TASK-01 → TASK-02/04 (and TASK-03 in parallel) → TASK-05.

## 8. Rollback / Reversibility
Reverting the view doc has no runtime effect (authoring artifact). If an invariant is found wrong, amend the SPEC and re-flag `[PROPOSED]`; the realizing component SPECs (A2/A13/D1/D2, audit plane A18) carry the enforceable change. No downstream code breaks from reverting the view itself.
