# SPEC ADR-0029 — Required Keycloak JWT claim schema [PROPOSED]

> kind: ADR · workstream: — · tier: T1
> upstream: [] · downstream: [A1, A7, A9, A8, A21, B13] · adrs: [0029] · views: [6.9, 6.11]
> canon-glossary: cf2d1a754a58 · canon-interface: 45ee7b798c47

## 1. Purpose & Problem Statement

ADR 0029 is a settled decision: the platform commits to the JWT claim schema specified in architecture-overview.md §6.9 ("Required Keycloak JWT claims") as an architectural invariant. Every Keycloak-issued platform JWT MUST carry that schema — standard claims plus `platform_tenants`, `platform_namespaces`, `platform_roles`, `tenant_roles`, `capability_set_refs` with the types and required-ness specified there — and every platform component that consumes identity MUST consume only those claims. This SPEC states what honoring that contract obliges: a single source of truth (§6.9, not duplicated); both human and service-account principals carry the schema; the platform-shipped cluster-OIDC mappers (ADR 0028) land SA principals in this shape; schema changes are breaking and go through ADR 0030 versioning, dragging the mapper bundles along. It does not re-argue the claim names.

The problem the decision solves: LiteLLM (auth + virtual-key binding), OPA (every identity-dependent decision), Headlamp (UI scoping), and LibreChat (endpoint visibility) must make consistent authorization decisions from the same identity. If each invented its own claim names, OPA policies, gateway bindings, and UI gating would drift, and per-install upstream-IdP customization would be re-mapped per consumer. The schema is also the multi-tenancy contract at the identity layer (`platform_tenants`/`platform_namespaces`/`tenant_roles`).

## 2. Scope

### 2.1 In scope
- The obligation that every Keycloak-issued platform JWT carries the §6.9 schema (standard claims + the five platform claims) with the §6.9 types and required-ness.
- The obligation that every identity-consuming component consumes ONLY those claims (no component-invented claim names).
- The obligation that both human and service-account principals carry the schema (the cluster-OIDC mappers land SA principals in the same shape).
- The obligation that the platform ships working, tested cluster-OIDC mappers for kind/EKS/AKS producing this schema.
- The obligation that any schema change (rename, type change, add/remove required claim) is a breaking change routed through ADR 0030 versioning, with mapper bundles pinned to the schema version.
- The §6.9 contract as the single source of truth (referenced, not duplicated).

### 2.2 Out of scope (and where it lives instead)
- The federation chain producing the JWTs — **ADR 0028**.
- Upstream-IdP → platform-claim mappers (per-install translation rules) — install-owned; out of architecture scope.
- The concrete `platform_roles` catalog (role names like `platform-admin`, `operator`, `developer`, `viewer`, `auditor` and their semantics) — design-time, deferred (backlog §1.2, §4). This ADR fixes only presence and shape.
- Tenant onboarding (the binding moment for new identifiers) — **ADR 0037** / component **A21**.
- The backlog §6 wording over-claim ("mapper authoring is out of scope") — flagged for follow-up narrowing in second-pass non-ADR doc updates, not edited here.

## 3. Context & Dependencies

Upstream consumed: none — this is the contract-fixing decision. (It depends on §6.9 as the source of truth and on ADR 0028 for who produces conforming JWTs.)
Downstream consumers: **A1** (LiteLLM auth + virtual-key claim binding), **A7** (OPA identity-dependent decisions), **A9** (Headlamp plugin gating), **A8** (LibreChat endpoint visibility), **A21** (tenant onboarding populates the claims), **B13** (kopf operator binds `VirtualKey` to identity/claims).

ADR decisions honored:
- **ADR 0029** (this) — §6.9 claim schema is invariant; consume-only-these-claims; schema changes are breaking.
- **ADR 0028** — JWTs are produced by the federation chain; cluster-OIDC mappers land SA principals in this shape and are platform-shipped/tested.
- **ADR 0016** — namespace-based tenancy is expressed at the identity layer through `platform_tenants`/`platform_namespaces`/`tenant_roles`.
- **ADR 0018** — RBAC is the authoritative permission floor; these claims feed OPA, which may further restrict but cannot grant beyond RBAC.
- **ADR 0030** — any schema change goes through CRD/API versioning discipline; mapper bundle versions are pinned to the schema version.
- **ADR 0037** — onboarding is the moment new tenant identifiers first appear in the claims; onboarding adapts to the contract, not vice versa.

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs
- `VirtualKey` (kopf-operator B13, namespaced) — `ownerIdentity`, `capabilitySetRef`, `budgetRef`, `environment`, `allowedModels[]`, `ttl`. Bound against the claim schema; `capability_set_refs` (claim) gates issuance.
- `TenantOnboarding` (XRD, namespaced, ADR 0037) — `tenantId`, `namespaces[]`, `defaultServiceAccounts[]`, `clusterOIDCClaimMapping`. Onboarding is where `platform_tenants`/`platform_namespaces`/`tenant_roles` are first populated.

### 4.2 APIs / SDK surfaces
- The Platform JWT claim schema is itself a versioned contract surface (ADR 0030). Canonical claim set (Canon, glossary "Platform JWT"): `platform_tenants`, `platform_namespaces`, `platform_roles`, `tenant_roles`, `capability_set_refs`. The cluster-OIDC mapper bundles (ADR 0028) are the platform-shipped artifacts producing it, version-pinned to the schema. The concrete `platform_roles` *values* are deferred — `[PROPOSED — not in source]` if enumerated.

### 4.3 CloudEvents emitted / consumed
- Identity-dependent policy decisions emit `platform.policy.*`; tenant identifier changes emit `platform.tenant.*` (ADR 0031, §2). A claim-schema version change is a versioning/security-relevant event; per-event-type names deferred to B12 — `[PROPOSED — not in source]`.

### 4.4 Data schemas / connection-secret contracts
N/A — the claim schema is a JWT payload contract, not a connection secret. No data-store schema introduced; §6.9 is the single source of truth and is referenced, not duplicated here.

## 5. OSS-vs-Custom Decision
N/A — ADR. (Enforcement note: upstream **Keycloak** issues the JWTs; the platform commits to the claim *schema* (destination shape) and ships custom cluster-OIDC mapper bundles producing it. Upstream-IdP mappers are install-owned, not built by the platform. No fork of Keycloak.)

## 6. Functional Requirements
- REQ-ADR-0029-01: Every Keycloak-issued platform JWT MUST carry the §6.9 schema — standard claims plus `platform_tenants`, `platform_namespaces`, `platform_roles`, `tenant_roles`, `capability_set_refs` — with the §6.9 types and required-ness.
- REQ-ADR-0029-02: Every platform component that consumes identity MUST consume ONLY the §6.9 claims; no component MUST invent its own claim names or expected shapes.
- REQ-ADR-0029-03: Both human-user and service-account principals MUST carry the schema; the platform-shipped cluster-OIDC mappers MUST land SA-derived principals in the same §6.9 shape as human principals.
- REQ-ADR-0029-04: The platform MUST ship working, tested cluster-OIDC mappers for kind, EKS, and AKS that produce this schema, testable in the platform's own CI against the supported distributions.
- REQ-ADR-0029-05: Service-account principals MUST carry `capability_set_refs`; OPA MUST use it to gate dynamic A2A/MCP registration and virtual-key issuance.
- REQ-ADR-0029-06: Any schema change (rename, type change, add/remove a required claim) MUST be treated as a breaking change to every consumer AND the platform-shipped mappers, routed through ADR 0030 versioning, with mapper bundle versions pinned to the schema version they target.
- REQ-ADR-0029-07: §6.9 MUST remain the single source of truth; the schema MUST NOT be duplicated in component specs (referenced, not copied).
- REQ-ADR-0029-08: These claims MUST feed OPA only as input to per-decision restriction; OPA MUST NOT grant beyond the RBAC floor (ADR 0018).

## 7. Non-Functional Requirements
- Security/multi-tenancy (§6.9): the schema is the identity-layer tenancy contract; no platform component trusts upstream identity directly (ADR 0028).
- Testability: the SA-namespace+annotations → `platform_tenants`/`platform_namespaces`/`capability_set_refs` mapping is an architectural commitment caught by platform CI, not customer-install time.
- Versioning (ADR 0030): schema changes carry a second blast radius across the kind/EKS/AKS mapper bundles, requiring coordinated bundle updates pinned to the schema version.
- Stability: OPA policies, LiteLLM virtual-key bindings, Headlamp gating, and LibreChat visibility may all be authored against the §6.9 names/types as a stable contract.

## 8. Cross-Cutting Deliverable Checklist
N/A — ADR (verification map lives in the PLAN). The §14.1 set applies to enforcing components (A1, A7, A9, A8, A21, B13) and the mapper bundles, not to this contract record.

## 9. Acceptance Criteria
- AC-ADR-0029-01: Honored when a sampled human JWT and a sampled SA JWT both validate against the §6.9 schema (all five platform claims present with correct types/required-ness). (REQ-01/03)
- AC-ADR-0029-02: Honored when each identity consumer (LiteLLM, OPA, Headlamp, LibreChat) reads only §6.9 claim names and a lint finds no invented claim names. (REQ-02/07)
- AC-ADR-0029-03: Honored when the platform-shipped cluster-OIDC mappers produce the §6.9 shape identically on kind/EKS/AKS in platform CI. (REQ-04)
- AC-ADR-0029-04: Honored when an SA principal's `capability_set_refs` gates a dynamic A2A/MCP registration and a `VirtualKey` issuance through OPA. (REQ-05)
- AC-ADR-0029-05: Honored when a simulated schema change is routed through ADR 0030 versioning and the kind/EKS/AKS mapper bundles are updated in lockstep, pinned to the new schema version. (REQ-06)
- AC-ADR-0029-06: Honored when OPA grants nothing beyond the RBAC floor given these claims. (REQ-08)

## 10. Risks & Open Questions
- OQ-1 (med): The `platform_roles` catalog (role names + semantics) is design-time, deferred (backlog §1.2, §4); ACs that depend on specific role values are partial until it lands. `[PROPOSED]`
- OQ-2 (low): backlog §6 currently states "mapper authoring is out of scope" without the upstream-vs-cluster-OIDC split; flagged for second-pass narrowing — not edited in this ADR. `[PROPOSED]`
- R-1 (high): A schema change is a breaking change across every consumer AND the platform-shipped mappers; mismanaged, it malforms principals platform-wide; mitigated by ADR 0030 versioning + mapper-bundle pinning + platform-CI testing.

## 11. References
- ADR 0029 (`adr/0029-keycloak-jwt-claim-schema.md`) — the decision enforced here.
- architecture-overview.md §6.9 (Required Keycloak JWT claims — single source of truth), §6.11 (identity federation).
- architecture-backlog.md §1.2, §4 (role catalog deferred), §6 (invariant; wording flagged).
- Enforcing/related components: A1 (LiteLLM), A7 (OPA), A9 (Headlamp), A8 (LibreChat), A21 (tenant onboarding), B13 (kopf operator / VirtualKey).
- ADR 0028 (identity federation chain; mapper split), 0016 (namespace tenancy), 0018 (RBAC-floor/OPA-restrictor), 0030 (versioning), 0037 (tenant onboarding).
