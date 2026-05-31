# SPEC ADR-0037 — Tenant onboarding via Headlamp + a TenantOnboarding XRD `[PROPOSED]`

> kind: ADR · workstream: — · tier: T1
> upstream: [B4; A9; A7] · downstream: [A21; A22; B4] · adrs: [0037] · views: [6.9]
> canon-glossary: FROZEN · canon-interface: FROZEN

## 1. Purpose & Problem Statement

ADR 0037 is a settled decision: a tenant is represented in GitOps by a Crossplane **`TenantOnboarding`
XRD**, and **Headlamp** (ADR 0039) is the primary onboarding surface. Reconciling an instance creates
the tenant's namespace, the templated default ServiceAccounts, and a record of the cluster-OIDC-side
claim mapping — and deliberately **does not create CapabilitySets** (identity and capability stay
decoupled). This SPEC states what honoring the decision requires of the XRD, the Headlamp plugin, and
offboarding — it does not re-argue the GitOps-tenant model.

The problem the decision solves: tenancy is namespace-based with membership carried as Keycloak claims
(ADR 0016 / 0029), but the onboarding flow was deferred (architecture-backlog §1.2). Without it, tenant
state is not in Git (not reviewable/auditable/rollback-able) and operators have a manual `kubectl` +
Keycloak runbook. Modelling the tenant as a Crossplane resource makes it a Git-tracked artifact.

## 2. Scope

### 2.1 In scope
- The `TenantOnboarding` XRD as the GitOps representation of a tenant; reconciling it creates namespace + default ServiceAccounts + the cluster-OIDC-side claim-mapping record.
- The decoupling rule: the XRD does NOT create CapabilitySets; a tenant can exist with zero CapabilitySets.
- Headlamp as the primary onboarding surface: form → schema-validate → propose claim-mapping → Git PR → ArgoCD apply (no direct cluster writes).
- Symmetric offboarding: deleting the resource reconciles namespace teardown + ServiceAccount cleanup + archives audit references + operator notification.

### 2.2 Out of scope (and where it lives instead)
- Upstream-IdP mapper authoring — install-administrator responsibility (ADR 0028); only the cluster-OIDC-side mapper is composition-owned.
- The concrete `platform_roles` catalog — deferred (architecture-backlog §4); the XRD records claim-value→tenant mapping, not role names.
- The per-resource detail of individual quota entries beyond the mandatory `cpu`/`memory` presence rule (additional resource classes such as max agents, sandboxes, virtual keys, budget, rate limits) — deferred (architecture-backlog §1.2); compose onto the XRD when designed. Note: `spec.quotas` *presence* is NOT deferred — it is mandatory (REQ-09/10 below), reversing the earlier deferral.
- The Headlamp graphical-editor framework itself — owned by ADR 0039 / components A9, A22.
- Audit retention of archived references — deferred to Workstream F (architecture-backlog §1.13).
- The tenant-onboarding reconciler/plugin component builds — owned by components A21 (reconciler) and A22/B5 (plugin).

## 3. Context & Dependencies

Upstream consumed: B4 (Crossplane Compositions — composes the `TenantOnboarding` XRD); A9 (Headlamp
framework — hosts the onboarding plugin); A7 (OPA/Gatekeeper — picks up tenancy-labelled namespaces
for scoping/admission). Downstream consumers: A21 (tenant onboarding reconciler), A22 (Headlamp
onboarding/editor surface), B4 (owns the Composition).

ADR decisions honored:
- **ADR 0037** (this): `TenantOnboarding` XRD as GitOps tenant; Headlamp primary surface; identity/capability decoupled; symmetric offboarding.
- **ADR 0016**: a tenant maps to one or more Kubernetes namespaces.
- **ADR 0029**: the cluster-OIDC-side mapper resolves the platform claim schema (`platform_tenants` / `platform_namespaces`) into this tenant's values.
- **ADR 0028**: upstream-IdP mapper authoring is out of scope (install-admin's responsibility); only the cluster-OIDC-side mapper is composition-owned.
- **ADR 0013**: identity (who exists / what namespaces) and capability bundles stay decoupled; no code-level coupling to any CapabilitySet.
- **ADR 0039**: the Headlamp plugin renders the form, validates against the XRD schema, proposes the claim-mapping record, and opens the PR — no direct cluster writes.
- **ADR 0030**: the XRD is versioned (conversion webhooks + deprecation windows) like every XRD.

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
- **`TenantOnboarding` (XRD)**, namespaced. Source-stated fields: `tenantId`, `namespaces[]`,
  `defaultServiceAccounts[]`, `spec.quotas`, `clusterOIDCClaimMapping`. `spec.quotas` is a **required
  extensible typed list** of quota entries; the `TenantOnboarding` admission controller MUST verify a
  `cpu` and a `memory` entry are present (each value is either a concrete limit or `unlimited: true`);
  omission of either is rejected at admission. Reconciling an instance creates: the tenant's
  Kubernetes namespace (labelled with platform tenancy metadata so OPA/Gatekeeper/observability scoping
  pick it up), the templated default ServiceAccounts (baseline to host Platform Agents, run Sandboxes,
  emit audit events), and a record of which Keycloak claim values map to that tenant (the cluster-OIDC-side
  mapper resolving ADR 0029 claims). **CapabilitySets are intentionally not coupled.** Versioned per ADR 0030.

### 4.2 APIs / SDK surfaces
- The **Headlamp onboarding plugin** is the primary surface: it renders the form, validates against the
  XRD's OpenAPI schema, proposes the claim-mapping record, and opens a Git PR that ArgoCD applies. No
  direct cluster writes from the plugin. The plugin *recommends* (UX prompt only) deploying at least one
  CapabilitySet at onboarding, but there is no code-level coupling and a tenant can be created without one.
- Concrete plugin/reconciler method signatures are **not specified in source** `[PROPOSED — not in source]`.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
- Tenant onboarding / namespace-association changes fall under **`platform.tenant.*`**. Offboarding
  notification is delivered through the standard Headlamp suggestion-card / Mattermost channel (ADR 0036).
  Concrete per-event-type names deferred to B12. `[PROPOSED — not in source]` whether the reconciler
  itself emits these vs. relying on standard lifecycle emission.

### 4.4 Data schemas / connection-secret contracts
N/A — no connection-secret surface. The `clusterOIDCClaimMapping` field records the claim-value→tenant
mapping (not role names). Audit references are archived on offboarding; audit *data* is retained per the
deferred retention policy.

## 5. OSS-vs-Custom Decision
N/A — ADR. The realizing component A21 builds the reconciler over Crossplane (B4) Compositions; the
Headlamp plugin (A22/B5) is custom TypeScript/React on the A9 framework. Those OSS-vs-custom calls live
in the A21 / A22 SPECs.

## 6. Functional Requirements

- REQ-ADR-0037-01: A tenant MUST be represented in GitOps by a `TenantOnboarding` XRD instance, applied by ArgoCD, with the same review/rollback/audit properties as other platform resources.
- REQ-ADR-0037-02: Reconciling a `TenantOnboarding` instance MUST create the tenant's namespace labelled with platform tenancy metadata so OPA, Gatekeeper, and observability scoping pick it up.
- REQ-ADR-0037-03: Reconciling MUST create the templated default ServiceAccounts (the baseline set to host Platform Agents, run Sandboxes, and emit audit events).
- REQ-ADR-0037-04: Reconciling MUST create a record of the cluster-OIDC-side claim mapping resolving the ADR 0029 claim schema into this tenant's `platform_tenants` / `platform_namespaces` values; upstream-IdP mapper authoring MUST remain out of scope (install-admin responsibility).
- REQ-ADR-0037-05: The XRD MUST NOT create CapabilitySets; identity and capability bundles MUST stay decoupled — a tenant MUST be creatable with zero CapabilitySets.
- REQ-ADR-0037-06: The Headlamp plugin MUST be the primary onboarding surface and MUST submit changes as Git PRs (form → schema-validate → propose claim-mapping → PR); there MUST be no direct cluster writes from the plugin.
- REQ-ADR-0037-07: The plugin MAY recommend deploying at least one CapabilitySet at onboarding as a UX prompt only, with no code-level coupling between `TenantOnboarding` and any CapabilitySet.
- REQ-ADR-0037-08: Offboarding MUST be the inverse: deleting the resource reconciles namespace teardown + ServiceAccount cleanup, archives audit references, and notifies the operator via the standard Headlamp suggestion-card / Mattermost channel; offboarding MUST NOT delete CapabilitySets the tenant referenced.
- REQ-ADR-0037-09: `TenantOnboarding.spec.quotas` MUST be a required extensible typed list (mandatory presence, reversing the earlier deferral); a `TenantOnboarding` with no `spec.quotas` MUST be rejected at admission.
- REQ-ADR-0037-10: The `TenantOnboarding` admission controller MUST verify that `spec.quotas` contains both a `cpu` and a `memory` entry; each entry's value MUST be either a concrete limit or `unlimited: true`; omission of either entry MUST be rejected.

## 7. Non-Functional Requirements
- Security/multi-tenancy (§6.9): tenancy-labelled namespaces drive OPA/Gatekeeper/observability scoping; identity and capability decoupled by construction in both directions.
- Auditability: tenant existence and offboarding are Git-tracked, ArgoCD-applied, auditable artifacts; archived audit references survive deletion until the deferred retention policy expires them.
- GitOps invariant: no direct cluster writes from the plugin — Git is the source of truth.
- Versioning (ADR 0030): `TenantOnboarding` XR schema changes are versioning events; quotas are a required part of the shape (REQ-07/08).

## 8. Cross-Cutting Deliverable Checklist
N/A — ADR. The realizing components A21 (reconciler) and A22/B5 (Headlamp plugin) carry the §14.1
deliverables (Composition, plugin, runbook, audit emission, 3-layer tests); conformance to this
decision is verified per §9.

## 9. Acceptance Criteria

- AC-ADR-0037-01: Honored when applying a `TenantOnboarding` instance via Git/ArgoCD produces the tenant's namespace, default ServiceAccounts, and the cluster-OIDC claim-mapping record. (→ REQ-01, REQ-02, REQ-03, REQ-04)
- AC-ADR-0037-02: Honored when the created namespace carries platform tenancy labels and an OPA/observability scope is shown to pick it up. (→ REQ-02)
- AC-ADR-0037-03: Honored when a tenant is created and usable (namespace + SAs present) with zero CapabilitySets, and no CapabilitySet is created by the reconciliation. (→ REQ-05)
- AC-ADR-0037-04: Honored when a Headlamp onboarding submission produces a Git PR (not a cluster write), validated against the XRD schema, with a proposed claim-mapping record. (→ REQ-06)
- AC-ADR-0037-05: Honored when the CapabilitySet recommendation is shown to be a dismissible UX prompt with no effect on whether the tenant is created. (→ REQ-07)
- AC-ADR-0037-06: Honored when deleting a `TenantOnboarding` instance tears down the namespace + ServiceAccounts, archives audit references, notifies the operator, and leaves referenced CapabilitySets intact. (→ REQ-08)
- AC-ADR-0037-07: Honored when applying a `TenantOnboarding` with no `spec.quotas` is rejected at admission, and one with a populated `spec.quotas` is admitted. (→ REQ-09)
- AC-ADR-0037-08: Honored when a `TenantOnboarding` whose `spec.quotas` omits the `cpu` or `memory` entry is rejected at admission, while one carrying both (each a concrete limit or `unlimited: true`) is admitted. (→ REQ-10)

## 10. Risks & Open Questions
- (med) Quota *presence* is now mandatory (REQ-09/10): `spec.quotas` must carry `cpu` and `memory` (concrete or `unlimited: true`) or admission rejects the tenant. Additional per-resource quota classes (max agents, sandboxes, virtual keys, budget, rate limits) remain deferred (architecture-backlog §1.2) and compose onto the list when designed.
- (low) The `platform_roles` catalog is deferred (architecture-backlog §4); the XRD records claim-value→tenant mapping, not role names.
- (low) `[PROPOSED]` — whether the reconciler emits `platform.tenant.*` events itself vs. relies on standard lifecycle emission, and plugin/reconciler method signatures, are not specified in source; flagged for A21 / B12 design.

## 11. References
- ADR 0037 (this decision). Enforcing/realizing components: A21 (tenant onboarding reconciler), A22/B5 (Headlamp onboarding plugin), B4 (`TenantOnboarding` Composition), A7 (tenancy-labelled namespace scoping).
- architecture-overview.md §6.9 (multi-tenancy/namespacing), §6.11 (identity federation). architecture-backlog.md §1.2 (multi-tenancy details, onboarding deferred), §4 (roles catalog deferred), §1.13 (retention deferred). ADR 0013, 0016, 0028, 0029, 0036, 0039.
