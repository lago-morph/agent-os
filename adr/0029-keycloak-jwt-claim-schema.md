# ADR 0029: Required Keycloak JWT claim schema

- **Status**: Accepted
- **Date**: 2026-05-10

## Context

Multiple platform components — LiteLLM (auth and virtual-key claim binding), OPA (every identity-dependent policy decision), Headlamp (UI scoping), and LibreChat (endpoint visibility) — must make consistent authorization decisions based on the identity in a Keycloak-issued JWT. If each component invented its own claim names or expected shapes, OPA policies, gateway bindings, and UI gating logic would drift, and per-install customization of the upstream IdP (AD, Okta, etc.) would have to be re-mapped per consumer. ADR 0028 establishes that all privileged platform identity flows resolve to a Keycloak-issued JWT; this ADR fixes the contract that JWT must satisfy. Tenancy itself is namespace-based (ADR 0016) but is *established* through these claims — `platform_tenants`, `platform_namespaces`, `tenant_roles` — so the schema is also the multi-tenancy contract at the identity layer.

## Decision

The platform commits to the JWT claim schema specified in architecture-overview.md § 6.9 ("Required Keycloak JWT claims") as an architectural invariant. Every Keycloak-issued platform JWT MUST carry that schema (standard claims plus `platform_tenants`, `platform_namespaces`, `platform_roles`, `tenant_roles`, `capability_set_refs` with the types and required-ness specified there), and every platform component that consumes identity MUST consume only those claims. The schema is referenced from § 6.9 rather than duplicated here so there is a single source of truth.

## Consequences

- Every privileged platform identity — human user or service-account principal — carries this claim schema; this is a platform invariant (architecture-backlog.md § 6) and no platform component trusts upstream identity directly (architecture-overview.md § 6.11).
- Mapper authoring splits along the federation chain established in ADR 0028. The two sides have different scope decisions and must be reasoned about separately.
- **Upstream-IdP-side mappers** — translating upstream IdP attributes (Active Directory groups, Okta groups, SAML assertions, etc.) into the platform claim schema — are **out of scope** for this architecture.
  - These mappers are install-specific: the AD group layout at one operator's site has no relationship to another's, so the platform cannot ship a single canonical mapping.
  - Authoring and maintaining them remains the responsibility of whoever administers a specific install (architecture-backlog.md § 6, § 1.17).
  - The platform commits to the claim schema (the destination shape), not to the per-install upstream mappings (the translation rules).
- **Cluster-OIDC-side mappers** — translating Kubernetes ServiceAccount tokens minted by the in-cluster OIDC issuer into platform JWTs that satisfy this schema — are **IN scope**, per the revised ADR 0028.
  - Workload identity federation is uniform across installs (a ServiceAccount on EKS looks structurally identical to one on kind), so the platform *can* and therefore *must* ship the mapping.
  - The platform MUST ship working, tested mappers for the supported cluster distributions (kind, EKS, AKS) so that ServiceAccount-derived principals land in the same § 6.9 claim shape as human principals.
  - Without these, workload identity cannot reach the platform plane and the § 6.9 contract is unenforceable for service-account callers — OPA, LiteLLM, and Headlamp would all see malformed or empty principals.
- **Backlog-doc inconsistency flagged for follow-up:** architecture-backlog.md § 6 currently states the invariant as "Mapper authoring … is out of scope for this architecture" without distinguishing the two sides. That wording predates the ADR 0028 revision and now over-claims. It needs to be narrowed to "upstream-IdP mappers are out of scope; cluster-OIDC mappers are in scope and shipped per ADR 0028 / ADR 0029" in the second-pass non-ADR doc updates. Flagged here, not edited in this ADR.
- The concrete `platform_roles` catalog (the specific role names such as `platform-admin`, `operator`, `developer`, `viewer`, `auditor`, and their semantics) is **design-time**, deferred per architecture-backlog.md § 1.2 and § 4. This ADR fixes only the *presence and shape* of the claim, not its enumerated values.
- OPA policies, LiteLLM virtual-key claim bindings, Headlamp plugin gating, and LibreChat endpoint visibility logic may all be authored against the § 6.9 names and types as a stable contract.
- Service-account principals (Platform Agents) carry `capability_set_refs`; OPA uses this to gate dynamic A2A/MCP registration and virtual-key issuance (architecture-overview.md § 6.9 visibility model).
  - Because the cluster-OIDC-side mappers are now in scope, the mapping from a ServiceAccount's namespace + annotations to `platform_tenants` / `platform_namespaces` / `capability_set_refs` is an architectural commitment, not an install-time concern.
  - This mapping must be testable in the platform's own CI against the supported distributions, so a regression in the EKS mapper (for example) is caught upstream rather than at customer install time.
- The cluster-OIDC mapper commitment also means schema changes have a second blast radius:
  - In addition to breaking every claim consumer (LiteLLM, OPA, Headlamp, LibreChat), they break the platform-shipped mappers themselves and require coordinated updates across the kind, EKS, and AKS mapper bundles.
  - ADR 0030 versioning therefore covers the mapper artifacts in lockstep with the schema; mapper bundle versions are pinned to the schema version they target.
- Tenant onboarding (ADR 0037) is the moment at which a newly provisioned tenant's identifiers first appear in `platform_tenants` / `platform_namespaces` and tenant-scoped roles first appear in `tenant_roles`.
  - ADR 0037 depends on this ADR for the claim shape it must populate at onboarding time.
  - Changes to onboarding ergonomics MUST NOT push back against the § 6.9 contract — the contract is the fixed point and onboarding adapts to it, not vice versa.
- RBAC remains the authoritative permission floor (ADR 0018); these claims feed OPA, which may further restrict per-decision but cannot grant beyond RBAC.
- Any change to the schema (renaming a claim, changing a type, adding a required claim, removing one) is a breaking change to every consumer — including the platform-shipped cluster-OIDC mappers — and MUST go through the CRD/API versioning discipline established in ADR 0030.

## References

(Cross-references reflect the post-ADR-0028-revision split between in-scope cluster-OIDC mappers and out-of-scope upstream-IdP mappers; see Consequences for the architecture-backlog.md § 6 wording flagged for follow-up narrowing.)

- [architecture-overview.md § 6.9](../architecture-overview.md#69-multi-tenancy-and-namespacing) (Required Keycloak JWT claims)
- [architecture-backlog.md](../architecture-backlog.md) [§ 4](../architecture-backlog.md#4-topics-that-need-further-design-before-implementation), [6](../architecture-backlog.md#6-architecture-level-invariants-worth-documenting-as-adrs)
- [ADR 0028](./0028-identity-federation.md) (identity federation chain; defines cluster-OIDC → Keycloak mappers as in-scope and is the source of the upstream-vs-cluster mapper split this ADR adopts)
- [ADR 0037](./0037-tenant-onboarding-xrd.md) (tenant onboarding; consumes `platform_tenants` / `platform_namespaces` / `tenant_roles` from this schema as the binding moment for new tenants)
- [ADR 0016](./0016-multi-tenancy-via-namespaces.md) (namespace-based multi-tenancy; ADR 0029 is the identity-layer expression of the tenancy model ADR 0016 defines at the namespace layer)
- [ADR 0018](./0018-rbac-floor-opa-restrictor.md) (RBAC permission floor; OPA may further restrict but cannot grant beyond)
- [ADR 0030](./0030-crd-and-api-versioning-policy.md) (CRD/API versioning discipline; covers the schema and the platform-shipped cluster-OIDC mapper bundles in lockstep)
