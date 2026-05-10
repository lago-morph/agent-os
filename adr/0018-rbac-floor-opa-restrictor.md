# ADR 0018: RBAC-as-floor / OPA-as-restrictor enforcement model platform-wide

- **Status**: Accepted
- **Date**: 2026-05-10

## Context

The platform layers two authorization mechanisms on every privileged action:
Kubernetes RBAC (plus Keycloak claims that mirror it for non-Kubernetes
surfaces) and OPA policy evaluated at admission and runtime. Without an
explicit contract between the two, reviewers cannot reason about whether a
given OPA rule is widening access beyond what RBAC granted, and component
authors cannot tell which layer to encode a given rule in. The architecture
needs a single, predictable composition rule so that every OPA decision point
in the platform — gateway callbacks, admission, egress, approvals, virtual key
issuance, capability access, dashboard visibility — has the same semantics.

## Decision

**RBAC is the permission ceiling; OPA may further restrict any decision; OPA
never grants permissions RBAC has not already given.** Every OPA decision
across the platform is evaluated against an identity whose permissions are
already bounded by Kubernetes RBAC and the Keycloak JWT claim schema, and OPA
may only return a result equal to or more restrictive than what RBAC permits.
"Restrict" includes raising the bar (e.g., elevating a required approval
level) and outright deny; it does not include any form of escalation.

## Consequences

- **Predictable security review story.** Reviewers can audit RBAC for the
  outer bound of what an identity can ever do, and audit OPA bundles for the
  per-decision narrowing rules, without worrying that an OPA rule silently
  widened a grant.
- **OPA callbacks at LiteLLM, Envoy egress, Gatekeeper admission, and the
  approval system all honor the same contract** — each is a restrictor over
  an RBAC-bounded identity, never a grantor. New OPA decision points added
  later inherit this rule by default.
- **Approval system (ADR 0017) relies on this model directly**: OPA can only
  elevate the required approval level for an `Approval`, never lower it, and
  never make an action approvable by someone RBAC would not allow to act.
- **Virtual key issuance and capability access** follow the same rule —
  OPA narrows what RBAC-permitted requestors may issue or bind, but cannot
  hand out a CapabilitySet binding the requestor's RBAC did not already
  authorize.
- **HolmesGPT toolsets honor RBAC ceilings**: even when an OPA policy
  permits an action, the underlying RBAC of HolmesGPT's service account
  remains the upper bound, so policy edits cannot accidentally hand the
  self-management agent broader access than its ServiceAccount holds.
- **Component authors get a clear authoring rule**: encode the broadest
  permissible permission in RBAC; encode contextual narrowing (time, spend,
  data sensitivity, tenant relationship, request shape) in OPA. Do not try
  to express grants in OPA.
- **Multi-tenancy (ADR 0016) composes cleanly**: namespace-scoped RBAC
  defines the tenant boundary, and OPA layers cross-tenant restrictions on
  top without ever being able to bridge tenants that RBAC has separated.

## References

- architecture-overview.md § 6.6
- architecture-backlog.md § 6
- ADR 0002 (OPA + Gatekeeper), ADR 0017 (Approval system), ADR 0016 (multi-tenancy)
