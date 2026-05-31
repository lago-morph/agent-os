# PLAN A23 — Kargo promotion fabric

> spec: SPEC-A23 · kind: COMPONENT · tier: T1
> wave: W4 · estimate: L
> upstream-pieces: [A7, B19] · downstream-pieces: [B5]

## 1. Implementation Strategy

Install Kargo (config/wrap, no fork) single-cluster with a single Stage, wire it to the path-based single-mainline GitOps layout, and configure the Warehouse + per-Stage verification. Then layer the platform-specific compositions: OPA promotion-action gating (A7), `Approval`-CRD human gates (B19), audit emission (A18 adapter), and identity (Keycloak OIDC + cluster-OIDC workload identity). Prove promotion mechanics early on the two-kind (dev+staging) CI setup so the promotion path is exercised without AWS. Treat the Stage list as evolving — build everything so a second/third Stage is additive config, not rework. Mock B19's create-Approval/report-back surface and the OPA promotion-action decision until those land; the Kargo Headlamp plugin is B5's and co-lands in W4 (coordinate the deep-link inventory entry). Pin and validate a tested Kargo version against the OPA-hook + OIDC + workload-identity requirements up front (OQ-A23-5).

## 2. Ordered Task List

- **TASK-01:** Pin + validate a Kargo version (OPA-gating hooks, Keycloak OIDC, workload identity); install Helm values + manifests single-cluster, single Stage — produces: Kargo install — depends-on: [] (A7 available).
- **TASK-02:** GitOps wiring — single mainline + `envs/dev|staging|prod/` path overlays; promotion updates `envs/<stage>/` kustomization to a Warehouse commit hash; no per-env branches — produces: GitOps layout + promotion mechanic — depends-on: [TASK-01].
- **TASK-03:** Warehouse configuration — candidate commit lands on merge — produces: Warehouse config — depends-on: [TASK-02].
- **TASK-04:** Per-Stage verification step (smoke suite) gating onward promotion — produces: verification config — depends-on: [TASK-03].
- **TASK-05:** `Approval`-CRD composition — Kargo creates `Approval`, waits for Argo Workflow decision report-back — produces: approval-gate integration — depends-on: [TASK-04, B19]. (Mock B19 until landed — OQ-A23-3.)
- **TASK-06:** OPA promotion-action gating — contribute promotion-action Rego to B16 bundle (who/what/where/when + time-of-day) — produces: Rego + gating wiring — depends-on: [TASK-02, A7].
- **TASK-07:** Identity — Keycloak OIDC for UI/API, cluster-OIDC workload identity for controller — produces: identity config — depends-on: [TASK-01].
- **TASK-08:** Audit emission via the platform adapter for every promotion action; Knative trigger flow (emit promotion/verification lifecycle, consume `platform.approval.*`) — produces: audit + event wiring — depends-on: [TASK-04, TASK-05].
- **TASK-09:** Recovery model — promote-previous-Warehouse-commit through affected Stages (verification + approval gates re-apply); no git-revert — produces: recovery procedure — depends-on: [TASK-04, TASK-05].
- **TASK-10:** Two-kind integration test setup (dev + staging) exercising promotion without AWS — produces: CI promotion harness — depends-on: [TASK-04].
- **TASK-11:** Headlamp deep-link inventory entry per Stage; coordinate with B5 Kargo plugin — produces: deep-link config — depends-on: [TASK-01]. (B5 owns the plugin; co-land.)
- **TASK-12:** Cross-cutting — alerts, `GrafanaDashboard` XR, runbook, HolmesGPT toolset, per-product docs — depends-on: [TASK-08, TASK-09].
- **TASK-13:** 3-layer tests (Chainsaw reconcile + Approval composition, two-kind promotion, Playwright UI deep-link/promotion with B5, PyTest OPA-gating + audit) mapping all ACs — depends-on: [TASK-05, TASK-06, TASK-08, TASK-10].
- **TASK-14:** Tutorials & how-tos ("promote a change through Stages") — depends-on: [TASK-12].

## 3. Dependency Map

### 3.1 Upstream that must ship first (HARD)
- **A7 (OPA / Gatekeeper)** — promotion-action gating engine + decision path.
- **B19 (B19-core)** — `Approval` CRD + Argo Workflow integration Kargo composes with for human gates (lands W3 ahead of A23/W4 per the cycle resolution).

### 3.2 Downstream blocked on this
- **B5** — the Kargo Headlamp plugin (status viz + promotion controls + deep-link) built against A23's Kargo install/API; co-lands in W4.

### 3.3 Continuous (non-blocking) inputs
- **B14** test framework (Chainsaw/Playwright/PyTest + two-kind harness). **B22** security threat model (promotion path is privileged). **A18** audit adapter (emission). **B4** Crossplane Compositions (substrate transparency). **B16** OPA bundle (promotion-action Rego target). **B9** `agent-platform promote` CLI (drives Kargo — OQ-A23-4). **B12** CloudEvent registry (promotion-event namespace — OQ-A23-2).

## 4. Parallelizable Subtasks

- After TASK-02/03/04 (the Stage/Warehouse/verification core): TASK-05 (approval), TASK-06 (OPA), TASK-07 (identity), TASK-08 (audit), TASK-10 (two-kind), TASK-11 (deep-link) are largely independent integration fan-out.
- TASK-12 (cross-cutting) and TASK-14 (docs) parallelize once their inputs exist.

## 5. Test Strategy

- **Chainsaw (operator/CRD):** AC-A23-03 (Warehouse), AC-A23-04 (verification gate), AC-A23-05 (Approval composition + report-back), AC-A23-09 (substrate-transparent XR promotion), AC-A23-11 (two-kind dev→staging promotion).
- **Playwright (UI/e2e):** AC-A23-07 (Keycloak OIDC login), AC-A23-12 (Headlamp deep-link per Stage) — coordinated with B5's plugin.
- **PyTest (logic):** AC-A23-06 (OPA promotion-action gating incl. time-of-day), AC-A23-10 (audit emission), AC-A23-08 (recovery promote-previous-commit logic), AC-A23-13 (post-Warehouse gate ordering), AC-A23-14 (overlay = sizing/replicas only assertion).
- **Config-level:** AC-A23-01 (single Stage + additive second Stage), AC-A23-02 (path overlay mutation, no branch).
- **Fakes/fixtures:** mock B19 create-Approval/report-back (OQ-A23-3); mock OPA decision until B16 Rego lands; mock Keycloak/cluster-OIDC; two-kind CI fixture; stub B5 plugin for deep-link e2e. Replace with real surfaces when landed and re-run.

## 6. PR / Branch Mapping

### 6.1 Stack position — base branch = `wave/4` (contains B19-core from W3 + A7 from W0; B5 co-lands in W4).
### 6.2 PR — `piece/A23-kargo-promotion-fabric` → base `wave/4`; carries SPEC-A23 + PLAN-A23.
### 6.3 Merge order — co-land with B5 (Kargo plugin) within W4; A23 install/API merges before/with B5's plugin; wave/4 rolls up to main. B19-core must already be in the base.

## 7. Effort Estimate

- Install + GitOps + Warehouse + verification (TASK-01..04): M (config/wrap; version validation is the early risk).
- Compositions (approval/OPA/identity/audit/recovery — TASK-05..09): L (the integration surface is the bulk).
- Two-kind + cross-cutting + tests + docs (TASK-10..14): M.
- **Rollup: L** (matches CSV). **Critical path:** TASK-01 → 02 → 04 → 05 (approval composition) → 13 (tests).

## 8. Rollback / Reversibility

Back out by uninstalling Kargo and reverting the GitOps wiring; within-environment deploy reverts to PR + ArgoCD sync (the pre-Kargo baseline), and promotion returns to branch-and-path conventions + manual handoff. Because promotion mutates `envs/<stage>/` kustomizations on the append-only mainline, already-promoted state persists in Git and continues to reconcile via ArgoCD — uninstalling Kargo does not roll back deployed environments. **Downstream break:** B5's Kargo plugin loses its backend; the per-Stage deep-links 404. The `Approval` CRD (B19) and OPA bundle (B16) are unaffected (Kargo only composed with them). Recovery during normal operation is never `git revert` — it is promote-a-previous-Warehouse-commit; only a full component back-out removes Kargo itself.
