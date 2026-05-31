# ADR 0049: Tenant quota model

- **Status**: Accepted
- **Date**: 2026-05-31

## Context

ADR 0037 introduced the `TenantOnboarding` XRD as the GitOps representation of a
tenant, but deliberately **deferred** tenant-scoped quotas (max agents, sandboxes,
virtual keys, cost budget, rate limits) to architecture-backlog § 1.2 — quotas
were intentionally absent from the schema, to be composed on later. That deferral
left a tenant onboardable with no resource ceiling at all: nothing in the platform
prevented a single namespace from exhausting cluster CPU or memory, and because the
ceiling was optional it was easy to never set one.

Tenancy is a Kubernetes namespace (ADR 0016), and the platform already enforces
admission defense-in-depth through Gatekeeper (ADR 0002). The substrate is being
modelled as Crossplane v2 composite resources (XRs) rather than the legacy
claim/XRC split (ADR 0044): `AgentEnvironment` is the XR that carries
`region` / `quotas` / `defaultCapabilitySetRef`, owned by the composition that
emits the namespace's enforcement objects. Those two facts make native Kubernetes
quota enforcement the obvious mechanism — `ResourceQuota` and `LimitRange` are
already understood by the API server, and the Crossplane v2 composition can emit
them as managed resources with no new runtime component.

The open question was therefore not *how* to enforce, but *whether quota is
mandatory* and *what schema* the tenant declares it in. This ADR records the
resolved answer.

## Decision

**Quota is a required field on `TenantOnboarding` at creation.** This **reverses**
ADR 0037's deferral of quotas: a tenant can no longer be onboarded without stating
its quota posture.

- **Enforcement mechanism.** v1 ships **native Kubernetes CPU and memory limits
  per namespace**, enforced via `ResourceQuota` / `LimitRange` objects emitted by
  the Crossplane v2 composition. Admission denies over-limit workloads at the
  Kubernetes API server; there is no separate quota runtime. (The
  `AgentEnvironment` XR is the Crossplane v2 composite resource carrying `quotas`;
  per-tenant quota itself lives on `TenantOnboarding`. Crossplane v2 only — there
  is no claim.)

- **Extensible typed-list schema.** `spec.quotas` is a **list of entries**, each
  with a `quotaType` discriminator plus type-specific fields — the same shape as a
  Kubernetes typed list or a Pod's `container.env`. v1 defines the `cpu` and
  `memory` quota types; new quota types (sandboxes, virtual keys, cost budget,
  rate limits) can be added later as additional entries with no breaking change to
  the schema or to existing tenant manifests.

- **Admission rule — mandatory presence, not mandatory limit.** The
  `TenantOnboarding` admission controller (OPA Gatekeeper, ADR 0002) MUST verify
  that a `cpu` entry **and** a `memory` entry are both present and **positively
  stated**. Each value is either a concrete limit **or** an explicit
  `unlimited: true`. **Omission is rejected**; an unbounded tenant is allowed only
  when `unlimited` is stated outright. The intent is that "no ceiling" is always a
  deliberate, reviewable choice in the manifest, never an accident of leaving a
  field blank.

## Consequences

- Every tenant has a stated CPU and memory posture at creation. The combination of
  a required field plus the presence-not-value admission rule means there is no
  path to an accidentally-unbounded tenant: the only way to be unlimited is to
  declare it, which is visible in the Git PR and auditable.
- Enforcement reuses primitives already in the cluster (`ResourceQuota`,
  `LimitRange`, Gatekeeper admission, the Crossplane v2 composition) — no new
  runtime component and no custom quota controller for v1.
- The typed-list schema is the extension seam: the deferred quota dimensions from
  ADR 0037 / architecture-backlog § 1.2 (sandboxes, virtual keys, cost budget,
  rate limits) land as new `quotaType` values without migrating existing tenants.
- Two enforcement points exist by design and must stay consistent: Gatekeeper
  validates the *declaration* on `TenantOnboarding` at admission; the Kubernetes
  API server enforces the *resulting* `ResourceQuota` / `LimitRange` on workloads
  at runtime. The composition is responsible for translating a declared limit (or
  `unlimited`) into the correct emitted objects.
- This narrows ADR 0037: the quota deferral recorded there is superseded by this
  ADR for CPU and memory; the remaining quota dimensions stay deferred but now have
  a defined place to land.

## References

- [ADR 0037](./0037-tenant-onboarding-xrd.md) (TenantOnboarding XRD — this ADR reverses its deferral of quotas)
- [ADR 0016](./0016-multi-tenancy-via-namespaces.md) (multi-tenancy via Kubernetes namespaces — the namespace is the quota boundary)
- [ADR 0002](./0002-opa-gatekeeper-policy-engine.md) (OPA + Gatekeeper admission — enforces mandatory quota presence on TenantOnboarding)
- [ADR 0044](./0044-crossplane-v2-resource-model.md) (Crossplane v2 resource model — the AgentEnvironment XR carrying `quotas`)
- [ADR 0052](./0052-policy-bundle-governance.md) (policy-bundle governance — the Gatekeeper bundle that carries the quota-presence constraint)
- [architecture-backlog.md](../architecture-backlog.md) [§ 1.2](../architecture-backlog.md#12-multi-tenancy-details) (deferred quota dimensions that land as future `quotaType` values)
