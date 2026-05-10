# ADR 0026: Independent-cluster-install topology with no v1.0 federation

- **Status**: Accepted
- **Date**: 2026-05-10

## Context

The platform runs on Kubernetes (EKS, AKS, or kind for development) and must commit to a topology that bounds identity, eventing, policy, and data design. Real installations may run multiple clusters — per-environment (dev / staging / prod), per-region, or per-team — and a multi-cluster federation model would touch nearly every architectural view (identity in §6.11, eventing in §6.7, multi-tenancy in §6.9, memory in §6.3). The architecture-overview baseline assumptions (§3) already state that the EKS / AKS / kind list describes where the platform can run, not a topology that knits clusters together. v1.0 has no concrete use case that requires cross-cluster awareness or shared resources, and committing to federation prematurely would force design decisions (cross-cluster identity root, cross-cluster event routing, cross-cluster RBAC) that cannot be validated without that use case.

## Decision

Each cluster is an independent install of the platform. v1.0 commits to no multi-cluster federation: no shared identity root across clusters, no cross-cluster event mesh, no shared CRDs or registries, and no cross-cluster A2A. Multi-environment deployments are modelled as N independent installs, with environment promotion handled by GitOps (ArgoCD reconciling each cluster from its own manifests) rather than by runtime federation. The architecture remains free to add federation later behind the explicit trigger documented in backlog 3.13.

## Consequences

- Cross-cluster identity is not a v1.0 concern. The platform stays on Kubernetes ServiceAccounts as the workload identity primitive (backlog 2.19); SPIFFE/SPIRE is deferred (backlog 3.5) because its primary value — a cross-cluster identity root — has no v1.0 consumer.
- Identity federation in §6.11 is scoped to a single cluster: cluster OIDC issuer → IRSA / Workload Identity for cloud access, and cluster OIDC issuer → Keycloak for platform JWTs. There is no inter-cluster trust path (see ADR 0028).
- Multi-environment installs (dev / staging / prod) are operated as independent platform instances. Environment-specific Knative event sources (ADR 0023) and per-cluster Crossplane providers (ADR 0033) align with this: each cluster owns its own eventing, secrets, and cloud-provider bindings.
- High availability and cross-region redundancy are explicitly out of v1.0 scope and tracked in future-enhancements §1; cross-cluster federation is listed there as the dependency that unlocks redundancy spanning clusters.
- Trigger to revisit (backlog 3.13): a concrete use case that requires cross-cluster awareness or shared resources — e.g., agents in cluster A invoking agents or memory in cluster B, a shared knowledge base across regions, or HA that genuinely spans clusters. At that point this ADR is superseded and SPIFFE / federated identity (ADR 0028) is reopened.
- Operational simplification for v1.0: a single set of CRDs, one Keycloak realm per platform install, one OPA bundle distribution, and one audit pipeline per cluster (Postgres + S3 system of record on AWS, Postgres-only on kind, with OpenSearch as advisory fanout — see ADR 0034). Tenancy boundaries remain namespace-scoped (ADR 0016) within a single cluster, not cross-cluster.
- Cross-cluster *promotion* (not federation) is handled by **Kargo (ADR 0040)**: Kargo controllers in each cluster, or a centralized Kargo control plane reaching into each ArgoCD, orchestrate environment-to-environment promotion across the independent installs. This does not introduce a federation requirement — each cluster remains independent — but provides a uniform promotion contract on top of the independent-cluster topology.

## References

- [architecture-overview.md](../architecture-overview.md) [§3](../architecture-overview.md#3-baseline-assumptions) (baseline assumptions, "Each cluster is an independent install of the platform"), [§6.9](../architecture-overview.md#69-multi-tenancy-and-namespacing) (multi-tenancy and namespacing), [§6.11](../architecture-overview.md#611-identity-federation) (identity federation)
- [architecture-backlog.md](../architecture-backlog.md) [§2.19](../architecture-backlog.md#219-identity-spiffe-vs-kubernetes-serviceaccounts) (Kubernetes ServiceAccounts vs SPIFFE), [§3.5](../architecture-backlog.md#35-spiffe-identity) (SPIFFE trigger), [§3.13](../architecture-backlog.md#313-multi-cluster-federation) (multi-cluster federation trigger), [§6](../architecture-backlog.md#6-architecture-level-invariants-worth-documenting-as-adrs) (invariant)
- [future-enhancements.md §1](../future-enhancements.md#1-redundancy-and-degraded-operations) (redundancy and degraded operations)
- [ADR 0016](./0016-multi-tenancy-via-namespaces.md) (multi-tenancy via namespaces)
- [ADR 0023](./0023-environment-specific-knative-sources.md) (environment-specific Knative sources)
- [ADR 0028](./0028-identity-federation.md) (identity federation)
- [ADR 0033](./0033-initial-implementation-targets-aws-github.md) (AWS + GitHub initial targets)
- [ADR 0034](./0034-audit-pipeline-durable-adapter.md) (audit pipeline — Postgres + S3 system of record, OpenSearch advisory fanout)
- [ADR 0040](./0040-kargo-promotion-fabric.md) (Kargo cross-cluster promotion)
