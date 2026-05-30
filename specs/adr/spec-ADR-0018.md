# SPEC ADR-0018 — RBAC-as-floor / OPA-as-restrictor enforcement model platform-wide [PROPOSED]

> kind: ADR · workstream: — · tier: T1
> upstream: [A7, B3] · downstream: [A1, A6, B19, B16, A14, B13, A9] · adrs: [0018] · views: [6.6]
> canon-glossary: 6aadcc2a4f38 · canon-interface: 54f5ede58e5f

## 1. Purpose & Problem Statement
ADR 0018 is a settled decision and platform invariant: **RBAC is the permission ceiling; OPA may further restrict any decision; OPA never grants permissions RBAC has not already given.** Every OPA decision point in the platform — gateway callbacks, admission, egress, approvals, virtual-key issuance, capability access, dashboard visibility — evaluates against an identity already bounded by Kubernetes RBAC and the Keycloak JWT claim schema, and may return only a result equal to or more restrictive than RBAC permits. This SPEC states the composition contract the invariant imposes on every OPA decision point and the authoring rule it imposes on component authors. It does not re-argue the model.

The problem the decision solves: without an explicit contract, reviewers cannot reason about whether an OPA rule widens access beyond RBAC, and authors cannot tell which layer to encode a rule in.

## 2. Scope
### 2.1 In scope
- The composition contract: OPA is always a restrictor over an RBAC-bounded identity, never a grantor.
- The definition of "restrict" — raise the bar (e.g., elevate a required approval level) and outright deny; explicitly excludes any escalation.
- The uniform application to all OPA decision points (LiteLLM callbacks, Envoy egress, Gatekeeper admission, the approval system, virtual-key issuance, capability access, dashboard visibility).
- The authoring rule for component authors: broadest permission in RBAC; contextual narrowing in OPA.
- Composition with multi-tenancy (ADR 0016) and the approval system (ADR 0017).

### 2.2 Out of scope (and where it lives instead)
- The OPA engine + Gatekeeper install — component **A7** / ADR 0002.
- The OPA policy library framework — component **B3**; initial content — **B16**.
- The Keycloak JWT claim schema — ADR 0029; federation chain — ADR 0028.
- Per-decision-point policy contents — owning component design specs.
- Policy simulation/preview — ADR 0038.

## 3. Context & Dependencies
Upstream consumed: **A7** OPA/Gatekeeper (the engine evaluating every decision), **B3** OPA policy library framework (encodes restrictor rules). Downstream consumers / conformers: **A1** LiteLLM callbacks, **A6** Envoy egress, **B19** approval system (elevate-only), **B16** OPA content, **A14** HolmesGPT toolsets (RBAC ceiling), **B13** virtual-key/capability access, **A9** dashboard visibility.

ADR decisions honored:
- **ADR 0018** (this) — RBAC ceiling; OPA restrict-or-deny only; never grant.
- **ADR 0002** — OPA/Gatekeeper is the engine across admission + runtime.
- **ADR 0016** — namespace-scoped RBAC defines the tenant boundary; OPA cannot bridge RBAC-separated tenants.
- **ADR 0017** — OPA may only elevate a required approval level; never lower it; never make an action approvable beyond RBAC.
- **ADR 0012** — HolmesGPT's ServiceAccount RBAC is the upper bound; policy edits cannot widen it.
- **ADR 0029** — Keycloak claims mirror RBAC for non-Kubernetes surfaces.

## 4. Interfaces & Contracts
### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
N/A — ADR 0018 introduces no CRD; it constrains how every OPA decision point composes with RBAC. Gatekeeper constraints (ADR 0002) and the `Approval` CRD (ADR 0017) inherit the contract.

### 4.2 APIs / SDK surfaces
- Every OPA decision point (LiteLLM callback API, Envoy ext_authz, Gatekeeper admission, approval `approval.elevation`, virtual-key issuance, capability bind, dashboard `visibility`) exposes a decision that MUST be ≤ the RBAC-permitted result. No new SDK surface introduced.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
- OPA decisions and policy violations under `platform.policy.*`; policy-bypass attempts under `platform.security.*`. No new top-level namespace.

### 4.4 Data schemas / connection-secret contracts
- N/A — no substrate primitive introduced; the identity bound is the Kubernetes RBAC + Keycloak JWT claim set (ADR 0029).

## 5. OSS-vs-Custom Decision
N/A — ADR. (Enforcement note: realized over upstream **Kubernetes RBAC** + **OPA/Gatekeeper** (A7) and the custom OPA policy library (B3/B16) — config/wrap + build-new policy content, no fork. Rejected alternative — encoding grants in OPA or letting OPA widen RBAC — is forbidden by the invariant.)

## 6. Functional Requirements
- REQ-ADR-0018-01: Every OPA decision in the platform MUST be evaluated against an identity already bounded by Kubernetes RBAC and the Keycloak JWT claim schema.
- REQ-ADR-0018-02: An OPA decision MUST return a result equal to or more restrictive than what RBAC permits; OPA MUST NOT grant any permission RBAC has not already given.
- REQ-ADR-0018-03: "Restrict" MUST mean raising the bar (e.g., elevating a required approval level) or outright deny, and MUST NOT include any form of escalation.
- REQ-ADR-0018-04: All OPA decision points — LiteLLM callbacks, Envoy egress, Gatekeeper admission, approval system, virtual-key issuance, capability access, dashboard visibility — MUST honor this contract; new decision points MUST inherit it by default.
- REQ-ADR-0018-05: The approval system (ADR 0017) MUST only allow OPA to elevate the required level and MUST NOT make an action approvable by an identity RBAC would not allow to act.
- REQ-ADR-0018-06: Virtual-key issuance and capability access MUST let OPA narrow only; OPA MUST NOT hand out a CapabilitySet binding the requestor's RBAC did not already authorize.
- REQ-ADR-0018-07: HolmesGPT toolsets MUST be bounded by their ServiceAccount RBAC even when an OPA policy would permit more (policy edits cannot widen the self-management agent's reach).
- REQ-ADR-0018-08: Component authors MUST encode the broadest permissible permission in RBAC and contextual narrowing (time, spend, data sensitivity, tenant relationship, request shape) in OPA; authors MUST NOT express grants in OPA.
- REQ-ADR-0018-09: Multi-tenancy (ADR 0016) MUST compose so OPA can layer cross-tenant restrictions but can never bridge tenants RBAC has separated.

## 7. Non-Functional Requirements
- Security: the model is the platform's review story — RBAC bounds what an identity can ever do; OPA bundles encode per-decision narrowing; no OPA rule may silently widen a grant.
- Observability (§6.5): OPA decisions emit under `platform.policy.*`; attempted bypasses under `platform.security.*`.
- Versioning (ADR 0030): policy bundles versioned with the OPA library (B3/B16).
- Predictability: new OPA decision points inherit the contract by default — a conformance lint should flag any policy that returns allow beyond the bounded identity.

## 8. Cross-Cutting Deliverable Checklist
N/A — ADR (verification map in the PLAN). The §14.1 set is owned by enforcing components (A7, B3, B16, A1, A6, B19, B13, A9).

## 9. Acceptance Criteria
- AC-ADR-0018-01: Honored when every sampled OPA decision point evaluates an RBAC+JWT-bounded identity. (REQ-01)
- AC-ADR-0018-02: Honored when a policy that returns allow for an action RBAC denies is rejected by conformance tests. (REQ-02)
- AC-ADR-0018-03: Honored when an attempted escalation rule fails review/CI, while a deny/elevate rule passes. (REQ-03)
- AC-ADR-0018-04: Honored when LiteLLM, Envoy, Gatekeeper, approval, virtual-key, capability, and dashboard decisions each demonstrate restrict-or-deny-only. (REQ-04)
- AC-ADR-0018-05: Honored when an `approval.elevation` rule can raise but not lower the level and cannot make an RBAC-disallowed actor an approver. (REQ-05)
- AC-ADR-0018-06: Honored when OPA cannot bind a CapabilitySet/issue a virtual key beyond the requestor's RBAC. (REQ-06)
- AC-ADR-0018-07: Honored when a HolmesGPT action OPA permits but its ServiceAccount RBAC forbids is blocked at RBAC. (REQ-07)
- AC-ADR-0018-08: Honored when a doc-lint over component specs confirms grants live in RBAC and narrowing in OPA. (REQ-08)
- AC-ADR-0018-09: Honored when an OPA rule cannot bridge two RBAC-separated tenants. (REQ-09)

## 10. Risks & Open Questions
- R-1 (med): Enforcement depends on every decision point evaluating a properly bounded identity; a decision point that fails to load RBAC/JWT context could appear to "grant" — mitigated by REQ-01 conformance and a default-deny posture.
- R-2 (low): Authoring discipline (grants in RBAC, narrowing in OPA) is review-enforced; a mis-encoded grant-in-OPA is caught by AC-02/AC-08.
- OQ-1 (low): A platform-wide conformance lint for "policy never widens RBAC" is implied; its home (B3 vs B16 vs CI) is `[PROPOSED]`.

## 11. References
- ADR 0018 (`adr/0018-rbac-floor-opa-restrictor.md`) — the decision enforced here.
- architecture-overview.md §6.6 (security & policy); architecture-backlog.md §6.
- Enforcing components: A7 (OPA/Gatekeeper), B3 (policy library framework), B16 (initial content), A1 (LiteLLM callbacks), A6 (Envoy), B19 (approval), B13 (virtual-key/capability), A9 (dashboard visibility).
- Related ADRs: 0002, 0012, 0016, 0017, 0029.
