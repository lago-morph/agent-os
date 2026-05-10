# ADR 0016: Multi-tenancy via Kubernetes namespaces with RBAC + OPA + network enforcement

## Status
Accepted

## Context
The platform hosts Platform Agents owned by different teams, projects, or business units on a shared cluster install. Tenancy needs a clear, enforceable boundary that applies uniformly to Agents, the CRDs they reference (CapabilitySet, MCPServer, A2APeer, RAGStore, VirtualKey, Memory), the Sandboxes they execute in, and the audit and observability data they emit. Each cluster is its own independent install of the platform (ADR 0026); tenants live within a cluster, not across clusters.

The architecture also commits to dynamic agent registration: a Platform Agent that exposes A2A or MCP advertises itself to LiteLLM at startup, gated by OPA. Tenancy must scope this registration — an agent can register only in namespaces it has admission rights for, and OPA decides the resulting visibility. Similarly, Memory stores declare one of three visibility modes per ADR 0025 (private, namespace-shared, RBAC/OPA-controlled), and those modes only make sense relative to a defined namespace boundary.

Defense-in-depth matters here: a single misconfigured policy or compromised gateway must not collapse tenant isolation.

## Decision
A tenant maps to one or more Kubernetes namespaces; the namespace is the tenancy boundary. Isolation is enforced at three independent layers:

1. **CRD admission** — Gatekeeper (ADR 0002) admits Agents, Sandboxes, and related CRs only if they comply with the namespace's tenancy rules.
2. **Runtime authorization** — every A2A / MCP call traverses LiteLLM, which consults OPA on each invocation: "may caller in tenant A invoke callee in tenant B?" Dynamic agent registration is gated the same way and produces one of three visibility modes — invisible outside namespace (default), read-only-visible to other tenants, or read-write-visible to specific tenants.
3. **Network enforcement** — Kubernetes NetworkPolicy plus the Envoy egress proxy (ADR 0003) block cross-namespace and external traffic that policy did not authorize, even if a higher layer is misconfigured.

RBAC remains the floor and OPA the restrictor (ADR 0018): cross-tenant access requires both an RBAC grant and an OPA allow. Memory visibility modes (ADR 0025) and namespaced GrafanaDashboard XRs follow the same model. Cross-tenant resource sharing (e.g., a CapabilitySet referenced from another namespace) is allowed only when explicitly published with an OPA-checked policy; the default is namespace-private.

Tenant onboarding flow, tenant-scoped resource quotas (max agents, sandboxes, virtual keys, cost budget, rate limits), and cross-tenant collaboration patterns are deferred to design (architecture backlog §1.2).

## Consequences
Positive:
- One isolation primitive (namespace) reused across all tenant-scoped concerns: Agents, CRDs, Sandboxes, Memory, dashboards, audit, virtual keys.
- Defense-in-depth: admission, gateway authorization, and network policy fail independently; no single layer is load-bearing for isolation.
- Dynamic agent registration and the three memory visibility modes share one mental model with cross-tenant agent visibility.
- Native Kubernetes tooling (kubectl, RBAC, NetworkPolicy, Gatekeeper) covers tenancy without bespoke machinery.

Negative:
- A namespace is a relatively coarse unit; tenants needing finer-grained sub-isolation must split into multiple namespaces, increasing CRD count.
- Three enforcement layers means three places to keep aligned; drift produces either over-permissive holes or false denials that are hard to diagnose.
- ARK's multi-tenancy and per-tenant RBAC coverage may be thin in technical-preview; some gaps will be filled with namespace-based isolation plus OPA admission until upstream catches up.

## Alternatives considered
- **Separate Kubernetes cluster per tenant.** Rejected: operational overhead is large, and it conflicts with the independent-cluster install topology (ADR 0026), which treats each cluster as one platform install hosting many tenants — not one cluster per tenant.
- **Virtual cluster per tenant (e.g., vcluster).** Rejected: adds a control-plane abstraction whose operational and observability cost is not justified by v1.0 isolation requirements; namespace + RBAC + OPA + NetworkPolicy already meet them.
- **Label-based logical partitioning without namespaces.** Rejected: relies on policy correctness for everything; loses native admission, RBAC, NetworkPolicy, and quota scoping; weaker defense-in-depth.

## Related
- Architecture overview §6.6 (security and policy), §6.9 (multi-tenancy and namespacing), §6.10 and elsewhere on dynamic agent registration.
- Architecture backlog §1.2 (tenant onboarding, quotas, cross-tenant collaboration deferred to design), §6 (invariants), §7 entry 16 (this decision).
- ADR 0002 (OPA + Gatekeeper — admission and runtime policy engine).
- ADR 0003 (Envoy egress proxy — network-level enforcement).
- ADR 0018 (RBAC-as-floor / OPA-as-restrictor — the enforcement model).
- ADR 0025 (Memory access modes — private / namespace-shared / RBAC-OPA-controlled).
- ADR 0026 (Independent-cluster install — each cluster is its own platform install; tenants live within).
