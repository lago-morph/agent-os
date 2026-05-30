# PLAN A9 — Headlamp install + framework

> spec: SPEC-A9 · kind: COMPONENT · tier: T1
> wave: W0 · estimate: L
> upstream-pieces: [] · downstream-pieces: [A21, A22, B1, B19]

## 1. Implementation Strategy
Install Headlamp via Helm with platform branding, then build the cross-cutting **framework**:
auth-context provider exposing the Platform JWT claims, the three ADR 0039 shared widgets
(reference picker, diff-against-Git viewer, simulator-panel host), a submit-to-Git helper enforcing
the no-cluster-writes rule, and plugin scaffolding + an integration contract. The discipline is to
draw the A9/A22/B5 boundary deliberately: A9 ships **widgets and plumbing**, never a per-component
plugin. Lock the framework API contract early because A21, A22, B1, and B19 all bind to it. Build
against a mocked A20 simulator and validate with a throwaway sample plugin (which does not ship).

## 2. Ordered Task List
- **TASK-01:** Helm values + branding + ArgoCD application for Headlamp — produces: install manifests — depends-on: [].
- **TASK-02:** Keycloak SSO + auth-context provider exposing Platform JWT claims — produces: auth handoff — depends-on: [TASK-01].
- **TASK-03:** Shared widget library — reference picker, diff-against-Git viewer, simulator-panel host — produces: widget libraries — depends-on: [TASK-02].
- **TASK-04:** Submit-to-Git helper enforcing PR-only (no cluster writes) — produces: submit helper — depends-on: [TASK-02].
- **TASK-05:** Plugin scaffolding + integration contract + extender docs — produces: scaffolding + contract — depends-on: [TASK-03, TASK-04].
- **TASK-06:** Served-schema rendering (OpenAPI version handling) — produces: schema-aware renderer — depends-on: [TASK-03].
- **TASK-07:** Audit emission of plugin/admin actions + `LogLevel` honoring — produces: audit + toggle wiring — depends-on: [TASK-01].
- **TASK-08:** OPA admission Rego for Headlamp workloads + plugin-action gating convention — produces: Rego + gating convention — depends-on: [TASK-02].
- **TASK-09:** `GrafanaDashboard` XR + availability alerts — produces: dashboard + alerts — depends-on: [TASK-01].
- **TASK-10:** Sample throwaway plugin to validate the framework (not shipped) — produces: validation plugin — depends-on: [TASK-05].
- **TASK-11:** Three-layer tests + docs/runbook/tutorials/extender guide — produces: tests + docs — depends-on: [TASK-04, TASK-05, TASK-06, TASK-07, TASK-09, TASK-10].

## 3. Dependency Map
### 3.1 Upstream pieces that must ship first (HARD)
- None (W0). Binds to Keycloak (baseline) and the Kubernetes API.
### 3.2 Downstream pieces blocked on this
- A22 (editor framework), A21 (`TenantOnboarding` editor), B1 (SSO integration), B19 (approval-queue
  UI), plus B5 cross-cutting plugins and every per-component plugin.
### 3.3 Continuous (non-blocking) inputs
- A20 policy-simulator service (mock until landed — panel host needs only its response shape);
  B14 test framework; B22 threat-model standards.

## 4. Parallelizable Subtasks
- After TASK-02: TASK-03 and TASK-04 run concurrently; TASK-07, TASK-08, TASK-09 also parallel.
- TASK-05 joins TASK-03+TASK-04; TASK-06 depends only on TASK-03.
- TASK-10 follows TASK-05.

## 5. Test Strategy
- **Chainsaw:** AC-A9-01 (install), AC-A9-04 (no-cluster-write verification), AC-A9-08 (LogLevel), AC-A9-09 (XR/alert).
- **Playwright:** AC-A9-02 (SSO + claims), AC-A9-03 (widgets render), AC-A9-05 (sample plugin loads), AC-A9-10 (simulator panel).
- **PyTest:** AC-A9-04 (submit-to-Git opens PR), AC-A9-06 (served-schema rendering), AC-A9-07 (audit emission), logic half of AC-A9-02.
- **Fixtures/fakes:** mock A20 simulator response; fake Git remote for PR submission; mock Keycloak
  + crafted JWTs with varying claims; CRDs served at `v1alpha1` then `v1` for the schema test.

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/0`.
### 6.2 PR — `piece/A9-headlamp-framework` → base `wave/0`; carries spec-A9.md + plan-A9.md.
### 6.3 Merge order — independent of W0 siblings; wave/0 rolls up to main. A22/A21/B5/B19 stack on the framework API once merged.

## 7. Effort Estimate
- TASK-01 S · TASK-02 M · TASK-03 L · TASK-04 M · TASK-05 M · TASK-06 M · TASK-07 S · TASK-08 S · TASK-09 S · TASK-10 S · TASK-11 L.
- Rollup: **L**. Critical path: TASK-01 → TASK-02 → TASK-03 → TASK-05 → TASK-11 (the shared-widget +
  scaffolding spine that downstream pieces consume).

## 8. Rollback / Reversibility
Remove the ArgoCD application. A9 holds no platform state, so rollback is clean. Downstream impact
is broad: A22 editors, B5 plugins, A21's editor, and B19's approval-queue UI have no framework to
load into; admin/dev navigation falls back to raw `kubectl`/vendor consoles. No data loss — Git
remains the source of truth, ArgoCD keeps reconciling.
