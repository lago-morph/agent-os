# ADR 0037: Tenant onboarding via Headlamp + a TenantOnboarding XRD

- **Status**: Accepted
- **Date**: 2026-05-10

## Context

Tenancy is built around Kubernetes namespaces (architecture-overview § 6.9,
ADR 0016): a tenant maps to one or more namespaces, and membership is
carried as Keycloak JWT claims (`platform_tenants`, `platform_namespaces`,
`tenant_roles`) per ADR 0029. The architecture commits to *what* a tenant
looks like at runtime, but the **onboarding flow** — "who creates a tenant,
what gets provisioned automatically" — is deferred in architecture-backlog
§ 1.2; the concrete `platform_roles` catalog is similarly deferred (§ 4).

Without a defined flow, tenant state is not in Git (so it cannot be
reviewed, audited, or rolled back like the rest of the platform), and
operators have no single surface for the provisioning steps a tenant needs
(namespace, default ServiceAccounts, claim mapping). The Git-as-source-of-
truth invariant plus the Crossplane convention for declarative
reconciliation point at modelling the tenant itself as a GitOps resource.

## Decision

A new Crossplane **`TenantOnboarding` XRD** is the GitOps representation of
a tenant. Reconciling an instance creates:

- **The tenant's Kubernetes namespace**, labelled with platform tenancy
  metadata so OPA, Gatekeeper, and observability scoping pick it up.
- **The templated default ServiceAccounts** the platform expects per tenant
  (the baseline set needed to host Platform Agents, run Sandboxes, and emit
  audit events).
- **A record of which Keycloak claim values map to that tenant** — i.e., the
  cluster-OIDC-side mapper that resolves the platform's claim schema
  (ADR 0029) into this tenant's `platform_tenants` / `platform_namespaces`
  values. Per the updated ADR 0028, **upstream-IdP mapper authoring remains
  out of scope** and is the install administrator's responsibility; only the
  cluster-OIDC-side mapper is owned by the composition.

The XRD **does not create CapabilitySets**. Tenant identity (who exists,
what namespaces they own) and capability bundles (what tools / models /
memory stores an agent may use) stay decoupled, consistent with ADR 0013
layering and the § 6.9 visibility model. The Headlamp onboarding plugin
*recommends* deploying at least one CapabilitySet at onboarding so a new
tenant is usable from day one, but this is a **UX prompt only**; there is
no code-level coupling between `TenantOnboarding` and any CapabilitySet,
and a tenant can be created without one.

The **Headlamp plugin** (ADR 0039) is the primary onboarding surface.
Edits flow as Git PRs that ArgoCD applies; there are no direct cluster
writes from the plugin, matching the GitOps invariant. The plugin renders
the form, validates against the XRD schema, proposes the claim-mapping
record, and opens the PR.

**Tenant offboarding** is the inverse: deleting the `TenantOnboarding`
resource reconciles namespace teardown and ServiceAccount cleanup, archives
the audit references (audit data itself is retained per the deferred
retention policy, architecture-backlog § 1.13), and notifies the operator
through the standard Headlamp suggestion-card / Mattermost channel.
Offboarding does not delete CapabilitySets the tenant referenced — the two
stay decoupled in both directions.

## Consequences

- Tenant existence becomes a Git-tracked, ArgoCD-applied artifact with the
  same review, rollback, and audit properties as other platform resources.
- Onboarding is one operator gesture in Headlamp (form → PR → merge → apply)
  instead of a runbook of manual `kubectl` and Keycloak-admin steps.
- Identity and capability stay decoupled by construction: a tenant can exist
  with zero CapabilitySets, and a CapabilitySet can be promoted across
  tenants without rewriting tenancy state.
- The cluster-OIDC-side mapper becomes a composition-owned artifact;
  upstream-IdP mapper authoring remains out of scope (ADR 0028, ADR 0029).
- Offboarding is symmetric and auditable; archived audit references survive
  tenant deletion until the (deferred) retention policy expires them.
- Defers, but does not resolve, tenant-scoped quotas (max agents, sandboxes,
  virtual keys, cost budget, rate limits) — those remain in
  architecture-backlog § 1.2 and will compose onto the XRD when designed.
- Defers the concrete `platform_roles` catalog (architecture-backlog § 4);
  the XRD records claim-value-to-tenant mapping, not role names.

## References

- architecture-overview.md § 6.9, § 6.11
- architecture-backlog.md § 1.2, § 4
- ADR 0013 (Capability CRD model and CapabilitySet layering)
- ADR 0016 (Multi-tenancy via Kubernetes namespaces)
- ADR 0028 (Identity federation chain)
- ADR 0029 (Required Keycloak JWT claim schema)
- ADR 0039 (Headlamp graphical editors for platform CRDs)
