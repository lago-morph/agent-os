# PLAN ADR-0021 — Dashboards as namespaced Crossplane-composed GrafanaDashboard XRs `[PROPOSED]`

> spec: SPEC-ADR-0021 · kind: ADR · tier: T1
> wave: authoring-parallel · estimate: S
> upstream-pieces: [B4, A7, A9] · downstream-pieces: [D1, D2, D3, B10, A14]

## 1. Implementation Strategy
ADR 0021 is enforced, not built. The enforcement map: **B4** ships the `GrafanaDashboard` XRD + per-substrate Compositions and is the single owner; **A7 (OPA/Gatekeeper)** supplies the admission constraint and visibility restriction; **A9 (Headlamp)** surfaces XR state. Conformance is tested by proving (a) the XR shape is mandatory, (b) admission rejects malformed/raw dashboards, (c) RBAC+OPA gate visibility, (d) one claim reconciles on kind and AWS. Authoring components (D1/D2/D3, every A component, B10/A14) consume the XR; their conformance is "delivers dashboards only as `GrafanaDashboard` instances."

## 2. Ordered Task List
- TASK-01: Map each enforcing piece ID to the REQ it satisfies (B4→REQ-01/02/05/07; A7→REQ-03/04; agents B10/A14→REQ-06) — produces: enforcement matrix — depends-on: []
- TASK-02: Define Gatekeeper constraint template requiring `dashboardJson`/`folder`/`visibility` — produces: admission test spec — depends-on: [TASK-01]
- TASK-03: Define visibility OPA conformance scenarios (floor/restrict/cross-ns) — produces: OPA test spec — depends-on: [TASK-01]
- TASK-04: Define cross-substrate claim-parity scenario — produces: Chainsaw test spec — depends-on: [TASK-01]
- TASK-05: Define agent-published-dashboard governance scenario — produces: e2e test spec — depends-on: [TASK-02, TASK-03]

## 3. Dependency Map
### 3.1 Upstream pieces that must ship first (HARD)
- B4 — `GrafanaDashboard` XRD + Compositions (the artifact under test).
- A7 — Gatekeeper admission + OPA restriction engine.
### 3.2 Downstream pieces blocked on this
- D1, D2, D3, B10, A14 — must author dashboards only as XR instances.
### 3.3 Continuous (non-blocking) inputs
- B14 test framework (Chainsaw/Playwright/PyTest); B22 threat model (dashboard visibility signals).

## 4. Parallelizable Subtasks
TASK-02, TASK-03, TASK-04 are independent once TASK-01 lands.

## 5. Test Strategy
- Chainsaw: AC-01, AC-02, AC-04, AC-06 (XR mandatory, admission reject, cross-substrate parity, versioning).
- Playwright: AC-03, AC-05 (visibility gating in Headlamp UI, agent-published governance).
- PyTest: OPA decision unit tests behind AC-03.
- Fakes: stub Grafana provider until B4 lands; fake AWS Composition for kind-only CI.

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/authoring`
### 6.2 PR — `piece/ADR-0021-grafanadashboard-xrs` → base `wave/authoring`; carries spec + plan.
### 6.3 Merge order — independent of sibling ADRs; rolls up to main.

## 7. Effort Estimate
TASK-01 S; TASK-02..05 S each. Rollup S. Critical path: TASK-01 → TASK-02/03 → TASK-05.

## 8. Rollback / Reversibility
Backing out the ADR conformance suite has no runtime effect; it only removes verification. Reverting the decision itself (allowing raw JSON dashboards) would break Gatekeeper/RBAC/OPA governance of dashboards and is out of scope.
