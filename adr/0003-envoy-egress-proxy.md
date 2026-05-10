# ADR 0003: Envoy egress proxy for CNI-agnostic egress control

## Status
Accepted

## Context
The platform installs onto EKS, AKS, and kind, and each cluster is an independent install (see ADR 0026). We cannot assume a particular CNI is present, and managed Kubernetes offerings vary in what L7 network policy they expose. At the same time, every Platform Agent must be constrained to a known set of external destinations, with audit and policy enforcement on each outbound call.

The architecture already establishes a single LLM/MCP/A2A choke point at the LiteLLM gateway. But Platform Agents and supporting components also make non-LiteLLM HTTP calls (vendor APIs, webhooks, internal service endpoints declared as egress targets). Those calls need the same governance properties: an FQDN allowlist tied to the agent's CapabilitySet, an OPA hook for runtime decisions, and emission to the audit index.

Cilium L7 NetworkPolicy was the natural fit if the platform standardized on Cilium, but Cilium is not uniformly available across EKS, AKS, and kind. Tying egress control to a specific CNI would break the independent-cluster install model.

We therefore need an egress mechanism that is CNI-agnostic, runs the same way in every cluster, and exposes the hooks the rest of the platform expects.

## Decision
Use an Envoy-based egress proxy as the single egress point for all non-LiteLLM HTTP traffic from Platform Agents and platform components. Envoy enforces the FQDN allowlist derived from the agent's CapabilitySet, calls the OPA hook for per-request decisions, terminates and re-originates mTLS where applicable, and emits structured egress events to the audit index.

This preserves the platform invariant: every Platform Agent invocation traverses LiteLLM, and Envoy handles non-LiteLLM HTTP. There are no other paths to external services.

The Envoy egress proxy is installed via Helm in every cluster, with per-agent-class configuration generated from CapabilitySet egress targets.

## Consequences
- Egress control runs identically on EKS, AKS, and kind; the platform is not coupled to a CNI.
- Two egress chokes (LiteLLM for LLM/MCP/A2A, Envoy for everything else) form a complete defense-in-depth boundary alongside the agent-sandbox kernel isolation.
- OPA policy authors get one consistent hook shape for egress decisions across clusters.
- Audit completeness is achievable: every outbound HTTP call lands in the audit index regardless of destination type.
- Operational cost: an extra in-cluster proxy to run, monitor, and upgrade. Envoy configuration must be regenerated when CapabilitySets change; this is owned by the LiteLLM kopf operator pattern's sibling reconciler for egress targets.
- Latency cost: one extra hop on non-LiteLLM HTTP. Acceptable for agent workloads.
- Bypass attempts (a workload trying to reach the internet directly) must still be blocked. We rely on default-deny pod egress at whatever network layer the cluster offers, with Envoy as the only allowed egress destination. Where the CNI cannot enforce this, the sandbox network namespace constrains it.

## Alternatives considered
- **Cilium L7 NetworkPolicy.** Rejected because Cilium is not available across EKS, AKS, and kind, which would break the independent-cluster install model in ADR 0026.
- **CNI-specific L7 policies per platform.** Rejected: three policy languages, three audit shapes, three OPA integration paths. Violates the single-mechanism principle the platform applies elsewhere.
- **No dedicated egress proxy, rely on LiteLLM plus per-workload HTTP client config.** Rejected: leaves non-LiteLLM HTTP without a uniform allowlist, OPA hook, or audit emission, and depends on every workload behaving correctly.
- **Service mesh (Istio/Linkerd) egress gateway.** Rejected for v1.0: the platform does not otherwise need a full mesh, and adopting one solely for egress is disproportionate. Envoy alone gives the same egress semantics without the mesh footprint.

## Related
- ADR 0002: OPA + Gatekeeper — provides the OPA hook Envoy calls for runtime egress decisions.
- ADR 0026: Independent-cluster topology — drives the CNI-agnostic requirement (EKS, AKS, kind).
- Architecture overview §3 (baseline), §4 (high-level), §6.6 (security), §5 (software list).
- Architecture backlog §2.3 (alternative considered), §6 (invariants), §7 (ADR candidates).
