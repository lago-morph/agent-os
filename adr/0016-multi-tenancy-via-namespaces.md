# ADR 0016: Multi-tenancy via Kubernetes namespaces with RBAC + OPA + network enforcement

- **Status**: Accepted
- **Date**: 2026-05-10

## Context

The platform must isolate tenants — their Platform Agents, CRDs, Sandboxes, audit data, and observability — while still allowing controlled cross-tenant discovery and collaboration. Kubernetes already provides namespaces as a natural unit of scoping for the resources the platform reconciles, and the platform already depends on RBAC, OPA + Gatekeeper (ADR 0002), and an Envoy egress proxy (ADR 0003) for authorization and traffic control. Identity is federated through Keycloak (ADR 0028) with a committed JWT claim schema (ADR 0029) that already names tenants, namespaces, and roles. A first-class tenancy abstraction layered on top of these would duplicate primitives the platform already relies on, without adding isolation strength.

## Decision

Tenancy is a Kubernetes namespace: a tenant maps to one or more namespaces, and the namespace boundary is the tenancy boundary for all Platform Agents, referenced CRDs, Sandboxes, and emitted audit/observability data. Isolation is enforced defense-in-depth across four layers: (1) Kubernetes **RBAC** as the permission floor, (2) **Gatekeeper admission** on Agent and related CRDs, (3) **OPA runtime authorization** at the LiteLLM gateway on every A2A/MCP call (including dynamic agent registration), and (4) **network policy** at the Envoy egress proxy so that even a gateway-authorization bypass cannot reach the callee. Cross-namespace exposure is governed by three explicit visibility modes per overview §6.9: **invisible** outside the namespace (default), **read-only-visible** to all other tenants, or **read-write-visible** to specific other tenants by RBAC + OPA policy.

## Consequences

- Tenant identity is not a platform-specific construct; it is carried as Keycloak JWT claims (`platform_tenants`, `platform_namespaces`, `platform_roles`, `tenant_roles`, `capability_set_refs`) consumed uniformly by LiteLLM, OPA, Headlamp, and LibreChat. See ADR 0029.
- Dynamic agent registration (Platform Agents advertising A2A/MCP interfaces to LiteLLM at startup) is governed by the same OPA policy that gates cross-namespace calls, so a registration grant and a call grant are decided by the same engine on the same claims.
- Cross-tenant resource sharing (e.g., a `CapabilitySet` defined in one namespace and referenced from another) is allowed only when explicitly published with an OPA-checked policy; the default for every resource is namespace-private.
- The four enforcement layers are independent: a misconfiguration in one (e.g., an over-broad RBAC grant) is still bounded by the others (Gatekeeper admission, OPA gateway decision, network policy). This composes with the platform-wide RBAC-as-floor / OPA-as-restrictor model — see ADR 0018.
- Headlamp provides an admin override path (force-register / force-deregister), which is itself OPA-gated and audit-emitting; tenant administrators do not gain implicit cross-tenant powers.
- Sub-decisions are deferred per architecture-backlog.md § 1.2: tenant onboarding flow (who creates a tenant, what gets provisioned), tenant-scoped resource quotas (agents, sandboxes, virtual keys, cost, rate-limit) and their defaults, concrete cross-tenant collaboration patterns when explicitly published, and the concrete `platform_roles` catalog.
- Multi-cluster federation is explicitly out of scope (ADR 0026); each cluster install carries its own tenant set.

## References

- architecture-overview.md § 6.9
- architecture-backlog.md § 1.2, § 6
- ADR 0018 (RBAC-floor / OPA-restrictor), ADR 0028 (identity federation), ADR 0029 (Keycloak JWT claim schema)
