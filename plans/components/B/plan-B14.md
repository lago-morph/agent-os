# PLAN B14 — `agent-platform` test framework

> spec: SPEC-B14 · kind: COMPONENT · tier: T1
> wave: W3 · estimate: L
> upstream-pieces: [B9] · downstream-pieces: [B15]

## 1. Implementation Strategy
Build the `agent-platform test` orchestration as a **subcommand area of the B9 CLI image** that
reads a manifest ("what runs where") and drives **Chainsaw / Playwright / PyTest** runners,
aggregating one pass/fail (ADR 0011). Layer on the **stress-probe harness** (concurrent invocation
on Chainsaw/Playwright watching existing observability), **result emission** (non-unit → OpenSearch
advisory index + OTel metrics/logs; unit → CI-output only), **audit emission** of runs, the
**`--debug`/`--trace`** toggle wiring (scoped + exhaustively restored, ADR 0035), and **dashboard
provisioning** (the views D3 builds on). **First task is specifying the test manifest schema** —
ADR 0011 requires it before implementation. Soft-dependency degradation (warn/continue, fail only
on real failure) is a correctness property, so it is built in from the start, not bolted on.

## 2. Ordered Task List
- **TASK-01:** Specify the **test manifest schema** ("what runs where") — design-time deliverable
  required before implementation (ADR 0011/backlog §4). — produces: manifest schema — depends-on: []
- **TASK-02:** Implement manifest-driven orchestration of Chainsaw/Playwright/PyTest + result
  aggregation, as a B9 `test` subcommand. — produces: orchestrator — depends-on: [TASK-01]
- **TASK-03:** Implement soft-dependency degradation (OpenSearch/OTel unreachable ⇒ warn/continue,
  exit non-zero only on real failure). — produces: degradation logic — depends-on: [TASK-02]
- **TASK-04:** Result emission — non-unit → OpenSearch advisory index; metrics+logs → OTel; unit →
  CI-output only. — produces: result sink — depends-on: [TASK-02]
- **TASK-05:** Audit emission of runs (start/finish/pass-fail/per-layer) via the adapter/endpoint. —
  produces: audit wiring — depends-on: [TASK-02]
- **TASK-06:** `--debug`/`--trace` toggle: activate ADR-0035 toggle scoped to exercised components;
  **exhaustive restore** on normal exit, SIGTERM, crash (signal handlers + atexit + `expiresAt`
  backstop). — produces: toggle wiring — depends-on: [TASK-02]
- **TASK-07:** Stress-probe harness — N-parallel invocation on Chainsaw/Playwright watching existing
  observability for saturation. — produces: stress harness — depends-on: [TASK-02]
- **TASK-08:** Dashboard provisioning — OpenSearch result views + OTel metric series for D3. —
  produces: provisioning — depends-on: [TASK-04]
- **TASK-09:** Pin runner versions in the CLI image; self-tests + docs/runbook. — produces: image +
  tests + docs — depends-on: [TASK-03, TASK-04, TASK-05, TASK-06, TASK-07]

## 3. Dependency Map
### 3.1 Upstream pieces that must ship first (HARD)
- **B9 (`agent-platform` CLI)** — the CLI framework/image/versioned surface B14's `test` area
  plugs into.

### 3.2 Downstream pieces blocked on this
- **B15** — the GitHub Actions reference pipeline calls `agent-platform test` + the stress probe.
- **D3** — builds test framework dashboards over B14's OpenSearch index + OTel metrics.
- (Fan-out) every **A/B** component contributes tests B14 orchestrates.

### 3.3 Continuous (non-blocking) inputs
- **B14 is itself the continuous fan-out input to all A/B** (§14.7) — not wave-depth-setting.
- Consumes: **A11/OpenSearch** + **A13/OTel** as *soft* deps; **A18** audit adapter; **B12** event
  schemas; **B22** threat-model mandatory test cases.

## 4. Parallelizable Subtasks
- After TASK-02: TASK-03, TASK-04, TASK-05, TASK-06, TASK-07 are largely independent fan-out groups.
- TASK-08 follows TASK-04; TASK-09 gathers all.

## 5. Test Strategy
B14 **self-tests** (it is the framework):
- **PyTest:** aggregation, soft-degradation (AC-B14-06,13), exhaustive restore across normal/SIGTERM/
  crash (AC-B14-08), toggle scoping (AC-B14-07), promptfoo-not-a-layer (AC-B14-12), semver gate
  (AC-B14-11).
- **Chainsaw:** orchestration invokes each runner + emits audit/`platform.audit.*` (AC-B14-01,05);
  result lands in OpenSearch index (AC-B14-04,10).
- **Playwright:** a UI/HTTP-flow smoke proving the Playwright layer is actually driven (AC-B14-01).
- **Stress probe self-check:** N-parallel run surfaces a saturation signal with no locust/k6 present
  (AC-B14-03).
- **Manifest gate:** schema published + validated before impl (AC-B14-09).
- **Fakes:** unreachable OpenSearch/OTel injected to exercise degradation; fake `LogLevel` toggle
  service if ADR-0035 owners not landed.

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/3` (contains B9 from W2 rolled up).
### 6.2 PR — `piece/B14-test-framework` → base `wave/3`; carries spec-B14 + plan-B14.
### 6.3 Merge order — independent of W3 siblings (B7, B10, B19-core); must merge **before** B15 (W4).
wave/3 rolls up to main.

## 7. Effort Estimate
- TASK-01 M · TASK-02 L · TASK-03 M · TASK-04 M · TASK-05 S · TASK-06 M · TASK-07 M · TASK-08 S ·
  TASK-09 M.
- Rollup: **L** (matches CSV). Critical path: TASK-01 → 02 → 06 (toggle restore is the hardest) → 09.

## 8. Rollback / Reversibility
B14 ships as a CLI image, not a cluster workload, so rollback is reverting to the prior CLI image
tag. Backing it out **breaks CI test stages (B15)** and removes the laptop/schedule test path, and
stops result streaming to OpenSearch/D3 — but leaves no persistent cluster state (results in the
advisory index are rebuildable/disposable). The one risk on rollback is a **left-verbose cluster**
if a `--trace` run was interrupted; the `expiresAt` backstop on the `LogLevel` CR self-heals that.
