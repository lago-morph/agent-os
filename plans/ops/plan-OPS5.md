# PLAN OPS5 — Day-2 Operations architecture

> spec: SPEC-OPS5 · kind: COMPONENT (cross-cutting NFR layer) · tier: T1
> wave: post-W4 (operational/NFR layer; authoring-parallel, real-realization after the F-series) · estimate: M
> upstream-pieces: [A23, B9, B4, A5, A6, B13, B19, A14, A13, A18, A7, A4, D1, F1, F2, F4, F5, F6] · downstream-pieces: []

## 1. Implementation Strategy

OPS5 is realized **not by building a service** but by (a) writing the Day-2 operating-model document + the REQ-OPS5-22 cross-reference matrix, (b) mapping each SPEC obligation onto the mechanism that already realizes or must realize it (ArgoCD/Kargo, the ADR-0030 versioning discipline, D1 dashboards, AlertManager→HolmesGPT, the B9/B14 test framework, the F-series readiness outputs), and (c) adding the **conformance/upgrade/migration verification cases** that prove each obligation, authored against the existing three-layer framework (ADR 0011) so no bespoke tooling is introduced. Because the SPEC is cross-cutting, the plan's center of gravity is the **dependency + verification map**: every REQ is realized by an owning component and verified by a named dashboard / GitOps-or-Argo mechanism / conformance-or-upgrade test. Genuinely open decisions (CRD migration mechanics, SLO targets/ownership, Crossplane `Operation`, recurring-health XRD, ArgoCD auto-heal stance) are surfaced as PROPOSED-ADRs in SPEC §10 and are **not** resolved here — the plan tasks call them out as gates, not as work to silently complete.

## 2. Ordered Task List

`TASK-NN: <description> — produces: <artifact> — depends-on: [refs]`

- **TASK-01:** Author the **Day-2 operating-model document** (upgrade, migration, observability/on-call, incident response, drift, capacity/lifecycle) binding each section to Canon names + ADRs. — produces: the OPS5 operating-model doc (SPEC §6 narrative form) — depends-on: [] (SPEC complete).
- **TASK-02:** Build the **REQ-OPS5-22 cross-reference matrix** (obligation → owning component spec → realizing mechanism → verifying test/dashboard). — produces: the OPS5 ownership matrix — depends-on: [TASK-01].
- **TASK-03:** Specify the **upgrade operating model** on top of A23/Kargo + ADR-0010 pre-merge checks: pinned-version-bump → Warehouse → per-Stage smoke gate → promote; staged-restart (ADR 0035) assumption documented. — produces: upgrade runbook spec + Stage-smoke obligations — depends-on: [TASK-01].
- **TASK-04:** Specify the **CRD/XRD migration operating model**: the ADR-0030 procedure as a Day-2 runbook + safety gates (new `vN`, conversion-webhook test, stored-version migration, `vN-1` deprecation, both substrate Compositions per ADR 0041), with the open mechanics flagged to the migration-strategy PROPOSED-ADR. — produces: migration runbook spec + gate list — depends-on: [TASK-01]; gated-by: PROPOSED-ADR (CRD migration strategy).
- **TASK-05:** Specify the **drift detection/remediation model**: ArgoCD source-of-truth + reconciliation, Crossplane Composition drift (B4), Kargo promote-previous-Warehouse-commit recovery; flag the auto-heal-vs-detect stance to its PROPOSED-ADR. — produces: drift/recovery runbook spec — depends-on: [TASK-01]; gated-by: PROPOSED-ADR (ArgoCD drift stance).
- **TASK-06:** Specify the **observability/on-call + capacity-watch model** against D1 dashboards + F5 baselines + ADR-0035 `LogLevel`; record the no-SLO-in-v1.0 posture and route SLO targets/ownership to its PROPOSED-ADR. — produces: observability/capacity obligations + alert→runbook rule — depends-on: [TASK-01]; gated-by: PROPOSED-ADR (SLO targets/ownership).
- **TASK-07:** Specify the **incident-response model**: AlertManager→HolmesGPT triage (A14), three-state OPA-gated + `Approval`-gated remediation, audit-system-of-record forensics, and the cross-cutting incident-runbook set OPS5 requires F6 to compile/exercise. — produces: incident-flow spec + required-runbook list for F6 — depends-on: [TASK-01].
- **TASK-08:** Author the **verification map** (PLAN §5): bind every AC to a Chainsaw/Playwright/PyTest case or a GitOps/Argo/Kargo mechanism, naming fixtures for not-yet-landed gates. — produces: the AC→verification table — depends-on: [TASK-03, TASK-04, TASK-05, TASK-06, TASK-07].
- **TASK-09:** Author/queue the **conformance + upgrade + migration test cases** in the B14 framework (Stage smoke reuse, conversion-webhook test, drift-reconcile test, `LogLevel` restore test, forensics-without-OpenSearch test). — produces: OPS5 conformance test set (in B14) — depends-on: [TASK-08].
- **TASK-10:** Finalize the **PROPOSED-ADR title list** with one-line rationales for the user (do not author). — produces: SPEC §10.1 list (already in SPEC) confirmed + linked from the plan — depends-on: [TASK-04, TASK-05, TASK-06].

## 3. Dependency Map

### 3.1 Upstream pieces that must ship first (HARD) — id + what is consumed

- **A23 (Kargo, W4)** — the promotion/recovery control plane the upgrade + rollback model rides (Stage verification, Warehouse-commit rollback, OPA-gated promotion). **Hard** for REQ-OPS5-01/02/05/19.
- **B4 (Crossplane Compositions, W1)** — XRD/Composition migration (both substrates, ADR 0041) + Composition drift reconciliation. **Hard** for REQ-OPS5-04/18.
- **A5 / A6 / B13 / B19 (W1/W3)** — the reconcilers owning the CRDs migrated; B19 owns `Approval` used for gated Day-2 actions. **Hard** for REQ-OPS5-03/06.
- **A14 (HolmesGPT, W0→rewired)** — incident-triage agent + AlertManager→HolmesGPT flow + three-state remediation. **Hard** for REQ-OPS5-13/14.
- **A13 (Tempo+Mimir, W0) / A18 (audit, W1) / D1 (dashboards, consumer)** — observability + audit system-of-record + the operator dashboard surface. **Hard** for REQ-OPS5-09/16.
- **A7 (OPA, W0) / B9 (CLI, W2) / B14 (test framework, W3)** — remediation/promotion gating; the operator entry points (`promote`, `--debug`/`--trace`); the verification harness. **Hard** for REQ-OPS5-06/07/12.
- **A4 (broker, W0)** — broker-backlog capacity signal + incident class. **Hard** for REQ-OPS5-15/20.
- **F1/F2/F4/F5/F6 (consumer band)** — retention, DR testing, security review, scale baselines, runbook compilation: the outputs OPS5's operating model consumes/folds in. **Hard** for REQ-OPS5-11/15/20.
- **ADR 0030 / 0041 / 0040 / 0035 / 0034 / 0012 / 0011 / 0010 / 0026** — the policy/architecture constraints OPS5 cannot contradict (versioning, substrate migration, promotion, restart, audit, self-management, testing, pre-merge, single-cluster).

### 3.2 Downstream pieces blocked on this — ids

- **None as a build edge** (OPS5 is a terminal NFR layer; no piece's build waits on it).
- **Operational absorption obligations** (the REQ-OPS5-22 matrix — these specs *must absorb* an OPS5 obligation but are not "blocked"; OPS5 cannot edit them per its constraint, so the matrix flags any unabsorbed obligation as a gap):
  - **A23** — own the upgrade/rollback Stage-smoke + recovery procedure (REQ-OPS5-01/05/19).
  - **A5/A6/B13/B19/B4** — own their CRD/XRD migration execution + conversion-webhook tests (REQ-OPS5-03/04).
  - **D1** — surface the Capacity baselines + the no-SLO posture; provide the Day-2 operator surface (REQ-OPS5-09/11/20).
  - **A14 + per-component toolsets** — own the triage + remediation + `LogLevel` driver (REQ-OPS5-12/13/14).
  - **F6** — compile + exercise-verify the OPS5-required cross-cutting incident runbooks (REQ-OPS5-15).
  - **B16/A7** — own the promotion-action + remediation Rego the gating relies on (REQ-OPS5-06).

### 3.3 Continuous (non-blocking) inputs

- **B14 (test framework) → all** and **B22 (security threat model) → all** — continuously-available inputs (waves.md fan-out edges), not wave-depth-setting; OPS5's conformance tests are authored against B14, and the incident-response model honors the B22 threat model.
- **B12 (CloudEvent schema registry)** — non-blocking: any "migration/drift" lifecycle event (SPEC OQ1) would register here if minted; OPS5 mints none.

## 4. Parallelizable Subtasks

- After **TASK-01/02** (operating-model doc + matrix), the five domain specs run **concurrently**: **TASK-03 (upgrade)**, **TASK-04 (migration)**, **TASK-05 (drift)**, **TASK-06 (observability/capacity)**, **TASK-07 (incident)** are an independent fan-out group (different mechanisms, different owners).
- **TASK-08 (verification map)** joins them (fan-in), then **TASK-09 (test cases)** fans out again per layer (Chainsaw vs Playwright vs PyTest cases are independent).
- **TASK-10 (PROPOSED-ADR list)** can finalize as soon as TASK-04/05/06 surface their open decisions — independent of TASK-08/09.

## 5. Test Strategy

Maps SPEC §9 ACs to layers; names the GitOps/Argo/Kargo mechanism or dashboard where the "test" is a runtime gate rather than code. Fixtures noted for not-yet-landed gates.

| AC | Layer / mechanism | What it asserts | Fixture if upstream not landed |
|---|---|---|---|
| AC-OPS5-01 | **Kargo Stage smoke (Chainsaw)** + **GitHub Actions pre-merge** + ArgoCD drift check | upgrade promotes only on passing per-Stage smoke; no Git↔running drift | two-kind setup (A23 REQ-A23-11) as dev+staging; stub vendor-chart bump |
| AC-OPS5-02 | **Chainsaw conversion-webhook test** | new `vN` group converts `vN-1`; `vN-1` deprecated not removed | a throwaway test CRD `vN→vN+1` if no real migration yet (`[PROPOSED]`) |
| AC-OPS5-03 | **Chainsaw, two-substrate** | conversion webhooks + deprecation on both kind & AWS Compositions; secret-shape unbroken | kind Composition only if AWS unavailable; label-skip AWS leg |
| AC-OPS5-04 | **Kargo promote-previous-commit (Chainsaw/Playwright)** | rollback re-runs verification+approval; no `git revert`; append-only history | Warehouse fixture with ≥2 candidate commits |
| AC-OPS5-05 | **PyTest (OPA decision) + Chainsaw (`Approval`)** | gated action creates `Approval`, proceeds on `decision`; denied action blocked | B19-core `Approval` fixture; OPA bundle stub for promotion-action policy |
| AC-OPS5-06 | **Repo/CI inspection** | Stage smoke + webhook tests are B14 cases, no bespoke harness | n/a |
| AC-OPS5-07 | **Chainsaw + load-drain (reuses F5 AC-F5-07)** | staged rolling restart at `replicas>=2`, SSE-safe; no hot-reload assumption | reuse F5 drain probe |
| AC-OPS5-08 | **Playwright (D1) + inspection** | six D1 dashboards are the surface; Capacity renders F5 baselines; no SLO panel; no parallel stack | D1 dashboard XRs; F5 baseline metrics stub if F5 late |
| AC-OPS5-09 | **Doc/link-check (via F6 matrix)** | every alert names a signal + runbook; orphan alert flagged | F6 verification-matrix fixture |
| AC-OPS5-10 | **Chainsaw/PyTest (kill injection)** | `LogLevel`/`--trace` raises scoped verbosity, restores on expiry/exit | reuse B9 AC-B9-05 toggle-restore harness |
| AC-OPS5-11 | **Chainsaw + PyTest** | AlertManager CloudEvent → Trigger → HolmesGPT finding; non-graduated remediation = auto-PR/card + `platform.audit.*` | A14 trigger-flow fixture; AlertManager source stub (ADR 0023) |
| AC-OPS5-12 | **F6 verification matrix + KB retrieval** | the six incident runbooks exist, exercise-verified, KB-retrievable | F6/C8 KB index fixture |
| AC-OPS5-13 | **PyTest/integration** | forensic query answered from Postgres+S3 with OpenSearch down | audit system-of-record seed; OpenSearch-down toggle |
| AC-OPS5-14 | **Chainsaw** | ArgoCD reconciles out-of-band change; Composition reconciles drifted substrate resource | ArgoCD app fixture; B4 Composition fixture |
| AC-OPS5-15 | **Playwright/inspection** | baseline regression surfaces on Capacity dashboard + routes to owner/runbook; lifecycle posture lists single-instance set + LiteLLM carve-out | F5 baseline + D1 Capacity panel |
| AC-OPS5-16 | **Inspection/link-check** | cross-reference matrix maps every §6 obligation to an owner; none unowned | n/a |

**Cross-layer notes:** OPS5 authors **no new test runner** (ADR 0011); all cases are Chainsaw (operator/CRD/migration/drift), Playwright (dashboard/UI/rollback e2e), or PyTest (OPA-gating, toggle-restore, forensics logic), orchestrated by B9/B14. Where a gate is a *runtime* GitOps mechanism (ArgoCD reconcile, Kargo Stage smoke) rather than a unit test, the table names the mechanism. Several ACs **reuse** existing component ACs (F5 drain, B9 toggle-restore, A23 two-kind, A14 trigger flow) rather than duplicate them — OPS5 asserts the *Day-2 composition*, not the component internals.

## 6. PR / Branch Mapping

### 6.1 Stack position — base branch

- OPS5 is a **post-W4 cross-cutting operational/NFR layer** (pending-operational-nfr-layer.md: runs AFTER the 119-piece spec/plan work, verification, consistency sweep, and PR graph are complete). Base branch = the integration branch that already contains all W0–W4 component specs + the consumer-band (C/D/E/F) and ADR/View specs — i.e. the post-W4 mainline (`main` after the component graph rolls up), **not** a `wave/<N>` branch.
- Rationale: OPS5 references A23 (W4), B9 (W2), B4 (W1), the F-series (consumer), and D1 (consumer); it can only be authored once those specs exist, so it stacks on top of the fully-merged component graph.

### 6.2 PR

- One PR: **`piece/OPS5-day2-operations`** → base post-W4 mainline. Carries exactly the two new files (`specs/ops/spec-OPS5.md`, `plans/ops/plan-OPS5.md`). No edits to existing specs/plans/source (SPEC constraint).
- PR body: links the SPEC §10.1 PROPOSED-ADR titles for the user to triage; notes the REQ-OPS5-22 cross-reference matrix as the absorption-obligation surface (component specs *should* later absorb their obligations in their own PRs — out of scope for this PR).

### 6.3 Merge order

- OPS5 is **one of six sibling OPS PRs** (OPS1–OPS6), each its own stacked PR per pending-operational-nfr-layer.md. The six are **independent of each other** (no OPS→OPS build edge) and can merge in any order; each rolls up to the post-W4 mainline.
- Within the OPS layer, OPS5 shares cross-links with OPS1 (scale/cost — capacity overlap), OPS2 (DR — restore/incident overlap), OPS4 (compute isolation — chokepoint/blast-radius overlap), and OPS6 (policy lifecycle — OPA-gating overlap). These are **reference** cross-links only, not merge dependencies; if a sibling lands first, OPS5's references resolve; if not, they are forward references (consistent with how the component specs cross-reference).

## 7. Effort Estimate

- TASK-01 (operating-model doc): **M** · TASK-02 (matrix): **S** · TASK-03 (upgrade): **S** · TASK-04 (migration): **M** (open mechanics) · TASK-05 (drift): **S** · TASK-06 (observability/capacity): **S** · TASK-07 (incident): **S** · TASK-08 (verification map): **M** · TASK-09 (test cases queued into B14): **M** · TASK-10 (PROPOSED-ADR list): **S**.
- **Rollup: M** (matches the piece estimate). OPS5 is authoring + mapping + verification-case work, not service build; the M weight is concentrated in TASK-04 (migration mechanics, gated by an open ADR) and TASK-08/09 (the verification map + conformance cases).
- **Critical path within the piece:** TASK-01 → (TASK-04 ∥ TASK-03/05/06/07) → TASK-08 → TASK-09. TASK-04 is the longest pole because its mechanics depend on the CRD-migration-strategy PROPOSED-ADR; until that lands, TASK-04 ships the procedure + gates with the open mechanics explicitly flagged (no silent decision).

## 8. Rollback / Reversibility

- **Backing out OPS5** = delete the two files (`specs/ops/spec-OPS5.md`, `plans/ops/plan-OPS5.md`) / revert the single PR. Because OPS5 edits no existing spec/plan/source and has **no downstream build edge**, reverting it breaks **nothing** in the component graph — it removes the Day-2 operating-model document + the cross-reference matrix + the queued OPS5 conformance test cases.
- **What is lost on revert:** the explicit Day-2 operating model (upgrade/migration/drift/incident/capacity), the obligation-to-owner matrix (so absorption gaps go un-flagged), and the surfaced PROPOSED-ADR titles (the open decisions become invisible again — the original problem the pending-NFR-layer exists to fix). The realizing mechanisms themselves (ArgoCD, Kargo, ADR-0030 discipline, D1, HolmesGPT, F-series) are untouched — they are owned by their components, not by OPS5.
- **Reversibility within the piece:** each of the five domain sub-specs (TASK-03..07) and each conformance test (TASK-09) is independently revertible; the verification map (TASK-08) degrades gracefully if a single AC's mechanism is pulled. No state migration, no data change — OPS5 owns no CRD, no datastore, no connection secret.
