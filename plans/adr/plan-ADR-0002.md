# PLAN ADR-0002 — OPA + Gatekeeper as the policy engine [PROPOSED]

> spec: SPEC-ADR-0002 · kind: ADR · tier: T1
> wave: authoring-parallel · estimate: S
> upstream-pieces: [] · downstream-pieces: [A17;A18;A20;A23;B2;B3;B16;B19;B20]

## 1. Implementation Strategy
Enforcement/verification map. ADR 0002 is enforced by component A7 (OPA/Gatekeeper install as the only engine + only admission path), with the runtime decision surface realized by B2 (LiteLLM OPA-bridge callback), the Envoy egress hook (ADR 0003), Headlamp gating (A9), and approval elevation (B19). Conformance is proven by: (a) cluster scans asserting no second policy engine and Gatekeeper-only admission; (b) per-kind admission-deny tests; (c) a Rego-as-code CI gate (SHA-pin + unit tests + RBAC-floor conformance). No new build work belongs to this ADR — it is a constraint map over A7/B2/B3/B16.

## 2. Ordered Task List
- TASK-01: Map each REQ to the enforcing component piece — produces: enforcement matrix — depends-on: []
- TASK-02: Specify the "single engine" cluster scan (no Kyverno/other governance webhook) — produces: CI/policy check (A7) — depends-on: [TASK-01]
- TASK-03: Specify per-kind Gatekeeper admission-deny conformance for the §4.1 surface — produces: Chainsaw suite (A7/B16) — depends-on: [TASK-01]
- TASK-04: Specify the runtime-decision conformance (LiteLLM/Envoy/Headlamp resolve via OPA) — produces: e2e + trace check (B2/A6/A9) — depends-on: [TASK-01]
- TASK-05: Specify the Rego-as-code gate (SHA-pin + unit tests + RBAC-floor conformance) — produces: CI gate spec (B3/B16, ADR 0030) — depends-on: [TASK-01]

## 3. Dependency Map
### 3.1 Upstream that must ship first (HARD)
- None — A7 is a foundation component.
### 3.2 Downstream blocked on this
- A17, A18, A20, A23, B2, B3, B16, B19, B20.
### 3.3 Continuous (non-blocking) inputs
- B14 test framework; B15 CI pipeline (Rego unit-test + bundle-lint required checks); B22 threat model.

## 4. Parallelizable Subtasks
TASK-02, TASK-03, TASK-04, TASK-05 fan out independently after TASK-01.

## 5. Test Strategy
- AC-01/02/03 → Chainsaw (admission webhook inventory + per-kind deny).
- AC-04 → Playwright + PyTest (Headlamp gate e2e; LiteLLM/Envoy OPA-query trace assertion).
- AC-05/06/07 → PyTest (bundle SHA-pin drift, Rego unit-test gate, RBAC-floor conformance).
Fixtures: stub LiteLLM callback + Envoy until A1/A6 land; canned Rego bundle for gate tests.

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/authoring`
### 6.2 PR — `piece/ADR-0002-opa-gatekeeper` → base `wave/authoring`; carries spec + plan.
### 6.3 Merge order — independent of sibling ADR PRs; rolls up to main.

## 7. Effort Estimate
TASK-01 S · TASK-02 S · TASK-03 S · TASK-04 S · TASK-05 S. Rollup: S. Critical path: TASK-01 → any gate task.

## 8. Rollback / Reversibility
Backing out means re-opening the policy-engine choice (Kyverno) and accepting two policy languages. Downstream breakage: every component that evaluates against OPA (admission, gateway, egress, approvals). Low reversibility once Rego is the platform-wide governance language.
