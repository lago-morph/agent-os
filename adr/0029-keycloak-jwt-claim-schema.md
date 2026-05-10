# ADR 0029: Required Keycloak JWT claim schema

- **Status**: Accepted
- **Date**: 2026-05-10

## Context

Multiple platform components — LiteLLM (auth and virtual-key claim binding), OPA (every identity-dependent policy decision), Headlamp (UI scoping), and LibreChat (endpoint visibility) — must make consistent authorization decisions based on the identity in a Keycloak-issued JWT. If each component invented its own claim names or expected shapes, OPA policies, gateway bindings, and UI gating logic would drift, and per-install customization of the upstream IdP (AD, Okta, etc.) would have to be re-mapped per consumer. ADR 0028 establishes that all privileged platform identity flows resolve to a Keycloak-issued JWT; this ADR fixes the contract that JWT must satisfy. Tenancy itself is namespace-based (ADR 0016) but is *established* through these claims — `platform_tenants`, `platform_namespaces`, `tenant_roles` — so the schema is also the multi-tenancy contract at the identity layer.

## Decision

The platform commits to the JWT claim schema specified in architecture-overview.md § 6.9 ("Required Keycloak JWT claims") as an architectural invariant. Every Keycloak-issued platform JWT MUST carry that schema (standard claims plus `platform_tenants`, `platform_namespaces`, `platform_roles`, `tenant_roles`, `capability_set_refs` with the types and required-ness specified there), and every platform component that consumes identity MUST consume only those claims. The schema is referenced from § 6.9 rather than duplicated here so there is a single source of truth.

## Consequences

- Every privileged platform identity — human user or service-account principal — carries this claim schema; this is a platform invariant (architecture-backlog.md § 6) and no platform component trusts upstream identity directly (architecture-overview.md § 6.11).
- Mapper authoring — translating upstream IdP attributes (AD groups, Okta groups, etc.) into the platform claim schema — is **out of scope** for this architecture and is the responsibility of whoever administers a specific install (architecture-backlog.md § 6, § 1.17). The architecture commits to the claim schema, not the mappers.
- The concrete `platform_roles` catalog (the specific role names such as `platform-admin`, `operator`, `developer`, `viewer`, `auditor`, and their semantics) is **design-time**, deferred per architecture-backlog.md § 1.2 and § 4. This ADR fixes only the *presence and shape* of the claim, not its enumerated values.
- OPA policies, LiteLLM virtual-key claim bindings, Headlamp plugin gating, and LibreChat endpoint visibility logic may all be authored against the § 6.9 names and types as a stable contract.
- Service-account principals (Platform Agents) carry `capability_set_refs`; OPA uses this to gate dynamic A2A/MCP registration and virtual-key issuance (architecture-overview.md § 6.9 visibility model).
- RBAC remains the authoritative permission floor (ADR 0018); these claims feed OPA, which may further restrict per-decision but cannot grant beyond RBAC.
- Any change to the schema (renaming a claim, changing a type, adding a required claim, removing one) is a breaking change to every consumer and MUST go through the CRD/API versioning discipline established in ADR 0030.

## References

- architecture-overview.md § 6.9 (Required Keycloak JWT claims)
- architecture-backlog.md § 4, 6
- ADR 0028 (identity federation), ADR 0016 (multi-tenancy), ADR 0030 (versioning)
