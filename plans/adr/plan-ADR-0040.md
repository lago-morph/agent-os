# PLAN ADR-0040 — Kargo as the v1.0 promotion fabric for GitOps environments `[PROPOSED]`

> spec: SPEC-ADR-0040 · kind: ADR · tier: T0
> wave: authoring-parallel · estimate: M
> upstream-pieces: [A7; B19; B4; A9] · downstream-pieces: [A23; B5; A18]

## 1. Implementation Strategy
Enforcement/verification map. ADR 0040 is enforced at **A23** (the Kargo install component: Helm values,
GitOps wiring, per-Stage verification configs, single-Stage→multi-Stage phasing) and at **B5** (the
Kargo Headlamp plugin). The composition contracts are honored by reusing existing primitives, not
re-inventing: the `Approval` CRD gate (B19), OPA promotion-action policies (A7/B16), Keycloak OIDC
(ADR 0028), the audit adapter (ADR 0034), and Crossplane claim shapes (B4/ADR 0041). The load-bearing
recovery model (re-promote a previous Warehouse commit, not git-revert) and the substrate-invisible
promotion property are each verified by targeted tests; the two-kind CI setup proves promotion mechanics
without AWS.

## 2. Ordered Task List
- TASK-01: Stand up Kargo single-cluster, one Stage; verify adding a second environment adds a second Stage — produces: Chainsaw/install test — depends-on: []
- TASK-02: Verify single-mainline + `envs/<stage>/` layout, no per-environment branches; merge→Warehouse→`envs/<stage>/` commit-hash update — produces: integration promotion test — depends-on: [TASK-01]
- TASK-03: Verify per-Stage verification (smoke suite) failure blocks onward promotion — produces: integration gate test — depends-on: [TASK-02]
- TASK-04: Verify approval-gated Stage creates an `Approval` CRD and proceeds only on decision report-back — produces: Chainsaw approval-composition test — depends-on: [TASK-02]
- TASK-05: Verify OPA promotion-action gating (incl. time-of-day window) blocks/allows promotion — produces: integration OPA-gate test — depends-on: [TASK-02]
- TASK-06: Verify Keycloak OIDC auth for UI + cluster-OIDC workload identity for the controller — produces: integration auth test — depends-on: [TASK-01]
- TASK-07: Verify same claim shape promotes across kind→AWS with no Kargo-visible substrate difference — produces: integration cross-substrate test — depends-on: [TASK-02]
- TASK-08: Verify recovery via re-promoting a previous Warehouse commit; git history append-only — produces: integration recovery test — depends-on: [TASK-02]
- TASK-09: Verify promotion actions emit audit via the platform adapter — produces: PyTest audit test — depends-on: [TASK-02]
- TASK-10: Stand up the two-kind (dev+staging) CI setup exercising promotion without AWS — produces: CI promotion harness — depends-on: [TASK-01]

## 3. Dependency Map
### 3.1 Upstream pieces that must ship first (HARD)
- A7 — OPA (promotion-action gating). B19 — `Approval` CRD + Argo Workflow (human gate). B4 — Crossplane Compositions (substrate uniformity). A9 — Headlamp (deep-links).
### 3.2 Downstream pieces blocked on this
- A23 (Kargo install build), B5 (Kargo Headlamp plugin), A18 (promotion-action audit path).
### 3.3 Continuous (non-blocking) inputs
- B16 OPA policy content (promotion-action Rego); ADR 0028 identity; ADR 0039 authoring surface; ADR 0041 Compositions; B14 test framework.

## 4. Parallelizable Subtasks
TASK-01, TASK-06, TASK-10 run early/concurrently. After TASK-02: TASK-03, TASK-04, TASK-05, TASK-07, TASK-08, TASK-09 fan out independently.

## 5. Test Strategy
- AC-01 → Chainsaw/install single-Stage + add-Stage (TASK-01).
- AC-02, AC-03 → integration layout + Warehouse promotion (TASK-02).
- AC-04 → integration smoke-gate (TASK-03).
- AC-05 → Chainsaw approval-composition (TASK-04); fake B19 `Approval` fixture until B19 lands.
- AC-06 → integration OPA time-of-day gate (TASK-05); fake B16 policy until it lands.
- AC-07 → integration Keycloak/cluster-OIDC auth (TASK-06).
- AC-08 → integration cross-substrate claim promotion (TASK-07); fake B4 Compositions until B4 lands.
- AC-09 → integration recovery re-promotion (TASK-08).
- AC-10 → PyTest audit-event presence (TASK-09).
- AC-11 → two-kind CI promotion harness (TASK-10).

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/authoring`
### 6.2 PR — `piece/ADR-0040-kargo-promotion` → base `wave/authoring`; carries spec + plan.
### 6.3 Merge order — independent of sibling ADR specs; rolls up to main.

## 7. Effort Estimate
TASK-01 S · TASK-02 M · TASK-03 S · TASK-04 S · TASK-05 S · TASK-06 S · TASK-07 M · TASK-08 S · TASK-09 S · TASK-10 M. Rollup M. Critical path: TASK-01 → TASK-02 → TASK-07 (cross-substrate promotion).

## 8. Rollback / Reversibility
Reverting backs out the Kargo install + plugin; promotion reverts to branch/path conventions + ad-hoc
handoff, losing the promotion system-of-record and audit trail. Because recovery is re-promotion (git
history append-only), backing out Kargo does not corrupt git state — the `envs/<stage>/` overlays remain
valid manifests ArgoCD can sync directly. No data migration; the Warehouse promotion records are lost as
a queryable handle but the deployed state is intact.
