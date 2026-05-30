# PLAN V6-06 — Security and policy architecture `[PROPOSED]`

> spec: SPEC-V6-06 · kind: VIEW · tier: T0
> wave: authoring-parallel · estimate: S
> upstream-pieces: [] · downstream-pieces: []

## 1. Implementation Strategy
This view is realized, not built. The realization map follows the wave layering: **B22 (security threat-model design)** lands in W0 as a design spec that ships *before* the first A/B implementation wave and feeds standards forward into every component (ADR 0027); **A7 (OPA/Gatekeeper)** lands in W0 as the policy engine; **B3 (OPA policy framework)** lands in W1; **A18 (audit endpoint/adapter)** lands in W1 as the audit plane for every enumerated audit point; **B16 (initial OPA policy content)** lands in W2 with the concrete policies for the enumerated decision points. The view holds when defense-in-depth, RBAC-as-floor/OPA-as-restrictor, the enumerated audit + OPA decision-point sets, self-serve OPA-gated key issuance, OPA-driven budgets, and the dry-run policy simulator all hold end-to-end. Verification proves each layer independently denies excess access, RBAC bounds OPA, every enumerated hook point fires, and the simulator returns per-layer dry-runs with no enforcement side effects.

## 2. Ordered Task List
- **TASK-01:** Confirm B22 threat-model design ships before the first A/B wave and emits standards/ACs/mandatory-tests/dashboard-signals consumed by component deliverable lists (ADR 0027) — produces: threat-model-first checklist — depends-on: []
- **TASK-02:** Confirm A7 enforces defense-in-depth (Gatekeeper admission + OPA runtime) and RBAC-as-floor/OPA-as-restrictor (ADR 0002/0018) — produces: enforcement-model checklist — depends-on: []
- **TASK-03:** Enumerate and confirm every audit-emission point (§6.6) routes via the A18 adapter and every OPA decision point consults OPA over the full admission set — produces: hook-point inventory + conformance map — depends-on: [TASK-02]
- **TASK-04:** Confirm self-serve OPA-gated `VirtualKey` issuance and OPA-driven `BudgetPolicy` (reconciled into OPA data + LiteLLM tracking), Headlamp as editing surface — produces: key+budget checklist — depends-on: [TASK-03]
- **TASK-05:** Confirm the policy simulator's structured dry-run at each layer (`simulated: true`, RBAC reported distinctly, audited under `platform.policy.*`, no side effects) (ADR 0038) — produces: simulator-dry-run checklist — depends-on: [TASK-03]
- **TASK-06:** Define end-to-end view verification mapping AC-01..11 — produces: view acceptance suite mapping — depends-on: [TASK-01, TASK-04, TASK-05]

## 3. Dependency Map
### 3.1 Upstream pieces that must ship first (HARD)
- B22 — threat-model design (consumed: standards/ACs/test-cases fed forward; HARD precedence before first A/B wave).
- A7 — OPA/Gatekeeper engine (consumed: admission + runtime decisions).
- B3 — OPA policy framework (consumed: reusable policy structure).
- B16 — initial OPA policy content (consumed: concrete decision-point policies).
- A18 — audit endpoint/adapter (consumed: audit plane for every enumerated point).
### 3.2 Downstream pieces blocked on this
- None directly (view imposes constraints platform-wide; every component binds enforcement to its own spec). B22's output also updates Workstream A deliverable lists and B16 targets.
### 3.3 Continuous (non-blocking) inputs
- B22 → A, → B is a continuously-available fan-out edge (per waves.md), not wave-depth-setting; A20 (policy simulator service) and B19 (approval system) supply realizing surfaces; B14 test framework.

## 4. Parallelizable Subtasks
- TASK-01 (threat-model precedence) and TASK-02 (engine) are independent roots; fan-out group: {TASK-01, TASK-02}. TASK-04 and TASK-05 run concurrently once TASK-03 is green; fan-out group: {TASK-04, TASK-05}.

## 5. Test Strategy
- **Chainsaw (operator/CRD):** OPA decision points over the full admission CRD/XR set (AC-06); `VirtualKey` self-issuance reconcile (AC-07); `BudgetPolicy` reconcile into OPA + LiteLLM (AC-08).
- **Playwright (UI/e2e):** Headlamp self-serve key issuance gated by OPA (AC-07); budget edit in Headlamp not LiteLLM admin UI (AC-08); simulator panel per-layer output with RBAC distinct (AC-09, AC-11).
- **PyTest (logic):** defense-in-depth multi-layer deny of capability excess (AC-03); RBAC-floor cannot be widened by OPA (AC-04); enumerated audit-point emission scan (AC-05); over-budget request denied (AC-08); dry-run no-side-effect + `simulated: true` + `platform.policy.*` audit (AC-09, AC-10).
- Fixtures/fakes: stub OPA returning allow/deny + dry-run shape; fake audit endpoint capturing per-point emissions; RBAC fixture; fake LiteLLM spend tracker; B22 standards stub if not yet landed.

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/auth` (views authoring band)
### 6.2 PR — `piece/V6-06-security-and-policy-architecture` → base `wave/auth`; carries spec-V6-06.md + plan-V6-06.md
### 6.3 Merge order — independent of sibling view PRs; rolls up to main with the other V6-0x views.

## 7. Effort Estimate
- TASK-01 S · TASK-02 S · TASK-03 M · TASK-04 S · TASK-05 S · TASK-06 M. Rollup: S (authoring). Critical path: TASK-02 → TASK-03 → {TASK-04, TASK-05} → TASK-06.

## 8. Rollback / Reversibility
Reverting the view doc has no runtime effect (authoring artifact). If an invariant is found wrong, amend the SPEC and re-flag `[PROPOSED]`; the realizing component SPECs (A7/B3/B16/A18 and the B22 design spec) carry the enforceable change. Because B22 feeds standards forward, a change to the threat-model deliverable can ripple into component ACs — but reverting the *view* doc itself breaks no downstream code.
