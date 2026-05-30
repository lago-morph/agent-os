# PLAN ADR-0018 — RBAC-as-floor / OPA-as-restrictor [PROPOSED]

> spec: SPEC-ADR-0018 · kind: ADR · tier: T1
> wave: authoring-parallel · estimate: S
> upstream-pieces: [A7, B3] · downstream-pieces: [A1, A6, B19, B16, A14, B13, A9]

## 1. Implementation Strategy
Enforcement/verification map. ADR 0018 is a platform-wide composition invariant enforced by **A7** (the OPA engine evaluating RBAC+JWT-bounded identities) and **B3** (the policy library framework that encodes restrictor-only rules), inherited by every OPA decision point: **A1** (LiteLLM callbacks), **A6** (Envoy egress), **B19** (approval elevation), **B13** (virtual-key/capability bind), **A9** (dashboard visibility), and **A14** (HolmesGPT ServiceAccount ceiling). **B16** owns the initial restrictor content. Conformance is tested by a policy-level lint/property test proving no policy returns allow beyond the bounded identity, that "restrict" excludes escalation, and that multi-tenancy (ADR 0016) and approvals (ADR 0017) compose under the contract.

## 2. Ordered Task List
- TASK-01: Map each REQ to its enforcing decision point + component piece ID — produces: decision-point matrix — depends-on: []
- TASK-02: Verify A7/B3 evaluate RBAC+JWT-bounded identities at every decision point — produces: bounded-identity conformance note — depends-on: [TASK-01]
- TASK-03: Define the "policy never widens RBAC" conformance lint/property test — produces: CI policy gate — depends-on: [TASK-02]
- TASK-04: Verify "restrict = raise-bar/deny, never escalate" across LiteLLM/Envoy/Gatekeeper/approval/virtual-key/capability/dashboard — produces: restrictor audit — depends-on: [TASK-01]
- TASK-05: Verify approval elevate-only (ADR 0017) + HolmesGPT RBAC ceiling (ADR 0012) honor the contract — produces: cross-ADR trace — depends-on: [TASK-04]
- TASK-06: Verify authoring rule (grants in RBAC, narrowing in OPA) via component-spec doc-lint — produces: authoring lint — depends-on: [TASK-01]
- TASK-07: Verify multi-tenancy composition (OPA cannot bridge RBAC-separated tenants) — produces: ADR 0016 cross-check — depends-on: [TASK-04]

## 3. Dependency Map
### 3.1 Upstream pieces that must ship first (HARD)
- A7 (OPA/Gatekeeper engine), B3 (policy library framework).
### 3.2 Downstream pieces blocked on this
- A1, A6, B19, B13, A9 (decision points inherit the contract), B16 (initial content), A14 (RBAC ceiling).
### 3.3 Continuous (non-blocking) inputs
- ADR 0029 (JWT claims), ADR 0038 (policy simulator), B22 threat model.

## 4. Parallelizable Subtasks
TASK-04, TASK-06 fan out once TASK-01 lands. TASK-03 follows TASK-02; TASK-05/07 follow TASK-04.

## 5. Test Strategy
- AC-01 → PyTest: every sampled decision point evaluates an RBAC+JWT-bounded identity.
- AC-02/AC-03 → PyTest property test: a policy returning allow beyond RBAC, or an escalation rule, fails CI; deny/elevate passes.
- AC-04 → Chainsaw+PyTest: LiteLLM/Envoy/Gatekeeper/approval/virtual-key/capability/dashboard each demonstrate restrict-or-deny-only.
- AC-05 → PyTest: `approval.elevation` raises but cannot lower / widen approver set.
- AC-06 → PyTest: OPA cannot bind a CapabilitySet/issue a key beyond requestor RBAC.
- AC-07 → Chainsaw: HolmesGPT action OPA-permitted but SA-RBAC-forbidden is blocked at RBAC.
- AC-08 → doc-lint: grants live in RBAC, narrowing in OPA across component specs.
- AC-09 → PyTest: OPA rule cannot bridge two RBAC-separated tenants.
- Fixtures/fakes: synthetic RBAC + JWT identities, OPA bundle harness for not-yet-landed B16 content.

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/authoring`
### 6.2 PR — `piece/ADR-0018-rbac-floor-opa-restrictor` → base `wave/authoring`; carries spec + plan.
### 6.3 Merge order — independent of sibling ADR specs; rolls up to main.

## 7. Effort Estimate
S overall. Per-task: TASK-01 S, 02 S, 03 M, 04 M, 05 S, 06 S, 07 S. Critical path: TASK-01 → 02 → 03.

## 8. Rollback / Reversibility
Decision record; back out by reverting spec+plan. If reverted, the platform loses its single security-review composition story and every OPA decision point loses the restrict-only conformance gate; approvals (ADR 0017) and HolmesGPT (ADR 0012) lose their RBAC-ceiling anchor. No runtime artifact is deleted.
