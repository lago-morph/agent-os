# PLAN A7 — OPA / Gatekeeper

> spec: SPEC-A7 · kind: COMPONENT · tier: T0
> wave: W0 · estimate: L
> upstream-pieces: [] · downstream-pieces: [A17, A18, A20, A23, B2, B3, B19, B20]

## 1. Implementation Strategy

Install OPA and Gatekeeper via Helm, then establish the platform's single decision contract: Gatekeeper admission for the v1.0 CRD/XR set, the RBAC-as-floor / OPA-as-restrictor enforcement invariant, the budget-enforcement contract, the substrate-claim admission rule (ADR 0044), and the structured dry-run surface (audit-mode + dry-run admission) that the policy simulator (A20) composes. Rego *content* is B3/B16's job; A7 ships the engine, the dry-run/decision-shape contract, the substrate-claim constraint, and the CI Rego-test harness. Because A7 is W0 foundation with no upstreams it is built first; nearly every governed component consumes its decision API. If A7 lands before A1, A7 carries the mandatory OPA-bridge LiteLLM callback. A central early decision to lock down: **fail-open vs fail-closed** webhook default (recommend fail-closed for security constraints).

## 2. Ordered Task List

- **TASK-01:** Helm install OPA + Gatekeeper; HA webhook — produces: policy engine — depends-on: []
- **TASK-02:** Gatekeeper admission wiring for the v1.0 CRD/XR set — produces: admission integration — depends-on: [TASK-01]
- **TASK-03:** Decision-shape + single-decision-API contract (allow/deny + obligations) — produces: decision contract — depends-on: [TASK-01]
- **TASK-04:** RBAC-as-floor / OPA-as-restrictor enforcement helpers + invariant tests — produces: restrictor guarantee — depends-on: [TASK-03]
- **TASK-05:** Budget-enforcement contract (spend + BudgetPolicy → allow/deny; OPA-data shape, coordinate B13/B16) — produces: budget decision — depends-on: [TASK-03]
- **TASK-06:** Structured dry-run surface + audit-mode/dry-run-admission for A20 (`simulated: true`) — produces: simulator data sources — depends-on: [TASK-02, TASK-03]
- **TASK-07:** Substrate-XR admission constraint (reject no-matching-Composition for env label, ADR 0044) — produces: substrate guard — depends-on: [TASK-02]
- **TASK-08:** Approval-level elevation contract (elevate-never-lower, ADR 0017; coordinate B19) — produces: elevation decision — depends-on: [TASK-04]
- **TASK-09:** SHA-pinned bundle loading + CI Rego unit-test harness — produces: bundle CI — depends-on: [TASK-01]
- **TASK-10:** Admit/deny audit emission via adapter (stub until A18) + `platform.policy.*` events — produces: audit + events — depends-on: [TASK-02]
- **TASK-11:** OPA-bridge LiteLLM callback (carried here if A1 not yet landed) — produces: gateway bridge callback — depends-on: [TASK-03]
- **TASK-12:** §14.1 deliverables (docs, runbook incl. fail-policy, alerts, dashboard XR, Headlamp policy-review integration, HolmesGPT toolset, tutorials) — produces: deliverable set — depends-on: [TASK-10]
- **TASK-13:** 3-layer test suite mapping all ACs — produces: tests — depends-on: [TASK-12]

Critical path: TASK-01 → 02 → 03 → 06 → 13 (the dry-run/decision contract is what most downstreams block on).

## 3. Dependency Map

### 3.1 Upstream pieces that must ship first (HARD)
None (W0 foundation). OPA + Gatekeeper OSS + k8-platform baseline.

### 3.2 Downstream pieces blocked on this
A17, A18, A20, A23, B2, B3, B19, B20; cross-cutting A1/A6/A4 enforcement points.

### 3.3 Continuous (non-blocking) inputs
- B3 (policy framework) / B16 (policy content) — A7 ships the engine + contract; content lands later. A7 uses minimal reference Rego to test the engine.
- A18 (audit endpoint) — stub until landed.
- B13 (BudgetPolicy→OPA data) / B19 (Approval) — coordinate the OPA-data and elevation contracts.
- A20 (policy simulator) — co-design the dry-run decision shape.
- B14 test framework, B22 threat model — continuous (B22's mandatory policy targets feed A7/B16).

## 4. Parallelizable Subtasks

- After TASK-01: TASK-02 (admission) ∥ TASK-03 (decision contract) ∥ TASK-09 (bundle CI).
- After TASK-03: TASK-04 (restrictor) ∥ TASK-05 (budget) ∥ TASK-11 (bridge callback).
- After TASK-02: TASK-07 (substrate guard) ∥ TASK-10 (audit/events); TASK-06 needs both 02+03.
- TASK-08 after TASK-04.

## 5. Test Strategy

| AC | Layer | Fixtures / fakes |
|---|---|---|
| AC-A7-01,02,06,07 | Chainsaw | kind + managed-K8s; sample CRDs/XRs; mislabeled-env cluster |
| AC-A7-03,04,05,08,10,11,12 | PyTest + Rego unit | reference Rego bundles; stub RBAC identities; stub audit endpoint; stub BudgetPolicy/Approval |
| AC-A7-08 | CI (Rego unit + SHA-pin check) | tampered-bundle fixture |
| AC-A7-13 | doc check | — |
| Headlamp policy-review view | Playwright | seeded constraints; admin session |

Fakes for not-yet-landed upstreams: stub audit endpoint (A18), reference Rego (B3/B16), stub BudgetPolicy data (B13), stub Approval (B19), stub LiteLLM callback chain (A1) for the bridge.

## 6. PR / Branch Mapping

### 6.1 Stack position — base branch = `wave/0`.
### 6.2 PR — `piece/A7-opa-gatekeeper` → base `wave/0`; carries spec-A7.md + plan-A7.md.
### 6.3 Merge order — W0 siblings independent; A7 ↔ A20/A1/A6 coordinate the dry-run decision shape; A7 ↔ B13/B19 coordinate OPA-data/elevation contracts; `wave/0` rolls up to main.

## 7. Effort Estimate

- S: TASK-09, TASK-11. M: TASK-01, TASK-03, TASK-04, TASK-05, TASK-06, TASK-07, TASK-08, TASK-10. L: TASK-02, TASK-12, TASK-13.
- Rollup: **L** (matches piece-index). Critical path ≈ TASK-01→02→03→06→13.

## 8. Rollback / Reversibility

Back out by reverting the OPA + Gatekeeper Helm releases (and the admission webhook registration). A7 holds **no system-of-record state** (bundles + constraints in Git), so rollback is clean. **Reverting A7 removes the platform's single policy layer** — admission for every CRD/XR, runtime authz at the gateway and egress, budget enforcement, approval elevation, Headlamp gating, and the substrate-claim guard all fail. With fail-closed admission, reverting Gatekeeper blocks CR writes; with fail-open it silently admits violations — either way it is a governance-down event. Critically, reverting A7 while A1/A6 still call its decision API means those enforcement points must default to deny (fail-closed) rather than allow. Treat as a controlled action; ensure dependent callers degrade to deny, not allow.
