# PLAN ADR-0033 — Initial implementation targets — AWS (EKS) and GitHub `[PROPOSED]`

> spec: SPEC-ADR-0033 · kind: ADR · tier: T1
> wave: authoring-parallel · estimate: S
> upstream-pieces: [B4] · downstream-pieces: [A18; A21; A23; B4; B15]

## 1. Implementation Strategy
Enforcement/verification map. ADR 0033 is enforced at three layers: **B4** realizes the dual-mode
hosting it mandates (one XRD + one Composition per substrate, kind + AWS), so the substrate-agnostic
contract holds; **B15** ships GitHub-Actions-only CI and is the gate that no Azure-specific pipeline
artifact ships; and the **kind OIDC bootstrap utility** makes the kind federation path real so
IRSA-vs-kind drift is caught in dev rather than on EKS. Conformance is asserted by dual-mode CI runs
and an install-artifact scope check. AKS-flag enforcement is recorded as a design-time precondition
on the deferred Azure bootstrap, not built in v1.0.

## 2. Ordered Task List
- TASK-01: Encode the v1.0 target set (AWS-EKS + GitHub + kind; no Azure) as an install/CI scope assertion — produces: scope-conformance check — depends-on: []
- TASK-02: Verify a dual-mode primitive provisions from one claim shape on kind and on AWS, consumer reads only connection-secret + agnostic status — produces: Chainsaw substrate-parity test — depends-on: [TASK-01]
- TASK-03: Verify the kind cluster-OIDC bootstrap lets Keycloak validate a projected SA token — produces: PyTest/Chainsaw federation test — depends-on: []
- TASK-04: Verify CI runs the component suite against both kind in-cluster and AWS-managed modes — produces: dual-mode CI matrix assertion — depends-on: [TASK-02]
- TASK-05: Record the AKS opt-in-flags precondition on the deferred Azure-bootstrap design note — produces: design-note gating entry — depends-on: []

## 3. Dependency Map
### 3.1 Upstream pieces that must ship first (HARD)
- B4 — Crossplane Compositions (the dual-mode hosting mechanism this ADR consumes).
### 3.2 Downstream pieces blocked on this
- A18 (audit backends), A21 (tenant onboarding), A23 (Kargo cross-substrate promotion), B15 (GitHub Actions pipeline).
### 3.3 Continuous (non-blocking) inputs
- B14 test framework (dual-mode harness); ADR 0028 realizing components (federation trust model).

## 4. Parallelizable Subtasks
TASK-03 and TASK-05 run concurrently with TASK-01/02. TASK-04 follows TASK-02.

## 5. Test Strategy
- AC-01 → scope-conformance check (TASK-01): install/CI artifact set has no Azure artifacts.
- AC-02 → Chainsaw substrate-parity test (TASK-02); fake B4 Compositions until B4 lands.
- AC-03 → PyTest/Chainsaw federation test on a kind cluster (TASK-03).
- AC-04 → CI-matrix assertion (TASK-04); requires a transient AWS cluster or recorded fixture for the managed mode.
- AC-05 → doc-presence check on the Azure-bootstrap design note (TASK-05).

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/authoring`
### 6.2 PR — `piece/ADR-0033-aws-github-targets` → base `wave/authoring`; carries spec + plan.
### 6.3 Merge order — independent of sibling ADR specs; rolls up to main.

## 7. Effort Estimate
TASK-01 S · TASK-02 S · TASK-03 S · TASK-04 S · TASK-05 S. Rollup S. Critical path: TASK-01 → TASK-02 → TASK-04.

## 8. Rollback / Reversibility
Reverting the scope-conformance gate re-permits unscoped target artifacts; the substrate-agnostic
contract and dual-mode CI lose their guarantee, risking IRSA-vs-kind drift. No data migration — the
artifacts are validation checks, the kind OIDC bootstrap utility, and tests.
