# PLAN A15 — Reloader + oauth2-proxy

> spec: SPEC-A15 · kind: COMPONENT · tier: T2
> wave: W0 · estimate: S
> upstream-pieces: [] · downstream-pieces: []

## 1. Implementation Strategy

Two independent, unmodified OSS utility installs (config-only, ADR 0007 / ADR 0035). Pin and Helm-install Reloader (annotation-driven rolling restarts) and oauth2-proxy (SSO proxy deployable), each with a deny-by-default posture, then ship the light T2 operate deliverables (docs, runbook, alerts, dashboard). oauth2-proxy's substantive auth configuration is deliberately left to B1; A15 stops at "installed, healthy, not open by default." No CRDs, no fork. Tasks are largely parallel; there is no meaningful critical path beyond install → operate deliverables.

## 2. Ordered Task List

- TASK-01: Pin + Helm-install Reloader, ArgoCD-synced; verify annotation-driven rolling restart on ConfigMap/Secret change — produces: Reloader install — depends-on: []
- TASK-02: Pin + Helm-install oauth2-proxy as the SSO proxy deployable with deny-by-default; leave auth config to B1 — produces: oauth2-proxy install — depends-on: []
- TASK-03: Operate deliverables — docs, runbook, component-failure alerts, dashboard, light HolmesGPT health-query toolset — depends-on: [TASK-01, TASK-02]
- TASK-04: Tests (Chainsaw restart-on-change, PyTest install/health) — produces: test suite — depends-on: [TASK-01, TASK-02]

## 3. Dependency Map

### 3.1 Upstream pieces that must ship first (HARD)
- None (W0 foundation; CSV upstream empty).

### 3.2 Downstream pieces blocked on this
- None in CSV. Effective: **B1** configures the installed oauth2-proxy; ADR 0035 staged-restart relies on Reloader. `[PROPOSED]` functional, not CSV edges.

### 3.3 Continuous (non-blocking) inputs
- B14 test framework, B22 threat model. A18 audit adapter (light, as it lands).

## 4. Parallelizable Subtasks

- TASK-01 and TASK-02 fully independent. TASK-03 and TASK-04 run concurrently after both installs.

## 5. Test Strategy

| AC | Layer | Notes |
|---|---|---|
| AC-A15-01, AC-A15-04 | Chainsaw | Reloader rolling restart on Secret change; staged-restart actuation of an annotated non-reloadable workload |
| AC-A15-02, AC-A15-03 | PyTest | oauth2-proxy install/health, deny-by-default check, deliverable presence |
| (none) | Playwright | N/A — A15 has no own UI (oauth2-proxy UI behavior is B1's config) |

Fakes: none required of substance; a stub upstream service behind oauth2-proxy for the deny-by-default check.

## 6. PR / Branch Mapping

### 6.1 Stack position — base branch = `wave/0` (foundation)
### 6.2 PR — `piece/A15-reloader-oauth2proxy` → base `wave/0`; carries SPEC-A15 + PLAN-A15
### 6.3 Merge order — independent of all W0 siblings; rolls up to main.

## 7. Effort Estimate

- S: all tasks. Rollup: **S** (matches CSV). Critical path: TASK-01/02 (parallel) → TASK-03/04 (parallel).

## 8. Rollback / Reversibility

Both are stateless; revert the ArgoCD releases to back out. Reverting Reloader leaves workloads to manual restarts (loss of auto-reload); reverting oauth2-proxy removes the SSO proxy in front of UIs — B1's configuration would have nothing to bind to, so reverting A15 while B1 is live would break UI auth. No durable data is lost in either case.
