# SPEC A21 — Tenant onboarding reconciler

> kind: COMPONENT · workstream: A · tier: T1
> upstream: [A9, A22] · downstream: [] · adrs: [0037, 0028, 0029, 0016, 0013, 0039, 0044, 0030, 0034] · views: [6.9, 6.11, 6.12]
> canon-glossary: see _meta/glossary.md · canon-interface: see _meta/interface-contract.md

## 1. Purpose & Problem Statement

Tenancy is built around Kubernetes namespaces (§6.9, ADR 0016): a tenant maps to one or more namespaces, and membership is carried as Keycloak JWT claims (`platform_tenants`, `platform_namespaces`, `tenant_roles`) per ADR 0029. The architecture commits to *what* a tenant looks like at runtime, but the onboarding flow — who creates a tenant and what gets provisioned automatically — was previously a runbook of manual `kubectl` and Keycloak-admin steps, leaving tenant state outside Git (so it could not be reviewed, audited, or rolled back like the rest of the platform).

A21 delivers the **`TenantOnboarding` Crossplane XRD as the GitOps representation of a tenant** (ADR 0037), its **composition** (creates the tenant namespace(s) labelled with platform tenancy metadata, the templated default ServiceAccounts wired to the cluster OIDC issuer, and the cluster-OIDC-side claim mapper that resolves the platform claim schema into this tenant's `platform_tenants`/`platform_namespaces` values), and the **Headlamp graphical editor** that drives it (built on the A22 framework). Onboarding becomes one operator gesture: form → PR → merge → ArgoCD apply. Offboarding is the symmetric inverse: deleting the resource reconciles namespace teardown + ServiceAccount cleanup and archives audit references.

CapabilitySets are intentionally **not coupled** to onboarding (ADR 0013, ADR 0037): a tenant can exist with zero CapabilitySets, and a CapabilitySet can move across tenants without rewriting tenancy state. The Headlamp plugin *recommends* seeding at least one CapabilitySet but couples nothing in code.

## 2. Scope

### 2.1 In scope

- The **`TenantOnboarding` XRD** schema (Canon key fields: `tenantId`, `namespaces[]`, `defaultServiceAccounts[]`, `clusterOIDCClaimMapping`) and its versioning (ADR 0030).
- The **composition** reconciling a `TenantOnboarding` claim into: the tenant namespace(s) labelled with platform tenancy metadata (so OPA, Gatekeeper, observability scoping pick them up); templated default ServiceAccounts (developer SA, operator SA, agent-default SA, etc.) wired to the cluster OIDC issuer; the cluster-OIDC-side claim mapper recording which claim values map to this tenant.
- The **Headlamp graphical editor** for `TenantOnboarding`, built on the A22 framework — form, schema validation, claim-mapping proposal, diff-against-Git, PR submission, and the CapabilitySet-seeding *recommendation* (UX prompt only).
- **Offboarding**: deletion reconciles namespace teardown + ServiceAccount cleanup, archives audit references (audit data retained per the deferred retention policy), and notifies the operator via the standard Headlamp suggestion-card / Mattermost channel. Offboarding does **not** delete referenced CapabilitySets.
- Standard Workstream A deliverables (§14.1): manifests, docs, runbook, alerts, Grafana dashboard, OPA integration, audit emission, Knative trigger flow, HolmesGPT toolset, 3-layer tests, tutorials/how-tos.

### 2.2 Out of scope (and where it lives instead)

- **The XRD's Crossplane Composition *machinery* / substrate plumbing** — the `TenantOnboarding` XRD is one of the Crossplane Compositions **owned by B4** (§14.2 B4 lists `TenantOnboarding`). A21 owns the tenant-onboarding *component* (XRD intent, composition behaviour spec, editor, reconcile semantics); B4 owns the Crossplane v2 Composition implementation surface. Reconciliation: see §10 OQ-A21-1.
- **The editor framework / shared widgets** — A22. A21 builds the `TenantOnboarding` editor *on* A22's framework (§14.1 A21: "A21 owns its own CRD's editor; A22 owns only the cross-cutting framework").
- **CapabilitySet creation/lifecycle** — kopf (B13) + the capability components; decoupled by design (ADR 0013/0037).
- **Upstream-IdP mapper authoring** — out of scope; the install administrator's responsibility (ADR 0028/0029, §6.9). A21 owns only the **cluster-OIDC-side** mapper.
- **The `platform_roles` catalog** — deferred (architecture-backlog §4); the XRD records claim-value-to-tenant mapping, not role names (ADR 0037).
- **Tenant-scoped quotas** — the **#5/#9 ruling reverses the ADR-0037 deferral**: `cpu` and `memory` entries are now **mandatory at admission** (concrete value or explicit `unlimited: true`; see REQ-A21-13), enforced via generated `ResourceQuota` + `LimitRange`. `spec.quotas` is an extensible typed list (`quotaType` discriminator), so additional quota types (max agents, sandboxes, virtual keys, cost budget, rate limits) compose on additively later without a breaking version bump — but cpu+memory presence is no longer optional.
- **Audit retention / deletion of archived audit data** — Workstream F (F1); offboarding archives references but does not expire data.
- **Keycloak realm / client install** — A15/B1 SSO layer + the install administrator; A21 records the claim-to-tenant binding, it does not stand up Keycloak.

## 3. Context & Dependencies

**Upstream consumed:**

- **A9 (Headlamp framework)** — base install + auth handoff for the editor surface.
- **A22 (Headlamp graphical-editor framework)** — the schema-driven form generation, validation hooks, diff-against-Git, simulator-panel, and reference-picker widgets the `TenantOnboarding` editor is built on. (A22 also ships a baseline `TenantOnboarding` editor in its initial set — see §10 OQ-A21-2 for the A21/A22 boundary.)
- **B4 (Crossplane v2 Compositions)** — the Crossplane composition engine and the `TenantOnboarding` composition implementation surface (§14.2). A21 depends on B4 for the composition runtime (B4 is W1/foundation per waves.md).

**Downstream consumers:** none in the piece graph (CSV downstream empty). The tenant namespaces + ServiceAccounts it provisions are consumed at runtime by every Platform Agent / Sandbox / audit emission scoped to that tenant (§6.9), but those are not piece-level dependents.

**ADR decisions honored:**

- **ADR 0037** — `TenantOnboarding` XRD is the GitOps tenant representation; composition creates namespace + default SAs + cluster-OIDC-side mapper; **no CapabilitySet coupling** (UX prompt only); Headlamp+GitOps flow (PR-only, no direct cluster writes from the plugin); symmetric auditable offboarding; defers quotas + role catalog.
- **ADR 0028** — identity federation chain; **only the cluster-OIDC-side mapper is owned here**; upstream-IdP mapper authoring is the install admin's job.
- **ADR 0029** — the Keycloak claim schema the mapper resolves into (`platform_tenants`, `platform_namespaces`, `tenant_roles`).
- **ADR 0016** — multi-tenancy via namespaces with RBAC + OPA + network enforcement; the namespace is the tenancy boundary.
- **ADR 0013** — identity/capability decoupling.
- **ADR 0039 / A22** — the editor is a schema-driven Headlamp editor, PR-only.
- **ADR 0044** — XRD/Composition versioning + uniform contract apply identically to this XRD (conversion webhooks + deprecation windows).
- **ADR 0030** — per-component CRD/XRD versioning, owned by the reconciler owner (Crossplane B4 for XRs).
- **ADR 0034** — onboarding/offboarding actions emit audit through the platform adapter.

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)

**`TenantOnboarding`** (Crossplane XRD; reconciler: Crossplane/B4; scope: **namespaced**) — Canon key fields (§6.12, interface-contract §1.6):

| Field | Source-stated meaning |
|---|---|
| `tenantId` | The tenant's identity. |
| `namespaces[]` | The tenant namespace(s) to create (a tenant may map to more than one). |
| `defaultServiceAccounts[]` | Templated default ServiceAccounts (developer SA, operator SA, agent-default SA, etc.) wired to the cluster OIDC issuer. |
| `clusterOIDCClaimMapping` | The cluster-OIDC-side mapper resolving the platform claim schema into this tenant's `platform_tenants` / `platform_namespaces` entries. |

- Field sets beyond the four above are **not specified in source**; any addition (e.g. labels, SA role bindings detail) is `[PROPOSED — not in source]`.
- **Quotas, role-catalog fields are deliberately absent** (deferred, ADR 0037) — do not invent.
- Versioning per ADR 0030/0044: `v1alpha1`→`v1`; breaking changes via new `vN` group + conversion webhook; owner = Crossplane B4.
- **Composition outputs (source-stated):** tenant namespace(s) labelled with platform tenancy metadata; templated default ServiceAccounts; cluster-OIDC-side claim mapper record. (No connection secret — this XRD provisions no substrate primitive; see §4.4.)

### 4.2 APIs / SDK surfaces

- **Headlamp editor (provided)** — the `TenantOnboarding` graphical editor; a Headlamp plugin built on A22 widgets. Not an HTTP API. Submits PRs (GitHub, ADR 0033).
- **Crossplane XR API (consumed)** — the namespace-scoped `TenantOnboarding` XR is authored in Git and reconciled by Crossplane (B4). Per ADR 0044 (Crossplane v2) there is no claim layer — the user-facing form *is* the namespace-scoped XR.
- No custom HTTP service of A21's own; §6.13 URL-path versioning N/A.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)

Per-event-type names deferred to B12 (interface-contract §2); A21 commits to namespaces:

- **Emitted:** tenant onboarding / namespace-association-change / offboarding events under **`platform.tenant.*`** (the namespace covering "tenant onboarding, namespace association changes" — interface-contract §2). Audit-emission of onboarding/offboarding actions under **`platform.audit.*`** (ADR 0034); audit emission is gated on the audit-adapter freeze-gate (D-05). `[PROPOSED — not in source]` for any concrete event-type name.
- **Namespace ownership (QN-03):** A21 (the tenant-onboarding reconciler) is the **single owner of the `platform.tenant` namespace** — A21 authors and registers its schema in B12 (the event catalogue). Dashboards and other consumers take an explicit dependency on A21 as the owner and do not co-own it. For any security-relevant event A21 detects (privilege anomaly during onboarding), A21 MUST handle it locally AND additionally emit under `platform.security` (schema owned by A7).
- **Consumed:** none required for core reconcile (Crossplane reconciles the XR; the editor reads Git synchronously). The operator offboarding notification rides the standard Headlamp suggestion-card / Mattermost channel (ADR 0037) — an existing flow, not a new A21 trigger.

### 4.4 Data schemas / connection-secret contracts

- **N/A — connection secret:** `TenantOnboarding` provisions namespaces, ServiceAccounts, and a claim-mapper record — not a substrate-asymmetric primitive — so it writes **no** host/port/user/password/dbname connection secret (interface-contract §4 applies only to substrate XRDs). The ServiceAccounts are wired to the cluster OIDC issuer (token-based federation), not a connection secret.
- The cluster-OIDC-side claim mapper is a record of claim-value → tenant mapping (ADR 0037); its concrete storage shape is `[PROPOSED — not in source]` beyond the `clusterOIDCClaimMapping` field name.
- Consumes Keycloak claims (`platform_tenants`, `platform_namespaces`, `platform_roles`, `tenant_roles`) for editor scoping and the mapping it records (§6.9, ADR 0029).

## 5. OSS-vs-Custom Decision

- **Base:** Crossplane v2 (B4) for the composition runtime; Headlamp (A9) + A22 framework for the editor. Both OSS + platform-built layers.
- **Decision:** **build-new** — author the `TenantOnboarding` XRD + Composition (declarative Crossplane artifacts) and the editor plugin. No OSS project provides a tenant-as-GitOps-resource model with namespace + default SA + cluster-OIDC mapper provisioning; ADR 0037 mandates building it.
- **Rationale:** Modelling tenant existence as a Git-tracked Crossplane resource gives it the same review/rollback/audit properties as the rest of the platform (ADR 0037). The composition is declarative config over Crossplane (no new controller binary needed beyond Crossplane B4).

## 6. Functional Requirements

- **REQ-A21-01:** The `TenantOnboarding` XRD SHALL expose the Canon fields `tenantId`, `namespaces[]`, `defaultServiceAccounts[]`, `clusterOIDCClaimMapping` and SHALL be namespaced (§6.12, ADR 0037).
- **REQ-A21-02:** On reconcile, the composition SHALL create the tenant namespace(s) in `namespaces[]`, each labelled with platform tenancy metadata so OPA, Gatekeeper, and observability scoping pick them up (ADR 0037).
- **REQ-A21-03:** The composition SHALL create the templated default ServiceAccounts in `defaultServiceAccounts[]` (the baseline set to host Platform Agents, run Sandboxes, emit audit events), wired to the cluster OIDC issuer (ADR 0037, §6.9).
- **REQ-A21-04:** The composition SHALL record the cluster-OIDC-side claim mapper from `clusterOIDCClaimMapping`, resolving the platform claim schema into this tenant's `platform_tenants`/`platform_namespaces` values; it SHALL NOT author upstream-IdP mappers (ADR 0028/0029).
- **REQ-A21-05:** The XRD SHALL NOT create or reference any CapabilitySet; identity and capability stay decoupled in code (ADR 0013/0037).
- **REQ-A21-06:** Editing a `TenantOnboarding` resource SHALL flow through the Headlamp editor as a Git PR (ArgoCD applies); the plugin SHALL perform no direct cluster writes (ADR 0037/0039).
- **REQ-A21-07:** The Headlamp editor SHALL validate the form against the XRD schema, propose the claim-mapping record, render a diff-against-Git, and open the PR (ADR 0037/0039), building on the A22 framework widgets.
- **REQ-A21-08:** The editor SHALL recommend seeding at least one CapabilitySet at onboarding (offering a starter from the agent profile library) as a UX prompt only, with no code-level coupling to any CapabilitySet (ADR 0037).
- **REQ-A21-09:** Deleting a `TenantOnboarding` resource SHALL reconcile namespace teardown and ServiceAccount cleanup, archive the tenant's audit references, and notify the operator via the standard Headlamp suggestion-card / Mattermost channel (ADR 0037).
- **REQ-A21-10:** Offboarding SHALL NOT delete CapabilitySets the tenant referenced (decoupled in both directions) and SHALL NOT delete archived audit data (retained per the deferred retention policy) (ADR 0037).
- **REQ-A21-11:** Onboarding and offboarding actions SHALL emit structured audit events through the platform audit adapter (ADR 0034) and tenant lifecycle events under `platform.tenant.*` (ADR 0031).
- **REQ-A21-12:** XRD versioning SHALL follow ADR 0030/0044 (`v1alpha1`→`v1`, new `vN` group + conversion webhook for breaking changes, owner Crossplane B4).
- **REQ-A21-13 (mandatory quotas, #5/#9 — reverses the ADR-0037 deferral):** Quotas are **no longer optional/deferred**. The `TenantOnboarding` admission controller SHALL reject any onboarding whose quota specification omits a `cpu` entry or a `memory` entry; both MUST be present, each as either a concrete limit OR an explicit `unlimited: true` (omission is rejected). `spec.quotas` SHALL be an **extensible typed list** (like `container.env`) whose entries each carry a `quotaType` discriminator plus type-specific fields, and SHALL be enforced via native Kubernetes `ResourceQuota` + `LimitRange` generated by the composition. (The tenant-level `AgentEnvironment` XR carrying `region`/`quotas`/`defaultCapabilitySetRef` is owned by B4; A21's admission enforces the mandatory-presence rule.)
- **REQ-A21-14 (distinct RBAC granularity, #23):** A21 SHALL provision/seed distinct RBAC permissions at the proposed granularity — set-agent-type-default-budget vs per-instance-budget-override vs budget-maintainer, and policy-change vs policy-maintain. Field-level rules native RBAC cannot express SHALL be enforced via Gatekeeper (OPA) admission (A7).

## 7. Non-Functional Requirements

- **Security / multi-tenancy (§6.9):** The created namespace is the tenancy boundary; tenancy-metadata labels must be correct for OPA/Gatekeeper/observability scoping to function. The editor is claim-gated (only `platform-admin`/onboarding-capable roles may onboard). Default ServiceAccounts must carry least-privilege baseline (the set needed to host agents/sandboxes/audit), not broad grants — RBAC-as-floor (ADR 0018).
- **Observability (§6.5):** Onboarding/offboarding fully audited; Grafana dashboard surfaces tenant count + recent onboarding/offboarding; `LogLevel` (ADR 0035) toggles reconcile verbosity per-tenant.
- **GitOps integrity:** Tenant state is a Git-tracked artifact (ADR 0037); no out-of-band `kubectl`/Keycloak-admin path is sanctioned. Rollback = revert the PR / Crossplane reconcile, not manual cleanup.
- **Scale:** A tenant may map to multiple namespaces (`namespaces[]`); the composition must handle multi-namespace tenants idempotently.
- **Versioning (ADR 0030/0044):** XRD evolves per-component; role composition is additive and may land later without breaking existing tenants.
- **Quota enforcement (#5/#9 — reverses the ADR-0037 deferral):** quotas are **no longer optional/deferred**. The `TenantOnboarding` admission controller MUST verify that the tenant's quota specification carries **both a `cpu` entry AND a `memory` entry** (mandatory presence); each value is either a concrete limit OR an explicit `unlimited: true`, and omission of either entry is **rejected** at admission. `spec.quotas` is an **extensible typed list** (like `container.env`): each entry has a `quotaType` discriminator plus type-specific fields. Quotas are enforced via native Kubernetes `ResourceQuota` + `LimitRange` generated by the composition. (The tenant-level `AgentEnvironment` XR that carries `region`/`quotas`/`defaultCapabilitySetRef` is owned by B4; A21's admission enforces the mandatory-presence rule above.)
- **Symmetry:** Offboarding must be the auditable inverse of onboarding; archived audit references survive tenant deletion until the deferred retention policy expires them.

## 8. Cross-Cutting Deliverable Checklist

Per §14.1:

- **Helm values / manifests in Git** — applicable: XRD + Composition manifests + editor plugin packaging.
- **Per-product architecture docs (10.5)** — applicable: tenant onboarding model + claim-mapping reference.
- **Per-product operator runbook (10.7)** — applicable: onboard a tenant, offboard a tenant, multi-namespace tenant, recover a failed reconcile.
- **Backup / restore** — applicable (light): tenant definitions live in Git (restore = re-apply); archived audit references are retained per F1.
- **Alert rules** — applicable: onboarding/offboarding reconcile failure, orphaned-namespace detection, claim-mapper write failure.
- **Grafana dashboard (Crossplane `GrafanaDashboard` XR)** — applicable: tenant inventory, onboarding/offboarding rate + failures.
- **Headlamp plugin** — applicable: the `TenantOnboarding` editor (built on A22).
- **OPA / Rego integration** — applicable: admission policy gating who may create/delete `TenantOnboarding`, and namespace-label conformance Rego contributed to the policy library (B3/B16).
- **Audit emission (ADR 0034)** — applicable: onboarding + offboarding actions (REQ-A21-11).
- **Knative trigger flow** — applicable: `platform.tenant.*` lifecycle events emitted; the offboarding operator notification rides the existing Headlamp suggestion-card / Mattermost channel (ADR 0037) rather than a new A21-owned trigger. (No new consumer trigger required in v1.0.)
- **HolmesGPT toolset** — applicable: "list tenants / tenant namespaces / recent onboarding activity" read tool.
- **3-layer tests (Chainsaw / Playwright / PyTest)** — applicable: Chainsaw for XRD reconcile (namespace + SA + mapper creation, offboarding teardown), Playwright for the editor, PyTest for claim-mapping logic.
- **Tutorials & how-tos** — applicable: "Onboard a tenant" how-to; tutorial coverage of the onboarding gesture.

## 9. Acceptance Criteria

- **AC-A21-01** (REQ-A21-01): A `TenantOnboarding` claim accepts `tenantId`, `namespaces[]`, `defaultServiceAccounts[]`, `clusterOIDCClaimMapping` and is rejected if applied cluster-scoped (namespaced only).
- **AC-A21-02** (REQ-A21-02): Reconciling a claim with two entries in `namespaces[]` creates both namespaces, each carrying the platform tenancy-metadata labels.
- **AC-A21-03** (REQ-A21-03): Reconcile creates each ServiceAccount in `defaultServiceAccounts[]`, federated to the cluster OIDC issuer.
- **AC-A21-04** (REQ-A21-04): Reconcile records the cluster-OIDC-side claim mapper from `clusterOIDCClaimMapping`; no upstream-IdP mapper is created.
- **AC-A21-05** (REQ-A21-05): Onboarding a tenant creates zero CapabilitySets; the tenant exists and is usable for namespace-scoped operations without one.
- **AC-A21-06** (REQ-A21-06): An editor submission opens a Git PR and performs no direct cluster write (API-server audit shows no write from the plugin identity).
- **AC-A21-07** (REQ-A21-07): The editor blocks a schema-invalid `TenantOnboarding` before PR creation and shows a diff-against-Git for a valid one.
- **AC-A21-08** (REQ-A21-08): The editor shows the CapabilitySet-seeding recommendation; declining it still produces a valid onboarding PR with no CapabilitySet reference.
- **AC-A21-09** (REQ-A21-09): Deleting the resource tears down the tenant namespace(s) and ServiceAccounts, archives audit references, and posts the operator notification.
- **AC-A21-10** (REQ-A21-10): After offboarding, CapabilitySets the tenant referenced still exist and archived audit data is retained.
- **AC-A21-11** (REQ-A21-11): Onboarding emits a `platform.audit.*` event via the adapter and a `platform.tenant.*` lifecycle event; offboarding does likewise.
- **AC-A21-12** (REQ-A21-12): A `v1alpha1`→`v1` XRD bump applies with a conversion webhook keeping existing stored tenants readable.

## 10. Risks & Open Questions

- **OQ-A21-1** (blast-radius: med): Ownership split — §14.2 lists `TenantOnboarding` under **B4** (Crossplane Compositions) while §14.1 A21 owns "the `TenantOnboarding` XRD, its composition … and the Headlamp editor." **Reconciliation note:** A21 owns the tenant-onboarding *component intent + composition behaviour spec + editor*; B4 provides the Crossplane composition *runtime/engine* and co-authors the Composition artifact. Confirm with B4 owner that the composition manifest lands in A21's deliverable set. `[PROPOSED]` boundary.
- **OQ-A21-2** (med): A22 ships a baseline `TenantOnboarding` editor in its ADR 0039 initial set, but §14.1 says A21 owns its CRD's editor. **Reconciliation note:** A21's editor is built on the A22 framework and supersedes/extends A22's baseline with tenant-specific validation (claim-mapping proposal, multi-namespace handling). Coordinate so the editor is not double-built. `[PROPOSED]` boundary.
- **OQ-A21-3** (med): The `clusterOIDCClaimMapping` storage/representation is not field-detailed in source (only the field name + intent). Concrete shape is `[PROPOSED — not in source]`; depends on the Keycloak/cluster-OIDC integration owned by the install admin + B1.
- **OQ-A21-4** (low): The `defaultServiceAccounts[]` *baseline set* (which SAs, with which RBAC) is described only by example (developer/operator/agent-default SA). Exact templates are `[PROPOSED — not in source]`; must align with RBAC-as-floor (ADR 0018) and the deferred `platform_roles` catalog.
- **OQ-A21-5** (low): Tenant-scoped quotas are deferred (ADR 0037 / backlog §1.2); the XRD must be shaped so quotas compose on additively later without a breaking version bump.
- **OQ-A21-6** (low): Offboarding "archive audit references" interacts with the deferred F1 retention policy; A21 archives but cannot define expiry — risk of ambiguity until F1 lands.

## 11. References

- architecture-overview.md §6.9 (multi-tenancy, claims, tenant onboarding flow, lines 720–772); §6.11 (identity federation, lines 837–851+); §6.12 (CRD inventory — `TenantOnboarding` row, line 972); §14.1 A21 (line 1687); §14.2 B4 (line 1698).
- ADR 0037 (Tenant onboarding via Headlamp + `TenantOnboarding` XRD — primary). ADR 0028 (identity federation chain — cluster-OIDC-side mapper only). ADR 0029 (Keycloak claim schema). ADR 0016 (multi-tenancy via namespaces). ADR 0013 (capability/identity decoupling). ADR 0039 / A22 (Headlamp editor). ADR 0044 (XRD/Composition contract + versioning). ADR 0030 (versioning). ADR 0034 (audit emission). ADR 0018 (RBAC-as-floor for default SAs).
- Related pieces: A9 (Headlamp framework), A22 (editor framework), B4 (Crossplane Compositions), B13 (CapabilitySet — decoupled), B1 (SSO/Keycloak config), F1 (audit retention).
- _meta/interface-contract.md §1.6 (`TenantOnboarding` XRD fields), §2 (`platform.tenant.*` / `platform.audit.*`), §4 (connection-secret — N/A here).
