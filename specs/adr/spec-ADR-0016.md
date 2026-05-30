# SPEC ADR-0016 — Multi-tenancy via Kubernetes namespaces with RBAC + OPA + network enforcement [PROPOSED]

> kind: ADR · workstream: — · tier: T1
> upstream: [A7, A6] · downstream: [A1, A9, A21, A22, B13, B16, A8] · adrs: [0016] · views: [6.9]
> canon-glossary: 6aadcc2a4f38 · canon-interface: 54f5ede58e5f

## 1. Purpose & Problem Statement
ADR 0016 is a settled decision: **tenancy is a Kubernetes namespace** — a tenant maps to one or more namespaces, and the namespace boundary is the tenancy boundary for Platform Agents, referenced CRDs, Sandboxes, and emitted audit/observability data. Isolation is enforced defense-in-depth across four independent layers: RBAC (floor), Gatekeeper admission, OPA runtime authorization at LiteLLM, and network policy at the Envoy egress proxy. This SPEC states the obligations that honoring the decision imposes, the three cross-namespace visibility modes, and conformance. It does not re-argue namespaces vs a first-class tenancy abstraction.

The problem the decision solves: tenants must be isolated (agents, CRDs, sandboxes, audit, observability) while allowing controlled cross-tenant discovery/collaboration, without duplicating primitives the platform already relies on.

## 2. Scope
### 2.1 In scope
- Namespace = tenancy boundary for all reconciled resources and emitted data.
- The four independent enforcement layers (RBAC floor, Gatekeeper admission, OPA at LiteLLM on every A2A/MCP call including dynamic registration, network policy at Envoy).
- The three visibility modes: invisible (default), read-only-visible (all tenants), read-write-visible (specific tenants by RBAC + OPA).
- Tenant identity carried as Keycloak JWT claims consumed uniformly by LiteLLM, OPA, Headlamp, LibreChat.
- Dynamic agent registration gated by the same OPA policy as cross-namespace calls.
- Cross-tenant resource sharing only via explicit OPA-checked publication; default namespace-private.
- Headlamp admin override (force-register/deregister), itself OPA-gated and audit-emitting.

### 2.2 Out of scope (and where it lives instead)
- Identity federation chain — ADR 0028; JWT claim schema — ADR 0029.
- Tenant onboarding flow / `TenantOnboarding` XRD — ADR 0037 / component **A21**.
- RBAC-floor / OPA-restrictor composition rule — ADR 0018.
- Tenant-scoped resource quotas + `platform_roles` catalog + cross-tenant collaboration patterns — design-time (architecture-backlog §1.2).
- Multi-cluster federation — out of scope (ADR 0026).

## 3. Context & Dependencies
Upstream consumed: **A7** OPA/Gatekeeper (admission + runtime authorization), **A6** Envoy egress proxy (network policy). Downstream consumers / conformers: **A1** LiteLLM (OPA gate on every call), **A9/A22** Headlamp (visibility + admin override), **A21** tenant onboarding reconciler, **B13** kopf operator (namespaced capability CRDs), **B16** OPA content (cross-namespace rules), **A8** LibreChat (claim-driven scoping).

ADR decisions honored:
- **ADR 0016** (this) — namespace = tenancy boundary; four-layer enforcement; three visibility modes.
- **ADR 0002** — Gatekeeper admission + OPA runtime authorization.
- **ADR 0003** — Envoy egress network policy as the fourth, independent layer.
- **ADR 0018** — RBAC-as-floor / OPA-as-restrictor composes; OPA cannot bridge RBAC-separated tenants.
- **ADR 0013** — CapabilitySets default namespace-private; cross-namespace reference requires OPA-checked publication.
- **ADR 0028 / 0029** — tenant identity is Keycloak JWT claims (`platform_tenants`, `platform_namespaces`, `platform_roles`, `tenant_roles`, `capability_set_refs`).
- **ADR 0037 / 0039** — onboarding via `TenantOnboarding` XRD through Headlamp editors, Git/ArgoCD SoR.
- **ADR 0026** — no multi-cluster federation.

## 4. Interfaces & Contracts
### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
- `TenantOnboarding` (XRD, namespaced, ADR 0037) — `tenantId`, `namespaces[]`, `defaultServiceAccounts[]`, `clusterOIDCClaimMapping`. Provisions namespace, default ServiceAccounts, JWT-claim-to-tenant mapping; CapabilitySets intentionally not coupled.
- All v1.0 platform CRDs are namespaced — the namespace is the scoping boundary.

### 4.2 APIs / SDK surfaces
- Platform JWT consumed by LiteLLM, OPA, Headlamp, LibreChat. No new SDK surface introduced by this ADR.
- Headlamp admin override path (force-register / force-deregister) — OPA-gated, audit-emitting.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
- Tenant onboarding, namespace-association changes, cross-tenant publish events under `platform.tenant.*`.
- OPA decisions / dynamic-registration accept-deny under `platform.policy.*`; admin overrides emit audit under `platform.audit.*`. No new top-level namespace.

### 4.4 Data schemas / connection-secret contracts
- N/A — no substrate primitive introduced; tenancy is enforced over existing primitives. `[PROPOSED — not in source]` any tenant-quota field set (deferred to design).

## 5. OSS-vs-Custom Decision
N/A — ADR. (Enforcement note: composes upstream **Kubernetes RBAC**, **OPA/Gatekeeper** (A7), **Envoy** (A6), **Keycloak** identity — config/wrap, no fork; the namespace is the native boundary. Rejected alternative — a first-class tenancy abstraction layered on top — is not built.)

## 6. Functional Requirements
- REQ-ADR-0016-01: A tenant MUST map to one or more Kubernetes namespaces, and the namespace boundary MUST be the tenancy boundary for all Platform Agents, referenced CRDs, Sandboxes, and emitted audit/observability data.
- REQ-ADR-0016-02: Isolation MUST be enforced across four independent layers: (1) RBAC floor, (2) Gatekeeper admission on Agent and related CRDs, (3) OPA runtime authorization at LiteLLM on every A2A/MCP call including dynamic registration, (4) network policy at the Envoy egress proxy.
- REQ-ADR-0016-03: The four layers MUST be independent — a misconfiguration in one (e.g., over-broad RBAC) MUST still be bounded by the others.
- REQ-ADR-0016-04: Cross-namespace exposure MUST be one of exactly three modes: invisible (default), read-only-visible to all tenants, or read-write-visible to specific tenants by RBAC + OPA.
- REQ-ADR-0016-05: Dynamic agent registration MUST be governed by the same OPA policy that gates cross-namespace calls (one engine, same claims).
- REQ-ADR-0016-06: Cross-tenant resource sharing MUST require explicit OPA-checked publication; the default for every resource MUST be namespace-private.
- REQ-ADR-0016-07: Tenant identity MUST be carried as Keycloak JWT claims consumed uniformly by LiteLLM, OPA, Headlamp, and LibreChat — not as a platform-specific construct.
- REQ-ADR-0016-08: The Headlamp admin override (force-register / force-deregister) MUST itself be OPA-gated and audit-emitting; tenant administrators MUST NOT gain implicit cross-tenant powers.
- REQ-ADR-0016-09: Tenant onboarding MUST be performed via a `TenantOnboarding` XRD through the Headlamp editors with Git/ArgoCD as the system of record (ADR 0037); CapabilitySets MUST remain decoupled from the onboarding XRD.

## 7. Non-Functional Requirements
- Security/tenancy (§6.9): defense-in-depth is the security property — even a gateway-authorization bypass cannot reach the callee because network policy at Envoy bounds it independently.
- Composition: namespace-scoped RBAC defines the boundary; OPA layers cross-tenant restrictions without bridging tenants RBAC separated (ADR 0018).
- Observability (§6.5): tenant/namespace-scoped audit + observability data inherit the boundary.
- Versioning (ADR 0030): `TenantOnboarding` XRD versioned with conversion webhooks.

## 8. Cross-Cutting Deliverable Checklist
N/A — ADR (verification map in the PLAN). The §14.1 set is owned by enforcing components (A7, A6, A1, A9/A22, A21, B16).

## 9. Acceptance Criteria
- AC-ADR-0016-01: Honored when an agent/CRD/Sandbox in namespace A is invisible from namespace B by default, and its audit/observability data are scoped to A. (REQ-01/04)
- AC-ADR-0016-02: Honored when an A2A/MCP call from A to B is denied unless all four layers permit it. (REQ-02)
- AC-ADR-0016-03: Honored when an over-broad RBAC grant in A still fails to reach B because Gatekeeper/OPA/network policy bound it. (REQ-03)
- AC-ADR-0016-04: Honored when each of the three visibility modes is exercised and enforced as specified. (REQ-04)
- AC-ADR-0016-05: Honored when a dynamic registration grant and a cross-namespace call grant are decided by the same OPA engine on the same claims. (REQ-05)
- AC-ADR-0016-06: Honored when an unpublished resource is unreferenceable cross-namespace, and a published one is reachable only after the OPA-checked publication. (REQ-06)
- AC-ADR-0016-07: Honored when LiteLLM, OPA, Headlamp, and LibreChat all read tenant identity from the same JWT claims. (REQ-07)
- AC-ADR-0016-08: Honored when a Headlamp force-register is denied without the right OPA grant and writes an audit record when allowed. (REQ-08)
- AC-ADR-0016-09: Honored when a `TenantOnboarding` XRD provisions namespace + ServiceAccounts + claim mapping via Git/ArgoCD, with no CapabilitySet coupling. (REQ-09)

## 10. Risks & Open Questions
- OQ-1 (med): Tenant-scoped quotas (agents, sandboxes, virtual keys, cost, rate-limit) and the `platform_roles` catalog remain design-time (architecture-backlog §1.2). `[PROPOSED]`
- OQ-2 (low): Cross-tenant collaboration patterns when explicitly published are design-time. `[PROPOSED]`
- R-1 (low): Defense-in-depth assumes the four layers stay independent; a shared misconfiguration source (e.g., a bad OPA bundle deployed everywhere) reduces independence — bounded by Gatekeeper + network policy still holding.

## 11. References
- ADR 0016 (`adr/0016-multi-tenancy-via-namespaces.md`) — the decision enforced here.
- architecture-overview.md §6.9 (multi-tenancy & namespacing); architecture-backlog.md §1.2, §6.
- Enforcing components: A7 (OPA/Gatekeeper), A6 (Envoy), A1 (LiteLLM), A9/A22 (Headlamp), A21 (onboarding), B16 (OPA content).
- Related ADRs: 0002, 0003, 0013, 0018, 0026, 0028, 0029, 0037, 0039.
