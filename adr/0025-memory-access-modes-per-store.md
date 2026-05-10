# ADR 0025: Memory access modes per memory store

- **Status**: Accepted
- **Date**: 2026-05-10

## Context

Platform Agents need memory that ranges from strictly private per-conversation
state, through context shared across agents in the same tenant namespace, up to
selectively-shared knowledge that crosses namespace boundaries under policy.
A single platform-wide access setting cannot serve all three use cases — some
stores must be invisible to anything but their owning agent, while others must
participate in cross-tenant sharing under explicit policy. Treating access mode
as a property of the store (not of the platform, not of the agent) keeps the
authorization story local to where the data lives and aligns with the
namespace-as-tenant boundary established in ADR 0016.

## Decision

Each `MemoryStore` declares exactly one of three access modes in its CRD spec:
**private to the agent** (only the writing Platform Agent can read), **shared
by namespace** (any Platform Agent in the same namespace can read), or
**RBAC/OPA-controlled** (Kubernetes RBAC plus OPA decide visibility and
read/write per request). Agent CRDs reference memory stores by name; the
referenced store's declared mode determines what the platform allows. There is
no platform-wide override that changes a store's mode after declaration.

## Consequences

- Private and namespace-shared modes are enforced by the platform without
  per-request policy evaluation; the boundary is structural (writer identity,
  namespace membership) and cheap to check.
- The RBAC/OPA-controlled mode follows the platform-wide authorization model
  from ADR 0018: RBAC grants the floor, OPA may restrict per decision, and OPA
  never grants access RBAC did not already permit.
- Namespace isolation is the default posture (consistent with ADR 0016); a
  store opts in to cross-namespace sharing by selecting the RBAC/OPA mode, not
  by a separate flag.
- The Letta-backed memory service (ADR 0005) and the Crossplane-composed
  `MemoryStore` XR both honour the declared mode; backend choice does not
  change the access semantics surfaced to agents.
- Detailed memory namespace and sharing semantics — naming, lifecycle,
  cross-store joins, eviction — are deferred per architecture-backlog § 4
  ("Memory namespace and sharing model details"). This ADR fixes only the
  three-mode access contract.
- Gatekeeper admission rejects `MemoryStore` resources whose `accessMode` is
  missing or outside the three permitted values, keeping the invariant
  declarative and auditable.
- Auditing of memory reads and writes attributes events to the agent identity
  and the store's declared mode, enabling later detection of unintended access
  excess (the v1.0 threat-model focus, ADR 0027).

## References

- [architecture-overview.md § 6.3](../architecture-overview.md#63-memory-and-data-architecture)
- [architecture-backlog.md](../architecture-backlog.md) [§ 4](../architecture-backlog.md#4-topics-that-need-further-design-before-implementation), [6](../architecture-backlog.md#6-architecture-level-invariants-worth-documenting-as-adrs)
- [ADR 0005](./0005-letta-memory-backend.md) (Letta), [ADR 0016](./0016-multi-tenancy-via-namespaces.md) (multi-tenancy), [ADR 0018](./0018-rbac-floor-opa-restrictor.md) (RBAC-floor / OPA-restrictor)
