# PLAN ADR-0038 — Policy simulators for OPA and RBAC `[PROPOSED]`

> spec: SPEC-ADR-0038 · kind: ADR · tier: T1
> wave: authoring-parallel · estimate: M
> upstream-pieces: [A7; A1; A6] · downstream-pieces: [A20; A22; A14]

## 1. Implementation Strategy
Enforcement/verification map. ADR 0038 is enforced at **A20** (the aggregator service that fans a
synthetic request out to layer-specific dry-run paths and composes cited verdicts) and at **each
decision-point component** (Gatekeeper, LiteLLM, Envoy, the approval system, Headlamp actions, the
CLI gates), which must each expose a structured dry-run as a deliverable. The two consumers — the
Headlamp panel (A22) and the HolmesGPT skill (A14) — are verified to return identical verdicts over the
same aggregator. Side-effect-freedom (no real decision, no `Approval` created) and `platform.policy.*`
audit are verified distinctly.

## 2. Ordered Task List
- TASK-01: Verify the aggregator returns a per-layer cited verdict set (admission/runtime/egress/RBAC/approval) for a synthetic request — produces: PyTest aggregator test — depends-on: []
- TASK-02: Verify every decision-point component exposes a structured dry-run; a missing one fails the deliverable check — produces: per-component dry-run conformance check — depends-on: []
- TASK-03: Verify Headlamp panel and HolmesGPT skill return identical verdicts for the same request/bundle — produces: integration parity test — depends-on: [TASK-01]
- TASK-04: Verify RBAC reported as a separate layer from OPA; denial attributed to the named layer — produces: PyTest layer-attribution test — depends-on: [TASK-01]
- TASK-05: Verify approval-bearing action returns resolved level and creates no `Approval` — produces: Chainsaw/PyTest no-side-effect test — depends-on: [TASK-01]
- TASK-06: Verify a simulator run causes no real admission/runtime/egress decision — produces: side-effect-freedom test — depends-on: [TASK-01]
- TASK-07: Verify each simulator run emits `platform.policy.*` audit — produces: PyTest audit test — depends-on: [TASK-01]

## 3. Dependency Map
### 3.1 Upstream pieces that must ship first (HARD)
- A7 — OPA/Gatekeeper (`data.allow` + audit-mode dry-run). A1 — LiteLLM (runtime dry-run). A6 — Envoy (ext-authz dry-run).
### 3.2 Downstream pieces blocked on this
- A20 (aggregator build), A22 (Headlamp simulator panel), A14 (HolmesGPT skill).
### 3.3 Continuous (non-blocking) inputs
- B16 OPA policy content; B19 approval-elevation evaluation; ADR 0039 editor inline-simulator embedding; B14 test framework; B22 threat model.

## 4. Parallelizable Subtasks
TASK-01 and TASK-02 run concurrently. After TASK-01: TASK-03, TASK-04, TASK-05, TASK-06, TASK-07 fan out independently.

## 5. Test Strategy
- AC-01 → PyTest aggregator verdict set (TASK-01); fake layer dry-runs until A1/A6/A7 paths land.
- AC-02 → per-component dry-run conformance (TASK-02, conftest over component deliverables).
- AC-03 → integration parity Headlamp vs HolmesGPT (TASK-03); fake A22/A14 fixtures until they land.
- AC-04 → PyTest layer attribution (TASK-04).
- AC-05 → Chainsaw/PyTest no-`Approval`-created (TASK-05).
- AC-06 → side-effect-freedom (TASK-06).
- AC-07 → PyTest `platform.policy.*` audit presence (TASK-07).

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/authoring`
### 6.2 PR — `piece/ADR-0038-policy-simulators` → base `wave/authoring`; carries spec + plan.
### 6.3 Merge order — independent of sibling ADR specs; rolls up to main.

## 7. Effort Estimate
TASK-01 M · TASK-02 S · TASK-03 S · TASK-04 S · TASK-05 S · TASK-06 S · TASK-07 S. Rollup M. Critical path: TASK-01 → TASK-03 (aggregator → consumer parity).

## 8. Rollback / Reversibility
Reverting backs out the aggregator, the Headlamp panel, and the HolmesGPT skill; operators and
HolmesGPT lose the "what would happen for this request" surface but enforcement is unaffected (the
simulator is read-only and side-effect-free, so nothing downstream depends on it for correctness).
The per-component dry-run obligation can stay as latent capability. No data migration.
