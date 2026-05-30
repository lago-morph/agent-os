# PLAN ADR-0032 — CapabilitySet overlay semantics `[PROPOSED]`

> spec: SPEC-ADR-0032 · kind: ADR · tier: T1
> wave: authoring-parallel · estimate: S
> upstream-pieces: [A1; B13] · downstream-pieces: [B13; A7; A22; A20; A18]

## 1. Implementation Strategy
Enforcement/verification map. ADR 0032 is enforced primarily at **B13** (the kopf operator),
which owns the single resolution algorithm that computes the effective `CapabilitySet`; all other
consumers (A7 admission, A22 effective-set rendering, A20 simulator, A18 audit) MUST read B13's
resolved output rather than re-derive it. Conformance is asserted with golden-fixture tests that
pin the overlay rule (replace-not-merge, declared order, overrides-last) and a cross-consumer
determinism test that proves identical inputs yield identical resolved sets.

## 2. Ordered Task List
- TASK-01: Encode overlay-rule golden fixtures (stacked sets, list-replace, order-sensitivity, override-last) — produces: conformance fixture set — depends-on: []
- TASK-02: Assert B13 resolver matches goldens — produces: PyTest resolver conformance test — depends-on: [TASK-01]
- TASK-03: Assert A7/A22 consume B13's resolved output (no divergent merge) — produces: cross-consumer determinism test — depends-on: [TASK-02]
- TASK-04: Negative test — overlay-rule change without ADR 0030 bump fails review gate — produces: versioning-guard test — depends-on: [TASK-01]

## 3. Dependency Map
### 3.1 Upstream pieces that must ship first (HARD)
- A1 — LiteLLM (target the resolved set reconciles into).
- B13 — kopf operator (owns the `CapabilitySet` CRD + resolution algorithm).
### 3.2 Downstream pieces blocked on this
- B13 (reconcile), A7 (admission), A22 (editor effective-set view), A20 (simulator), A18 (audit).
### 3.3 Continuous (non-blocking) inputs
- B14 test framework (fixture harness); design-time resolution of architecture-backlog §1.1
  sub-decisions (recursion, override-grant rule).

## 4. Parallelizable Subtasks
TASK-03 and TASK-04 run concurrently once TASK-01/02 land.

## 5. Test Strategy
- AC-01..04 → PyTest resolver conformance against goldens (TASK-01/02).
- AC-05 → cross-consumer determinism test; fake A7/A22 resolved-set readers until those land (TASK-03).
- AC-06 → versioning-guard / review-gate negative test (TASK-04).
- Chainsaw: a CRD-level test applying stacked `CapabilitySet`s + `Agent` and asserting resolved status.

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/authoring`
### 6.2 PR — `piece/ADR-0032-capabilityset-overlay` → base `wave/authoring`; carries spec + plan.
### 6.3 Merge order — independent of sibling ADR specs; rolls up to main.

## 7. Effort Estimate
TASK-01 S · TASK-02 S · TASK-03 S · TASK-04 S. Rollup S. Critical path: TASK-01 → TASK-02 → TASK-03.

## 8. Rollback / Reversibility
Reverting the conformance fixtures re-permits divergent/additive merges across consumers, breaking
the determinism guarantee that audit and security review depend on. No data migration — artifacts
are fixtures plus tests; the resolution algorithm itself lives in B13.
