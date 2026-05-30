# PLAN ADR-0016 — Multi-tenancy via namespaces [PROPOSED]

> spec: SPEC-ADR-0016 · kind: ADR · tier: T1
> wave: authoring-parallel · estimate: M
> upstream-pieces: [A7, A6] · downstream-pieces: [A1, A9, A21, A22, B13, B16, A8]

## 1. Implementation Strategy
Enforcement/verification map. ADR 0016 is enforced by four independent layers: **A7** (RBAC floor + Gatekeeper admission + OPA runtime authorization), **A6** (Envoy network policy), **A1** (OPA gate on every A2A/MCP call including dynamic registration), and **A9/A22** (Headlamp visibility + OPA-gated, audit-emitting admin override). Tenant identity is the Keycloak JWT claim set consumed uniformly by LiteLLM/OPA/Headlamp/LibreChat (A8). Onboarding is enforced by **A21** via the `TenantOnboarding` XRD through Headlamp editors with Git/ArgoCD SoR. Conformance is tested by proving namespace-scoped invisibility-by-default, the three visibility modes, layer independence (a bad RBAC grant still bounded by the other three), one OPA engine for registration + cross-namespace calls, and namespace-private defaults.

## 2. Ordered Task List
- TASK-01: Map each REQ to its enforcing layer + component piece ID — produces: four-layer enforcement matrix — depends-on: []
- TASK-02: Verify A7 carries RBAC-floor + Gatekeeper admission + OPA runtime obligations — produces: A7 conformance checklist — depends-on: [TASK-01]
- TASK-03: Verify A6 Envoy network policy as the independent fourth layer — produces: A6 conformance note — depends-on: [TASK-01]
- TASK-04: Verify the three visibility modes (invisible/read-only/read-write) at A1/A9 — produces: visibility-mode trace — depends-on: [TASK-01]
- TASK-05: Verify dynamic registration uses the same OPA policy as cross-namespace calls — produces: OPA single-engine note — depends-on: [TASK-02]
- TASK-06: Verify namespace-private default + OPA-checked publication (cross-cut with ADR 0013) — produces: Chainsaw test — depends-on: [TASK-02]
- TASK-07: Verify Headlamp admin override is OPA-gated + audit-emitting — produces: A22 trace — depends-on: [TASK-04]
- TASK-08: Verify JWT-claim uniformity across LiteLLM/OPA/Headlamp/LibreChat — produces: claim-consumption matrix — depends-on: [TASK-01]
- TASK-09: Verify onboarding via `TenantOnboarding` XRD with CapabilitySet decoupling (ADR 0037 cross-check) — produces: A21 trace — depends-on: [TASK-01]

## 3. Dependency Map
### 3.1 Upstream pieces that must ship first (HARD)
- A7 (OPA/Gatekeeper), A6 (Envoy).
### 3.2 Downstream pieces blocked on this
- A1 (gate), A9/A22 (visibility/override), A21 (onboarding), B13 (namespaced CRDs), B16 (cross-namespace rules), A8 (claim scoping).
### 3.3 Continuous (non-blocking) inputs
- ADR 0028/0029 (identity/claims), ADR 0037 (onboarding flow), ADR 0018 (composition), B22 threat model.

## 4. Parallelizable Subtasks
TASK-02, TASK-03, TASK-04, TASK-08, TASK-09 fan out once TASK-01 lands. TASK-05/06 follow TASK-02; TASK-07 follows TASK-04.

## 5. Test Strategy
- AC-01 → Chainsaw: namespace-A resource invisible from B; audit/observability scoped to A.
- AC-02/AC-03 → Chainsaw: cross-namespace call denied unless all four layers permit; over-broad RBAC still bounded by Gatekeeper/OPA/network policy.
- AC-04 → Chainsaw: each visibility mode exercised and enforced.
- AC-05 → PyTest: registration + cross-namespace call decided by the same OPA engine on the same claims.
- AC-06 → Chainsaw: unpublished resource unreferenceable cross-namespace; published one reachable after OPA-checked publication.
- AC-07 → PyTest: LiteLLM/OPA/Headlamp/LibreChat read tenancy from the same JWT claims.
- AC-08 → Playwright: Headlamp force-register denied without OPA grant; writes audit when allowed.
- AC-09 → Chainsaw: `TenantOnboarding` provisions namespace+SAs+claim mapping via Git/ArgoCD, no CapabilitySet coupling.
- Fixtures/fakes: stub Keycloak JWT issuer, OPA decision endpoint, Envoy ext_authz for not-yet-landed deps.

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/authoring`
### 6.2 PR — `piece/ADR-0016-multi-tenancy-via-namespaces` → base `wave/authoring`; carries spec + plan.
### 6.3 Merge order — independent of sibling ADR specs; rolls up to main. Coordinate with ADR 0037 onboarding spec.

## 7. Effort Estimate
M overall. Per-task: TASK-01 S, 02 M, 03 S, 04 M, 05 S, 06 S, 07 S, 08 S, 09 S. Critical path: TASK-01 → 02 → 05.

## 8. Rollback / Reversibility
Decision record; back out by reverting spec+plan. If reverted, the platform loses its defense-in-depth tenancy contract; A7/A6/A1/A9/A21 lose their conformance anchor. No runtime artifact is deleted. Tenant-quota and `platform_roles`-catalog sub-decisions remain design-time regardless.
