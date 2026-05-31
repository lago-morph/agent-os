# SPEC ADR-0021 — Dashboards as namespaced Crossplane-composed GrafanaDashboard XRs `[PROPOSED]`

> kind: ADR · workstream: — · tier: T1
> upstream: [ADR-0016, ADR-0018, ADR-0041, ADR-0040, ADR-0002] · downstream: [B4, D1, D2, D3, A-components, B10, A14] · adrs: [0021] · views: [6.5]
> canon-glossary: 6aadcc2a4f38 · canon-interface: 54f5ede58e5f

## 1. Purpose & Problem Statement

ADR 0021 is a settled decision: every Grafana dashboard the platform ships — per-component, integrated operator, developer-facing, and dashboards Platform Agents publish about themselves — is delivered as a Crossplane v2 Composition of a `GrafanaDashboard` XR rather than ad-hoc JSON imported into Grafana. This SPEC states what honoring that decision requires: the CRD shape that must exist, the governance hooks (RBAC + OPA + Gatekeeper) that must apply, the ownership boundary (B4 owns the XRD/Composition), and the acceptance evidence proving dashboards never escape the GitOps + tenancy + admission model.

The problem ADR 0021 closes is divergence: a dashboard delivered outside the CRD model would escape Gatekeeper admission, namespace tenancy (ADR 0016), and the RBAC-as-floor / OPA-as-restrictor model (ADR 0018). Honoring the ADR means dashboards are governed exactly like every other composed resource.

## 2. Scope

### 2.1 In scope
- The obligation that all dashboards are instances of the namespaced `GrafanaDashboard` XR.
- The required XR fields `dashboardJson`, `folder`, `visibility` and their governance semantics.
- The constraint that visibility is gated by Kubernetes RBAC + OPA, and that Gatekeeper admission applies.
- Ownership: the XRD + Composition are a B4 deliverable; per-substrate Composition (kind / AWS) behind a uniform XR schema.

### 2.2 Out of scope (and where it lives instead)
- Building the XRD/Composition and Grafana provider — owned by **B4** (this ADR imposes constraints on it, does not build it).
- Authoring concrete dashboards — Workstream **D** (D1/D2/D3) and each Workstream A component author instances.
- SLO / error-budget / ops-metrics dashboards — deferred to future enhancements (overview §11); ship later as additional `GrafanaDashboard` instances.
- Substrate-abstraction mechanics in general — **ADR 0044**.
- Cross-environment promotion of dashboards — **ADR 0040** (Kargo).

## 3. Context & Dependencies

Upstream decisions honored:
- **ADR 0016** (namespace tenancy): `GrafanaDashboard` XRs are namespaced; tenant A's dashboards are invisible to tenant B by default.
- **ADR 0018** (RBAC-floor / OPA-restrictor): `visibility` read/write is gated by RBAC floor plus OPA restriction; OPA never grants beyond RBAC.
- **ADR 0044** (substrate abstraction): the same XRD shape applies on kind and AWS with one Composition per substrate behind a uniform XR schema; `GrafanaDashboard` is the earliest concrete example.
- **ADR 0040** (Kargo): the uniform XR schema is what promotion consumes.
- **ADR 0002** (Gatekeeper): admission applies to `GrafanaDashboard` XRs.

Downstream consumers: B4 (implements), Workstream D, every Workstream A component, B10 (Coach) and A14 (HolmesGPT) which publish agent-authored dashboards.

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
- `GrafanaDashboard` (XR), **namespaced** (created directly in the tenant namespace, Crossplane v2). Fields from Canon: `dashboardJson`, `folder`, `visibility` (RBAC + OPA-controlled).
- Versioned per ADR 0030 (Kubernetes API versioning; breaking change → new `vN` group + conversion webhook; `vN-1` deprecated ≥1 minor release). Owner of versioning lifecycle: B4.

### 4.2 APIs / SDK surfaces
N/A — ADR imposes no new SDK/API surface; agent-published dashboards are declared via the Agent CRD spec or standalone XR (Canon names only).

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
N/A — ADR 0021 mandates no specific event; lifecycle of XRs falls under `platform.lifecycle.*` if emitted (deferred to component design, B12 registry).

### 4.4 Data schemas / connection-secret contracts
N/A — `GrafanaDashboard` composes Grafana folders/JSON/datasource bindings, not a substrate primitive writing the connection-secret shape; per ADR 0044 it is a higher-level XR.

## 5. OSS-vs-Custom Decision
N/A — ADR (no component build). The enforcing build (Crossplane v2 Composition + Grafana provider) is decided in the B4 SPEC; ADR 0021 only fixes that dashboards take the Crossplane-composition path, not the kopf path (ADR 0006).

## 6. Functional Requirements
- REQ-ADR-0021-01: Every platform dashboard MUST be delivered as an instance of the `GrafanaDashboard` XR; no dashboard is imported into Grafana outside this XR.
- REQ-ADR-0021-02: The `GrafanaDashboard` XR MUST be namespaced and MUST carry `dashboardJson`, `folder`, and `visibility` fields.
- REQ-ADR-0021-03: `visibility` (read/write on the resource) MUST be enforced by Kubernetes RBAC as floor and OPA as restrictor (ADR 0018), namespace-scoped by default (ADR 0016).
- REQ-ADR-0021-04: Gatekeeper admission MUST apply to `GrafanaDashboard` XR creation/update.
- REQ-ADR-0021-05: The XRD + Composition MUST be owned by B4 and MUST present one uniform XR schema with one Composition per substrate (kind / AWS).
- REQ-ADR-0021-06: Platform Agents (Coach, HolmesGPT, others) MUST be able to publish dashboards declaratively via the same XR without bypassing governance.
- REQ-ADR-0021-07: `GrafanaDashboard` MUST follow ADR 0030 versioning (conversion webhook + ≥1-minor-release deprecation on breaking change).

## 7. Non-Functional Requirements
- Security / multi-tenancy: namespace isolation default; no cross-tenant dashboard read without explicit OPA-gated share.
- Observability: dashboards are themselves the observability surface; XR reconcile state visible in Headlamp.
- Versioning: per ADR 0030 (§4.1).
- Scale: GitOps-managed on the standard ArgoCD reconciliation path; promotion via Kargo (ADR 0040) using the substrate-agnostic XR schema.

## 8. Cross-Cutting Deliverable Checklist
N/A — ADR. (Enforcing deliverables — Helm/manifests, Gatekeeper policy, Headlamp visibility, 3-layer tests — are carried by B4 and the authoring components D/A; see PLAN §1.)

## 9. Acceptance Criteria
- AC-ADR-0021-01: Honored when a Chainsaw test proves a dashboard delivered as raw Grafana JSON (not via the XR) is rejected/absent from the GitOps path, and an XR-delivered dashboard appears. (REQ-01, REQ-02)
- AC-ADR-0021-02: Honored when applying a `GrafanaDashboard` missing `dashboardJson`/`folder`/`visibility` is rejected by Gatekeeper admission. (REQ-02, REQ-04)
- AC-ADR-0021-03: Honored when a principal with RBAC floor but OPA-denied is refused read on a `GrafanaDashboard`, and a same-namespace permitted principal succeeds; cross-namespace read fails without an explicit share. (REQ-03)
- AC-ADR-0021-04: Honored when the same XR manifest reconciles on both kind and AWS Compositions producing an equivalent dashboard. (REQ-05)
- AC-ADR-0021-05: Honored when an Agent (e.g. HolmesGPT) publishes a dashboard via the XR and it is governed identically to an operator-authored one. (REQ-06)
- AC-ADR-0021-06: Honored when a breaking schema change is shown to require a new `vN` group + conversion webhook with `vN-1` retained ≥1 minor release. (REQ-07)

## 10. Risks & Open Questions
- Risk (med): Grafana provider / reconciliation path choice is a B4 decision; if no mature Crossplane Grafana provider exists, B4 may need an equivalent path. Reconciliation note: ADR permits "the Grafana provider or equivalent reconciliation path" — does not constrain.
- Risk (low): agent-published dashboards could carry unvetted `dashboardJson`; mitigated by Gatekeeper + OPA on the XR.
- Open question: whether `visibility` is an enum or a structured policy reference is `[PROPOSED — not in source]`; resolve in B4 SPEC.

## 11. References
- ADR 0021 (this decision). Enforcing components: B4 (XRD + Composition), Workstream D (D1/D2/D3), Workstream A components (per-component dashboards), B10/A14 (agent-published).
- architecture-overview.md §11 (Grafana dashboards), §14.4 (Workstream D).
- ADR 0016, ADR 0018, ADR 0002 (Gatekeeper), ADR 0006 (kopf vs Crossplane split), ADR 0040, ADR 0041, ADR 0030.
- Canon: `_meta/interface-contract.md` §1.6 (`GrafanaDashboard` XR), glossary (`GrafanaDashboard`).
