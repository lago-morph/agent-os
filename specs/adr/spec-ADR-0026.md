# SPEC ADR-0026 — Independent-cluster-install topology with no v1.0 federation [PROPOSED]

> kind: ADR · workstream: — · tier: T2
> upstream: [] · downstream: [A23, A18, A7, B4, A4] · adrs: [0026] · views: [6.9, 6.11]
> canon-glossary: cf2d1a754a58 · canon-interface: 45ee7b798c47

## 1. Purpose & Problem Statement

ADR 0026 is a settled decision: each cluster is an independent install of the platform, and v1.0 commits to no multi-cluster federation — no shared identity root, no cross-cluster event mesh, no shared CRDs or registries, no cross-cluster A2A. Multi-environment deployments are modelled as N independent installs, with environment promotion handled by GitOps (ArgoCD per cluster) rather than runtime federation. This SPEC states what honoring that topology obliges: identity, eventing, policy, and data designs stay single-cluster-scoped; each cluster owns its own Keycloak realm, OPA bundle, audit pipeline, and eventing; cross-cluster *promotion* (not federation) is the only inter-cluster contract, carried by Kargo. It does not re-argue the choice to defer federation.

The problem the decision solves: a federation model would touch nearly every architectural view (identity §6.11, eventing §6.7, multi-tenancy §6.9, memory §6.3), and v1.0 has no concrete use case requiring cross-cluster awareness. Committing to federation prematurely would force unvalidatable design decisions (cross-cluster identity root, event routing, RBAC). Bounding the topology to independent installs keeps v1.0 design tractable.

## 2. Scope

### 2.1 In scope
- The obligation that identity, eventing, policy, and data designs are single-cluster-scoped (no cross-cluster trust path, mesh, or shared registry).
- The obligation that multi-environment installs (dev/staging/prod) are N independent platform instances reconciled per-cluster by ArgoCD.
- The obligation that each cluster owns one Keycloak realm, one OPA bundle distribution, one audit pipeline, and its own eventing/secrets/cloud bindings.
- The obligation that workload identity stays on Kubernetes ServiceAccounts (no SPIFFE/SPIRE in v1.0).
- The obligation that cross-cluster *promotion* is via Kargo (ADR 0040) and does NOT introduce a federation requirement.

### 2.2 Out of scope (and where it lives instead)
- Kargo promotion fabric — component **A23** SPEC (ADR 0040).
- Per-cluster identity federation chain — **ADR 0028** (single-cluster scope).
- Environment-specific Knative sources — **ADR 0023**.
- Audit pipeline per cluster — **ADR 0034** / component **A18**.
- HA / cross-region redundancy — out of v1.0 scope (future-enhancements §1).
- Multi-cluster federation, SPIFFE, shared knowledge base across regions — deferred behind explicit triggers (backlog 3.13, 3.5).

## 3. Context & Dependencies

Upstream consumed: none — this is a topology-bounding decision, not a consumer of another piece.
Downstream consumers: **A23** (Kargo) implements cross-cluster promotion on top of independent installs; **A18** (audit) is one pipeline per cluster; **A7** (OPA) is one bundle distribution per cluster; **B4** (Crossplane) providers are per-cluster; **A4** (eventing) is per-cluster.

ADR decisions honored:
- **ADR 0026** (this) — independent installs; no v1.0 federation; promotion-not-federation via Kargo.
- **ADR 0016** — tenancy stays namespace-scoped within a single cluster, never cross-cluster.
- **ADR 0028** — identity federation is single-cluster: cluster OIDC issuer → IRSA/Workload Identity for cloud, cluster OIDC issuer → Keycloak for platform JWTs; no inter-cluster trust path.
- **ADR 0023** — each cluster owns its own environment-specific event sources.
- **ADR 0033** — per-cluster Crossplane providers / cloud bindings align with independent installs.
- **ADR 0034** — one audit pipeline per cluster (Postgres+S3 on AWS, Postgres-only on kind, OpenSearch advisory fanout).
- **ADR 0040** — Kargo provides the cross-cluster promotion contract without federation.

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs
N/A — ADR 0026 introduces no CRD/XRD. It constrains scope: all v1.0 platform CRDs are namespaced (glossary invariant) and per-cluster; there are no shared or cluster-federated CRDs/registries. The `TenantOnboarding` XRD's `clusterOIDCClaimMapping` (interface-contract §1.6) is per-cluster, consistent with this topology.

### 4.2 APIs / SDK surfaces
- Cross-cluster A2A is explicitly absent in v1.0; A2A peers are intra-cluster only. The Kargo promotion contract (A23) is the sole inter-cluster surface and is a GitOps/promotion API, not a runtime federation API.

### 4.3 CloudEvents emitted / consumed
- The CloudEvent broker/mesh is per-cluster; there is no cross-cluster event mesh. The taxonomy (ADR 0031) is uniform per install but events do not route between clusters. `platform.tenant.*` cross-tenant events stay within a single cluster's broker.

### 4.4 Data schemas / connection-secret contracts
- One audit pipeline per cluster (ADR 0034). Connection secrets (ADR 0044 uniform shape) are per-cluster; no shared data substrate spans clusters. Memory, knowledge base, and registries are not shared across clusters in v1.0.

## 5. OSS-vs-Custom Decision
N/A — ADR. (Enforcement note: the topology is realized by per-cluster installs of existing components — **ArgoCD** reconciling each cluster, **Keycloak** one realm per install, **Kargo** (A23) for promotion. No federation control plane is built. SPIFFE/SPIRE is explicitly NOT adopted in v1.0.)

## 6. Functional Requirements
- REQ-ADR-0026-01: Each cluster MUST be an independent install: no shared identity root, no cross-cluster event mesh, no shared CRDs/registries, no cross-cluster A2A.
- REQ-ADR-0026-02: Multi-environment deployments (dev/staging/prod) MUST be modelled as N independent installs, each reconciled from its own manifests by ArgoCD.
- REQ-ADR-0026-03: Workload identity MUST remain Kubernetes ServiceAccounts; SPIFFE/SPIRE MUST NOT be introduced in v1.0.
- REQ-ADR-0026-04: Each cluster MUST own exactly one Keycloak realm, one OPA bundle distribution, one audit pipeline, and its own eventing/secrets/cloud-provider bindings.
- REQ-ADR-0026-05: Identity federation MUST be single-cluster-scoped per ADR 0028 — no inter-cluster trust path.
- REQ-ADR-0026-06: Cross-cluster promotion MUST be handled by Kargo (ADR 0040) on top of the independent-cluster topology and MUST NOT introduce a federation requirement (each cluster remains independent).
- REQ-ADR-0026-07: HA and cross-region redundancy MUST be out of v1.0 scope; adding federation later MUST be gated behind the explicit trigger (backlog 3.13).

## 7. Non-Functional Requirements
- Operational simplification: one realm, one OPA bundle, one audit pipeline, one CRD set per cluster — the simplification the topology exists to deliver.
- Multi-tenancy (§6.9): tenancy boundaries remain namespace-scoped within a single cluster (ADR 0016), never cross-cluster.
- Reversibility: federation can be added later behind the backlog 3.13 trigger; at that point this ADR is superseded and SPIFFE/federated identity (ADR 0028) reopens.
- Versioning (ADR 0030): independent installs mean each cluster's component versions evolve independently; no cross-cluster lockstep.

## 8. Cross-Cutting Deliverable Checklist
N/A — ADR (verification map lives in the PLAN). The §14.1 set applies to per-cluster enforcing components (A23, A18, A7, B4, A4), not to this topology record.

## 9. Acceptance Criteria
- AC-ADR-0026-01: Honored when no v1.0 artifact establishes a shared identity root, cross-cluster mesh, shared registry, or cross-cluster A2A path. (REQ-01/05)
- AC-ADR-0026-02: Honored when a dev/staging/prod layout is realized as three independent installs, each ArgoCD-reconciled from its own manifests. (REQ-02)
- AC-ADR-0026-03: Honored when workload identity resolves to Kubernetes ServiceAccounts with no SPIFFE/SPIRE component present. (REQ-03)
- AC-ADR-0026-04: Honored when each cluster has exactly one Keycloak realm, one OPA bundle, one audit pipeline, and its own eventing/secrets/cloud bindings. (REQ-04)
- AC-ADR-0026-05: Honored when cross-cluster promotion runs via Kargo with each cluster still independent (no shared runtime state introduced). (REQ-06)
- AC-ADR-0026-06: Honored when HA/cross-region redundancy is absent from v1.0 scope and any federation work is gated behind the backlog 3.13 trigger. (REQ-07)

## 10. Risks & Open Questions
- OQ-1 (low): The federation revisit trigger (backlog 3.13) is a future use case (cross-cluster agent/memory invocation, shared KB across regions, cross-cluster HA); until then no design exists. `[PROPOSED]`
- R-1 (med): Independent installs mean no cross-cluster redundancy — a single cluster loss is unmitigated in v1.0; accepted and tracked in future-enhancements §1.
- R-2 (low): Kargo's centralized control plane reaching into each ArgoCD could be mistaken for federation; mitigated by the explicit "promotion is not federation; each cluster remains independent" boundary.

## 11. References
- ADR 0026 (`adr/0026-independent-cluster-install-no-federation.md`) — the decision enforced here.
- architecture-overview.md §3 (baseline assumptions), §6.9 (multi-tenancy), §6.11 (identity federation).
- architecture-backlog.md §2.19 (SA vs SPIFFE), §3.5 (SPIFFE trigger), §3.13 (federation trigger), §6.
- future-enhancements.md §1 (redundancy and degraded operations).
- Enforcing/related components: A23 (Kargo), A18 (audit), A7 (OPA), B4 (Crossplane), A4 (eventing).
- ADR 0016 (multi-tenancy), 0023 (env-specific sources), 0028 (identity federation), 0033 (AWS+GitHub), 0034 (audit), 0040 (Kargo).
