# PLAN F5 — Scale evaluation

> spec: SPEC-F5 · kind: COMPONENT · tier: T2
> wave: authoring-parallel (consumes A/B; runs at end of v1.0) · estimate: M
> upstream-pieces: [A1, A6, A4, A18, B14] · downstream-pieces: []

## 1. Implementation Strategy

F5 measures actual scale on the load-bearing paths and reports gaps versus expected v1.0 scale — no SLO, no HA build, no custom load harness (all deferred: future §3, §1, §14). The approach: author repeatable stress probes on the B14 harness (Chainsaw/Playwright concurrent invocation, ADR 0011), run them on the AWS/EKS substrate for production-representative numbers, read results from Mimir/Tempo/Langfuse, and compile a measured-vs-expected report with a prioritized gap list. Critical path: confirm the expected-scale baseline (itself `[PROPOSED]`) → author probes → AWS measurement runs (parallel per path) → gap report + feed-forward to F1/F2/F4/F6.

## 2. Ordered Task List

- **TASK-01:** Pin the assumed "expected v1.0 scale" + the realistic load profile (multi-tenant), get it confirmed; instrument the measurement read-path (Mimir/Tempo/Langfuse, trace_id correlation) — produces: scale baseline + measurement plan — depends-on: [].
- **TASK-02:** Gateway throughput probe — sustained concurrent load through LiteLLM at `replicas>=2`; capture throughput/latency incl. OPA-callback + audit overhead — produces: gateway numbers — depends-on: [TASK-01].
- **TASK-03:** In-load rolling-restart probe — restart LiteLLM under load; measure SSE-safe drain error/latency impact (ADR 0035) — produces: restart-under-load numbers — depends-on: [TASK-02].
- **TASK-04:** Sandbox cold-start probe — measure cold-start; vary `warmPoolSize`/`hibernationEnabled` — produces: cold-start numbers — depends-on: [TASK-01].
- **TASK-05:** Broker backlog probe — sustained CloudEvent load on NATS JetStream; measure backlog/lag + post-burst recovery — produces: broker numbers — depends-on: [TASK-01].
- **TASK-06:** Audit-path scale probe — `audit_events` growth + ~5-min batch keep-up under load; hand numbers to F1 (ISM/retention) + F2 (reindex volume) — produces: audit-scale numbers — depends-on: [TASK-01].
- **TASK-07:** Gap report — measured vs expected per path, prioritized gap list, explicitly no SLO; label every number by substrate (AWS representative / kind non-representative) — produces: scale-evaluation report — depends-on: [TASK-02, TASK-03, TASK-04, TASK-05, TASK-06].
- **TASK-08:** Capacity runbook (known limits, backlog-recovery, cold-start tuning) feeding F6; how-tos (run a scale probe; read the report); candidate-SLI note for future §3 — depends-on: [TASK-07].
- **TASK-09:** Repeatable 3-layer probe set on B14 (Chainsaw concurrent AgentRun/Sandbox; Playwright concurrent gateway/UI; PyTest metric extraction) mapping ACs — depends-on: [TASK-02, TASK-04, TASK-05, TASK-06].

## 3. Dependency Map

### 3.1 Upstream that must ship first (HARD)
- **A1** — LiteLLM gateway (throughput path, at `replicas>=2`, incl. OPA/audit overhead).
- **A6** — agent-sandbox (cold-start path; `warmPoolSize`/`hibernationEnabled`).
- **A4** — NATS JetStream broker (backlog/lag). **A18** — audit path (batch keep-up).
- **B14** — test framework harness (Chainsaw/Playwright concurrency, ADR 0011) probes are authored on.

### 3.2 Downstream blocked on this
- None (terminal). F1 (retention sizing), F2 (reindex volume), F4 (DoS baseline), F6 (capacity runbook) consume F5 numbers.

### 3.3 Continuous (non-blocking) inputs
- **B2** LiteLLM callbacks (overhead measured). **A13** Tempo/Mimir + **A2** Langfuse (measurement read-path). **D3** test-framework dashboards (may render F5 metrics). Component owners (confirm expected-scale baseline, R2/OQ1).

## 4. Parallelizable Subtasks
- After TASK-01: TASK-02, TASK-04, TASK-05, TASK-06 are an independent fan-out (gateway / cold-start / broker / audit paths).
- TASK-03 (restart-under-load) chains off TASK-02 (same gateway harness).
- TASK-08 (runbook/how-tos) and TASK-09 (probe set) parallelize after the report.

## 5. Test Strategy
- **Chainsaw (operator/CRD):** AC-F5-02 (concurrent Sandbox cold-start warm-on/off), AC-F5-03 (concurrent AgentRun → broker backlog/recovery) — these probes *are* the measurement.
- **Playwright (UI/e2e):** AC-F5-01 (concurrent gateway invocation throughput/latency), AC-F5-07 (in-load restart impact).
- **PyTest (logic):** AC-F5-04 (audit growth/batch keep-up extraction), AC-F5-05 (measured-vs-expected report + no-SLO assertion), AC-F5-06 (probe re-run reproducibility), metric aggregation from Mimir/Tempo/Langfuse.
- **Fixtures/fakes:** AWS/EKS substrate required for representative numbers (kind labeled non-representative, REQ-F5-07); multi-tenant load fixture; load-ceiling note if Chainsaw/Playwright concurrency saturates before the component (R3 → future §14 harness).

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/F` (A1, A6, A4, A18, B14 merged to main).
### 6.2 PR — `piece/F5-scale-evaluation` → base `wave/F`; carries SPEC-F5 + PLAN-F5.
### 6.3 Merge order — independent of F-siblings; F1/F2/F4/F6 reference F5 numbers post-merge.

## 7. Effort Estimate
- Baseline + probe authoring (TASK-01): M (baseline confirmation + harness setup).
- Measurement runs (TASK-02..06): M (AWS runs across four paths, parallelizable).
- Report + runbook + probe set (TASK-07..09): S.
- **Rollup: M** (matches CSV). **Critical path:** TASK-01 → 02 → 07 → 08.

## 8. Rollback / Reversibility
F5 is measurement — it makes no lasting platform change; rollback is discarding load runs and tearing down the load-test cluster (the report, gap list, and probe set persist as artifacts). **No downstream break** on revert (F5 is terminal); reverting only removes the measured baselines, re-opening the scale-readiness gate and the inputs F1/F2/F4 relied on (those revert to their `[PROPOSED]` assumptions).
