# PLAN ADR-0001 — ARK as the agent operator [PROPOSED]

> spec: SPEC-ADR-0001 · kind: ADR · tier: T1
> wave: authoring-parallel · estimate: S
> upstream-pieces: [A6] · downstream-pieces: [A5;B7;B9;B10;B16;B17;B18;B20]

## 1. Implementation Strategy
Enforcement/verification map. ADR 0001 is enforced by component A5 (ARK install + the seven CRD reconcilers as the only agent-control path), with admission/isolation enforced by A7 (Gatekeeper) and visibility by A9 (Headlamp). Conformance is proven by: (a) an admission policy plus CI check that no agent runs outside an ARK `Agent`; (b) a version-pin drift check against the B6 compatibility matrix; (c) a CRD-versioning conformance gate (ADR 0030). No new build work belongs to this ADR — it is a constraint map over existing components.

## 2. Ordered Task List
- TASK-01: Map each REQ to the enforcing component piece — produces: enforcement matrix — depends-on: []
- TASK-02: Specify Gatekeeper constraint(s) that reject non-ARK agent declarations + cross-namespace access — produces: Rego/constraint refs (owned by A7/B16) — depends-on: [TASK-01]
- TASK-03: Specify CI drift check: installed ARK version == SDK matrix entry — produces: CI gate spec (B15) — depends-on: [TASK-01]
- TASK-04: Specify CRD-versioning conformance gate for the seven CRDs — produces: CI gate spec (ADR 0030) — depends-on: [TASK-01]
- TASK-05: Specify Headlamp cross-tenant visibility check — produces: e2e check (A9) — depends-on: [TASK-01]

## 3. Dependency Map
### 3.1 Upstream that must ship first (HARD)
- A6 — `Sandbox`/`SandboxTemplate` ARK schedules into.
### 3.2 Downstream blocked on this
- A5, B7, B9, B10, B16, B17, B18, B20.
### 3.3 Continuous (non-blocking) inputs
- B14 test framework; B22 threat model; B15 CI pipeline; B6 SDK matrix.

## 4. Parallelizable Subtasks
TASK-02, TASK-03, TASK-04, TASK-05 fan out independently after TASK-01.

## 5. Test Strategy
- AC-01/02/03/05/06 → Chainsaw (CRD/admission/reconcile + versioning gate).
- AC-04 → PyTest (version-pin drift check in CI).
- AC-07 → Playwright (Headlamp visibility e2e).
Fixtures: fake `SandboxTemplate` until A6 lands; stub SDK matrix until B6.

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/authoring`
### 6.2 PR — `piece/ADR-0001-ark-operator` → base `wave/authoring`; carries spec + plan.
### 6.3 Merge order — independent of sibling ADR PRs; rolls up to main.

## 7. Effort Estimate
TASK-01 S · TASK-02 S · TASK-03 S · TASK-04 S · TASK-05 S. Rollup: S. Critical path: TASK-01 → any gate task.

## 8. Rollback / Reversibility
Backing out means re-opening the operator choice (kagent). Downstream breakage: every B-piece built on the ARK CRD surface. Low reversibility once agents are authored against `Agent`.
