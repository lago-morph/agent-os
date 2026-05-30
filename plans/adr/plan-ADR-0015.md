# PLAN ADR-0015 — Tempo + Langfuse correlated by trace_id [PROPOSED]

> spec: SPEC-ADR-0015 · kind: ADR · tier: T1
> wave: authoring-parallel · estimate: S
> upstream-pieces: [A13, A2] · downstream-pieces: [B6, B7, A1, A6, A5, A14, D1, D2]

## 1. Implementation Strategy
Enforcement/verification map. ADR 0015 is enforced primarily by **B6/B7** (the agent + platform SDK that generate and propagate a `trace_id` at request entry and emit correlated spans) and **A1** (LiteLLM emitting the same `trace_id` to both backends) — the load-bearing invariant — over the two backends **A13** (Tempo+Mimir) and **A2** (Langfuse). **D1/D2** enforce the Grafana deep-link surfaces, **A14** consumes both backends, and **A6/A5** route general OTel spans to Tempo only. Trace granularity is gated by the ADR 0035 toggle. Conformance is tested by proving one request yields a Tempo span + Langfuse trace sharing a `trace_id`, propagation across A2A/MCP/gateway hops, the deep link, and zero span cost in toggle-off mode.

## 2. Ordered Task List
- TASK-01: Map each REQ to its enforcing component piece ID — produces: enforcement matrix — depends-on: []
- TASK-02: Verify B6/B7 + A1 emit the same `trace_id` to both backends (the invariant) — produces: SDK/gateway conformance checklist — depends-on: [TASK-01]
- TASK-03: Verify request-entry generation + propagation across A2A/MCP/gateway hops — produces: propagation trace — depends-on: [TASK-02]
- TASK-04: Verify backend routing split (non-LLM → Tempo; LLM → Langfuse) — produces: A13/A2 routing note — depends-on: [TASK-01]
- TASK-05: Verify Grafana Tempo↔Langfuse deep links (as `GrafanaDashboard` XRs) — produces: D1/D2 trace — depends-on: [TASK-04]
- TASK-06: Verify toggle-gated granularity / zero off-mode cost (ADR 0035 cross-check) — produces: PyTest set — depends-on: [TASK-02]
- TASK-07: Verify HolmesGPT consumes both backends + test-run OTel correlation (ADR 0011/0012) — produces: A14 + test trace — depends-on: [TASK-04]

## 3. Dependency Map
### 3.1 Upstream pieces that must ship first (HARD)
- A13 (Tempo+Mimir), A2 (Langfuse) — the two backends.
### 3.2 Downstream pieces blocked on this
- B6/B7 (correlated emission), D1/D2 (deep-link dashboards), A14 (consumption).
### 3.3 Continuous (non-blocking) inputs
- ADR 0035 (toggle gating), ADR 0021 (GrafanaDashboard XR), B14 test framework.

## 4. Parallelizable Subtasks
TASK-02 and TASK-04 fan out once TASK-01 lands. TASK-03/06 follow TASK-02; TASK-05/07 follow TASK-04.

## 5. Test Strategy
- AC-01 → PyTest: one LLM request → Tempo span + Langfuse trace, identical `trace_id`.
- AC-02 → PyTest+Chainsaw: A2A→MCP→gateway hop chain carries one `trace_id`, generated at entry when absent.
- AC-03 → PyTest: non-LLM Envoy/ARK span in Tempo, absent from Langfuse.
- AC-04 → Playwright: Tempo span deep-links to correlated Langfuse trace.
- AC-05 → PyTest: HolmesGPT retrieves correlated spans from both backends.
- AC-06 → PyTest: toggle off → zero spans; raise on one component → spans only for it.
- AC-07 → PyTest: non-unit test run emits OTel on Tempo/Mimir/Loki path with shared correlation.
- Fixtures/fakes: stub Tempo + Langfuse collectors, OTel collector for not-yet-landed A13/A2.

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/authoring`
### 6.2 PR — `piece/ADR-0015-tempo-langfuse-correlated-tracing` → base `wave/authoring`; carries spec + plan.
### 6.3 Merge order — independent of sibling ADR specs; rolls up to main.

## 7. Effort Estimate
S overall. Per-task: TASK-01 S, 02 M, 03 S, 04 S, 05 S, 06 S, 07 S. Critical path: TASK-01 → 02 → 03.

## 8. Rollback / Reversibility
Decision record; back out by reverting spec+plan. If reverted, B6/B7/A1 lose the `trace_id` correlation contract and D1/D2 lose the deep-link requirement; a future single-backend migration stays a config swap because the SDK centralizes span emission. No runtime artifact is deleted.
