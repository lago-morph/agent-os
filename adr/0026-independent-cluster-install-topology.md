# ADR 0026: Independent-cluster-install topology, no v1.0 federation

## Status

Accepted

## Context

The platform is built on top of an existing Kubernetes baseline. The supported
cluster targets are EKS, AKS, and kind for development. That list describes
*where* the platform can run; it does not describe a topology that knits
clusters together.

Larger Kubernetes-based platforms often grow toward multi-cluster federation:
shared control planes, cross-cluster service discovery, federated identity,
cross-cluster A2A traffic, or shared stateful resources. Each of those
introduces substantial complexity — federated identity (e.g. SPIFFE), a
cross-cluster networking layer, federated policy distribution, and operational
practices for split-brain and partial-failure scenarios.

For v1.0 we do not have a use case that requires any of that. The architecture
overview §3 baseline states explicitly that **each cluster is an independent
install of the platform**, and the backlog (§3.13, §3.5, §6 invariants, §7)
records federation as a future evolution path with explicit triggers, not a
v1.0 commitment.

## Decision

Each cluster is an independent install of the platform.

- The platform installs the same way on EKS, AKS, and kind. EKS/AKS/kind is the
  set of supported environments, not a federated topology.
- There is **no cross-cluster A2A traffic and no shared resources across
  clusters in v1.0**. Each cluster has its own control plane components, its
  own data stores, its own event mesh, and its own tenants.
- Identity is **Kubernetes ServiceAccounts**, scoped to a single cluster. We do
  not run SPIFFE or any federated identity layer in v1.0.
- Triggers to revisit (from backlog §3.13 and §3.5):
  - A concrete use case that requires cross-cluster awareness or shared
    resources reopens the federation question (§3.13).
  - If federation is adopted, SPIFFE identity becomes the trigger to replace
    ServiceAccount-only identity (§3.5).

## Consequences

- Operational model is simple: one cluster, one install, one tenancy boundary
  at the cluster edge.
- Disaster recovery and regional failover, if needed, are handled by running
  another independent install — not by a federated control plane.
- Tenants that need isolation get it via namespace tenancy within a cluster
  (ADR 0016); cross-cluster isolation is not a feature we offer.
- Components are designed assuming a single cluster scope: Postgres is a
  per-cluster managed instance (ADR 0014), the agent runtime and SDK selection
  are per-cluster (ADRs 0001, 0019), event sources are configured per
  environment (ADR 0023), and the egress proxy runs inside each cluster
  (ADR 0003).
- We accept that adding federation later is non-trivial. The backlog captures
  the trigger and the dependent decisions (notably identity) so the cost is
  paid only when justified.

## Alternatives considered

- **Multi-cluster federation from day one.** Rejected. The added complexity in
  identity, networking, policy distribution, and operations is not justified by
  any v1.0 use case. Carrying that machinery would slow every other workstream
  for a capability nobody has asked for.
- **Single-cluster-only as a hard constraint.** Rejected as too strong. We
  state the v1.0 topology and the triggers to revisit, rather than foreclosing
  federation forever.

## Related

- ADR 0003 — Envoy egress proxy is CNI-agnostic, which is what lets the same
  install work on EKS, AKS, and kind.
- ADR 0016 — Namespace multi-tenancy lives within a single cluster.
- ADR 0023 — Event sources differ per environment (AwsSqsSource on AWS, Azure
  equivalents, webhook receivers in kind).
- ADR 0014 — Postgres is a per-cluster managed instance.
- ADRs 0001, 0019 — Agent operator and SDK selection scope is per-cluster.
- Backlog §3.5 (SPIFFE identity trigger), §3.13 (multi-cluster federation
  trigger), §6 invariants, §7 ADR candidates.
