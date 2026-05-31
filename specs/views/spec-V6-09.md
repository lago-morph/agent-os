# SPEC V6-09 — Multi-tenancy and namespacing [PROPOSED]

> kind: VIEW · workstream: — · tier: T1
> upstream: [] · downstream: [] · adrs: [0016, 0028, 0029, 0037, 0018, 0031] · views: [6.9]
> canon-glossary: see _meta/glossary.md · canon-interface: see _meta/interface-contract.md

## 1. Purpose & Problem Statement

This view is the **integration contract** for the platform's multi-tenancy slice: the
architectural commitment that **the Kubernetes namespace is the tenancy boundary** (§6.9, ADR 0016).
A `Tenant` maps to one or more **Approved namespaces**; every Platform Agent, the CRDs it
references, the `Sandbox` it runs in, and the audit/observability data it emits are scoped to that
boundary. Tenant membership, accessible namespaces, and roles are carried as **Platform JWT** claims
minted by Keycloak (the claim schema is owned by V6-11 / ADR 0029; this view consumes it).

The slice is realized by several components — the `TenantOnboarding` XRD that provisions namespaces
and ServiceAccounts (B4 Composition, A21 reconciler), the SSO/auth proxy that scopes UI sessions
(B1), and the policy enforcement that gates cross-tenant calls (OPA via A7/B16). This SPEC does
**not** build any of them; it fixes the invariants, the spanning XR-API contract, the
three-layer visibility-enforcement model, and the tenant/capability lifecycle decoupling that each
participating component must honor so tenancy presents one coherent isolation surface.

## 2. Scope

### 2.1 In scope
- The invariant that the namespace is the tenancy boundary and that all platform CRDs are namespaced
  (no cluster-scoped platform CRDs in v1.0).
- The spanning contract that tenant context is carried **only** via Platform JWT claims
  (`platform_tenants`, `platform_namespaces`, `platform_roles`, `tenant_roles`,
  `capability_set_refs`) consumed by LiteLLM, OPA, Headlamp, LibreChat.
- The three-layer, defense-in-depth visibility-enforcement model (Gatekeeper admission, LiteLLM OPA
  callback, Envoy network policy) and the read-only / read-write / invisible visibility states.
- The `TenantOnboarding` XRD as the provisioning entry point, and the **decoupling of CapabilitySet
  lifecycle from tenant onboarding**.

### 2.2 Out of scope (and where it lives instead)
- The Keycloak **claim schema** definition and JWT issuance contract — **V6-11 / ADR 0029**.
- The `TenantOnboarding` Composition + namespace/ServiceAccount provisioning — **component B4** (XRD)
  and **component A21** (tenant onboarding reconciler) / ADR 0037.
- Upstream-IdP mapper authoring (AD/Okta → claim schema) — **out of scope, install owner** (§6.9).
- SSO session scoping at the UI layer — **component B1** (SSO/auth proxy).
- OPA policy content for cross-tenant decisions — **components A7 / B16**.
- CapabilitySet overlay and cross-namespace publication mechanics — **V6-08 / ADR 0032**.
- Per-event-type `platform.tenant.*` schemas — **component B12 registry**.

## 3. Context & Dependencies

Realizing components and what each contributes to the view:
- **B4** (Crossplane v2 Compositions) — owns the `TenantOnboarding` XRD that provisions the tenant
  namespace(s), templated default ServiceAccounts, and the cluster-OIDC claim mapping.
- **A21** (tenant onboarding reconciler) — drives the Headlamp + GitOps onboarding flow (ADR 0037).
- **B1** (SSO/auth proxy layer) — scopes UI sessions to the principal's `platform_namespaces` /
  `tenant_roles`.

ADR decisions honored:
- **ADR 0016** — multi-tenancy via Kubernetes namespaces with RBAC + OPA + network enforcement.
- **ADR 0028 / 0029** — identity federation chain and the Keycloak claim schema this view consumes.
- **ADR 0037** — tenant onboarding via Headlamp + a `TenantOnboarding` XRD.
- **ADR 0018** — RBAC-as-floor / OPA-as-restrictor governs every identity-scoped decision.
- **ADR 0031** — tenancy lifecycle changes flow under the `platform.tenant.*` CloudEvent namespace.

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
Spanning CRDs/XRDs (Canon; all namespaced):
- `TenantOnboarding` (XRD, owner B4) — `tenantId`, `namespaces[]`, `defaultServiceAccounts[]`,
  `clusterOIDCClaimMapping`. CapabilitySets intentionally not coupled (ADR 0037).
- `Agent` (owner A5) — admitted only into namespaces it has tenancy rights for; carries `exposes`
  (A2A/MCP) whose visibility is OPA-gated.
- All capability CRDs (`MCPServer`, `A2APeer`, `RAGStore`, `EgressTarget`, `Skill`, `CapabilitySet`,
  owner B13) — namespaced; the namespace is the tenancy boundary; cross-namespace reference is
  OPA-checked publication only.

Versioning: all follow ADR 0030 (per-component reconciler owns lifecycle).

### 4.2 APIs / SDK surfaces
- **Platform JWT claim consumption** (the spanning interface): LiteLLM (auth + virtual-key claim
  binding), OPA (every identity-scoped decision; `platform_namespaces` is authoritative for resource
  scope), Headlamp (UI scoping), LibreChat (endpoint visibility) all read the same claim set.
- **LiteLLM gateway registration API** — Platform Agents dynamically register A2A/MCP exposings;
  OPA decides whether the registration is allowed and what visibility it gets.
- **Headlamp admin override** — force-register / force-deregister an exposing.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
- Emitted: `platform.tenant.*` — tenant onboarding, namespace-association changes, cross-tenant
  publish events.
- Consumed: by observability and audit subscribers; cross-tenant call decisions also surface under
  `platform.policy.*` (OPA decisions). Per-event-type schemas deferred to B12.

### 4.4 Data schemas / connection-secret contracts
N/A — this view introduces no substrate-backed primitive. Tenant databases are provisioned via
`AgentDatabase` (scope `tenant`) under V6-03's data architecture, not here.

## 5. OSS-vs-Custom Decision
N/A — VIEW. Realizing components carry their own OSS-vs-custom decisions (Keycloak per ADR 0028/0029;
B4 Crossplane Compositions; A21 custom reconciler; B1 oauth2-proxy config).

## 6. Functional Requirements

- **REQ-V6-09-01** (invariant): The Kubernetes namespace MUST be the tenancy boundary; a `Tenant`
  maps to one or more namespaces and every Platform Agent, referenced CRD, `Sandbox`, and audit /
  observability record is scoped to a namespace.
- **REQ-V6-09-02** (invariant): All platform CRDs MUST be namespaced; there are no cluster-scoped
  platform CRDs in v1.0.
- **REQ-V6-09-03** (constraint): Tenant context MUST be carried only via the Platform JWT claim
  schema (`platform_tenants`, `platform_namespaces`, `platform_roles`, `tenant_roles`,
  `capability_set_refs`); no component derives tenancy from any other source. No platform component
  trusts upstream identity directly.
- **REQ-V6-09-04** (constraint): `platform_namespaces` MUST be authoritative for OPA decisions on
  resource scope; RBAC remains the permission floor and OPA may further restrict per-decision
  (ADR 0018).
- **REQ-V6-09-05** (constraint): Cross-namespace / cross-tenant visibility MUST be policy-driven, not
  implicit, resolving to one of: read-only-visible, read-write-visible (to specific tenants), or
  invisible (default for tenant-private agents).
- **REQ-V6-09-06** (constraint): Cross-tenant call authorization MUST be enforced at three layers,
  defense-in-depth: `Agent` CRD admission (Gatekeeper), LiteLLM gateway (OPA callback at request
  time), and network policy at the Envoy egress proxy. A failure at the gateway layer MUST still be
  blocked at the network layer.
- **REQ-V6-09-07** (constraint): A Platform Agent exposing an A2A/MCP interface MUST register
  dynamically with LiteLLM at startup, gated by OPA, which decides admission and visibility.
- **REQ-V6-09-08** (constraint): Cross-tenant `CapabilitySet` reference MUST be allowed only when
  explicitly published with an OPA-checked policy; the default is namespace-private.
- **REQ-V6-09-09** (constraint): Tenant provisioning MUST flow through the `TenantOnboarding` XRD
  (Headlamp + GitOps), creating the tenant namespace(s), templated default ServiceAccounts wired to
  the cluster OIDC issuer, and the claim-to-tenant binding.
- **REQ-V6-09-10** (constraint): CapabilitySet lifecycle MUST be decoupled from tenant onboarding in
  code — a tenant may exist with no CapabilitySet, and a CapabilitySet may be added/removed/replaced
  without re-onboarding.

## 7. Non-Functional Requirements
- **Multi-tenancy (§6.9):** this view *is* the tenancy contract; isolation is enforced at admission,
  gateway, and network layers, not by convention.
- **Security:** the three-layer model is defense-in-depth; no single layer is trusted alone. Upstream
  identity is never trusted directly — only the Keycloak-issued Platform JWT.
- **Observability (§6.5):** tenancy is a label dimension; `LogLevel` scope can be `tenant`; audit and
  observability data inherit the namespace boundary.
- **Versioning (ADR 0030):** `TenantOnboarding` XRD follows per-component (B4) versioning.

## 8. Cross-Cutting Deliverable Checklist
N/A — VIEW. The view is realized by components; cross-cutting deliverables (Helm, OPA integration,
Headlamp plugin, audit emission, tests) belong to B4/A21/B1/A7/B16, verified end-to-end in PLAN §5.

## 9. Acceptance Criteria
The view holds when:
- **AC-V6-09-01** (→REQ-01/02): Every platform CRD instance resolves to a namespace; no platform CRD
  is cluster-scoped; resources in tenant-A's namespace are not enumerable from tenant-B's scope.
- **AC-V6-09-02** (→REQ-03/04): An OPA decision on resource scope uses `platform_namespaces` from the
  Platform JWT; a request with no matching namespace claim is denied.
- **AC-V6-09-03** (→REQ-05/06): A cross-tenant A2A call disallowed by policy is blocked at the
  gateway; with the gateway bypassed/faked, the same call is still blocked by Envoy network policy.
- **AC-V6-09-04** (→REQ-07): A Platform Agent's A2A/MCP exposing appears in the gateway only after an
  OPA-allowed dynamic registration; a denied registration yields no discoverable interface.
- **AC-V6-09-05** (→REQ-08): A `CapabilitySet` referenced cross-namespace without an OPA-checked
  publication is rejected.
- **AC-V6-09-06** (→REQ-09): Applying a `TenantOnboarding` XR provisions the namespace(s),
  default ServiceAccounts wired to the cluster OIDC issuer, and the tenant onboarding binding.
- **AC-V6-09-07** (→REQ-10): A tenant can be onboarded with zero CapabilitySets, and a CapabilitySet
  can be removed without re-running onboarding.

## 10. Risks & Open Questions
- **[PROPOSED]** This SPEC is `[PROPOSED]` pending Canon review; it coins no new names.
- **(med)** The concrete `platform_roles` role catalog (`platform-admin`, `operator`, etc.) is
  design-time; this view fixes only the XR schema's presence, not the catalog. Routed to V6-11 /
  design specs.
- **(med)** Self-red-team: an agent that registers in a namespace it has admission rights for but
  should not expose cross-tenant — mitigated because REQ-07/06 require OPA to decide both admission
  *and* visibility, with network policy as the final backstop; the gateway-trust-only failure mode is
  explicitly covered by AC-V6-09-03.
- **(low)** Whether a single Platform JWT may carry multiple tenants simultaneously and how OPA
  disambiguates active tenant per request — claim shape supports it (`platform_tenants` is an array);
  active-tenant selection mechanics deferred to OPA policy design (B16).

## 11. References
- architecture-overview.md §6.9 (multi-tenancy, ~L720–772): claim table (~L728–740), visibility
  model (~L744–762), tenant onboarding (~L764–772).
- ADR 0016 (namespace tenancy + RBAC/OPA/network), ADR 0028/0029 (federation + claim schema),
  ADR 0037 (TenantOnboarding via Headlamp + XRD), ADR 0018 (RBAC-floor/OPA-restrictor),
  ADR 0031 (`platform.tenant.*`).
- Realizing components: B4, A21, B1 (+ A7/B16 for policy). Related views: V6-11 (identity
  federation / claim schema), V6-08 (capability cross-tenant publish), V6-12 (CRD inventory).
