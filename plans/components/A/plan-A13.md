# PLAN A13 — Tempo + Mimir

> spec: SPEC-A13 · kind: COMPONENT · tier: T1
> wave: W0 · estimate: M
> upstream-pieces: [] · downstream-pieces: []

## 1. Implementation Strategy
Install Tempo and Mimir unmodified via Helm as ArgoCD releases and wire them behind the baseline
OTel collector (Tempo) and Prometheus remote-write (Mimir), with `XObjectStore`-backed storage.
Land early (W0) so every other component can emit traces and metrics from day one. Enforce the two
ADR 0015 invariants on the receiving side: store `trace_id` unchanged for the Langfuse join, and
configure Grafana→Langfuse deep links. Implement ADR 0035 conditional span creation (zero-cost
off-mode). Build against fakes for not-yet-landed emitters and a local object-store fixture for
`XObjectStore`. Deliver the §14.1 set including HolmesGPT trace/metric query tools.

## 2. Ordered Task List
- **TASK-01:** Helm values + ArgoCD applications for Tempo and Mimir — produces: install manifests — depends-on: [].
- **TASK-02:** Wire Tempo behind OTel collector; Mimir behind Prometheus remote-write — produces: ingest wiring — depends-on: [TASK-01].
- **TASK-03:** `XObjectStore`-backed storage for both backends — produces: storage wiring — depends-on: [TASK-01].
- **TASK-04:** `trace_id`-preserving Tempo ingest + Langfuse join contract — produces: correlation test — depends-on: [TASK-02].
- **TASK-05:** Grafana→Langfuse deep-link target config — produces: deep-link config — depends-on: [TASK-04].
- **TASK-06:** `LogLevel`-gated conditional trace granularity in Tempo (zero-cost off-mode) — produces: ADR 0035 wiring — depends-on: [TASK-02].
- **TASK-07:** Non-unit test run publication path (ADR 0011) validation — produces: test-publication wiring — depends-on: [TASK-02].
- **TASK-08:** Tempo + Mimir `GrafanaDashboard` XRs + alert rules + OPA admission Rego — produces: dashboards/alerts/Rego — depends-on: [TASK-01].
- **TASK-09:** HolmesGPT Tempo-trace + Mimir-metric query toolset entries — produces: toolset — depends-on: [TASK-04].
- **TASK-10:** `platform.observability.*` events + audit of operator actions — produces: events/audit — depends-on: [TASK-02].
- **TASK-11:** Three-layer tests + docs/runbook/observability how-tos — produces: tests + docs — depends-on: [TASK-04, TASK-06, TASK-07, TASK-08].

## 3. Dependency Map
### 3.1 Upstream pieces that must ship first (HARD)
- None (W0). Foundation: B4 `XObjectStore` (fake with local object store until B4 lands); baseline
  OTel collector + Prometheus + Grafana.
### 3.2 Downstream pieces blocked on this
- None declared. Runtime consumers: every OTel-emitting component + A14 (HolmesGPT toolset, dotted edge).
### 3.3 Continuous (non-blocking) inputs
- B14 test framework (also a *producer* of test metrics via ADR 0011); B22 threat-model standards;
  dotted toolset edge `A13 -.-> A14`.

## 4. Parallelizable Subtasks
- After TASK-01: TASK-02, TASK-03, TASK-08 run concurrently.
- After TASK-02: TASK-04, TASK-06, TASK-07, TASK-10 fan out; TASK-05 and TASK-09 follow TASK-04.

## 5. Test Strategy
- **Chainsaw:** AC-A13-01 (install), AC-A13-06 (LogLevel conditional spans), AC-A13-08 (XR/alert).
- **Playwright:** AC-A13-05 (Grafana→Langfuse deep link).
- **PyTest:** AC-A13-02 (span→Tempo, metric→Mimir), AC-A13-03 (storage/restart), AC-A13-04 (trace_id join), AC-A13-07 (test publication), AC-A13-09 (HolmesGPT tools), AC-A13-10 (events + audit).
- **Fixtures/fakes:** local object store standing in for `XObjectStore`; fake emitter posting spans
  with a fixed `trace_id`; stub Langfuse trace for the join assertion; baseline collector/Prometheus
  in the test cluster; mock Grafana for deep-link config validation.

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/0`.
### 6.2 PR — `piece/A13-tempo-mimir` → base `wave/0`; carries spec-A13.md + plan-A13.md.
### 6.3 Merge order — independent of W0 siblings (A2 pairs with it for correlation but is independent at build); wave/0 rolls up to main.

## 7. Effort Estimate
- TASK-01 S · TASK-02 M · TASK-03 M · TASK-04 M · TASK-05 S · TASK-06 M · TASK-07 S · TASK-08 M · TASK-09 S · TASK-10 S · TASK-11 M.
- Rollup: **M**. Critical path: TASK-01 → TASK-02 → TASK-04 → TASK-11.

## 8. Rollback / Reversibility
Remove the ArgoCD applications; trace/metric data in `XObjectStore` is retained or dropped by choice.
Downstream impact: general distributed tracing and long-term metrics go dark — HolmesGPT loses two of
its four observability surfaces and the Tempo→Langfuse deep link breaks. Baseline Prometheus/Grafana
short-term metrics and Langfuse LLM traces remain; audit (A18) is on a separate path and unaffected.
