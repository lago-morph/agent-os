# ADR 0001: ARK as the agent operator

- **Status**: Accepted
- **Date**: 2026-05-10

## Context

The platform needs a Kubernetes-native operator to manage Platform Agent lifecycle declaratively, so that agents can be defined as CRDs in Git, reconciled by ArgoCD, and run inside `agent-sandbox`-managed pods (architecture-overview.md §6.2). The operator must supply the `Agent`, `AgentRun`, `Team`, `Tool`, `Memory`, `Evaluation`, and `Query` CRDs that anchor the declarative model (§6.12) and integrate with LiteLLM, Letta, OPA admission, OTel emission, and Knative event flows (§6.5, §6.7). Two OSS options were evaluated: ARK and kagent (architecture-backlog.md § 2.1).

## Decision

The platform adopts **ARK** as the agent operator (component **A5**, architecture-overview.md §5, §14.1). Platform Agents are declared as ARK `Agent` CRDs and reconciled by ARK into pods running inside Sandboxes; ARK also owns the `AgentRun`, `Team`, `Tool`, `Memory`, `Evaluation`, and `Query` CRDs listed in §6.12. ARK is installed and operated under the standard Workstream A deliverables (§14.1), with version pinning per the OSS-limitations mitigation in §9.

## Alternatives considered

- **kagent** — Rejected: narrower CRD set and weaker CLI / SDK / dashboard surface, less aligned with the declarative-governance posture (architecture-backlog.md § 2.1).

## Consequences

- ARK CRDs (`Agent`, `AgentRun`, `Team`, `Tool`, `Memory`, `Evaluation`, `Query`) are the canonical declarative surface for Platform Agents and follow the CRD versioning policy in §6.13, with ARK install (A5) owning the reconcilers.
- ARK is technical-preview; mitigations are pinning a tested version, planning upstream contributions, and layering namespace isolation + OPA admission + a Headlamp plugin for cross-tenant visibility (architecture-overview.md §9).
- Multi-tenancy relies on namespace-based isolation rather than ARK-native per-tenant RBAC (§6.9, §9); per-tenant RBAC details deferred to design (architecture-backlog.md § 1.2).
- The agent base container image bundles the ARK ADK alongside other SDKs (§5), and the Platform SDK ships a version-pinned compatibility matrix against ARK (§6.13).
- Letta is consumed exclusively through ARK's `Memory` CRD, keeping memory bindings declarative (§6.3, §9).
- Knative event adapters and Argo Workflows steps create ARK `AgentRun` resources to drive triggered and long-running executions (§6.7, §7.2).
- HolmesGPT receives an ARK toolset for platform self-management (§6.10).

## References

- [architecture-overview.md](../architecture-overview.md) [§5](../architecture-overview.md#5-software-added-to-baseline), [§6.2](../architecture-overview.md#62-agent-runtime-architecture), [§6.12](../architecture-overview.md#612-crd-inventory), [§6.13](../architecture-overview.md#613-versioning-policy), [§9](../architecture-overview.md#9-oss-limitations-and-required-custom-development), [§14.1](../architecture-overview.md#141-workstream-a--platform-installation-and-operations)
- [architecture-backlog.md § 2.1](../architecture-backlog.md#21-agent-operator-kagent-vs-ark)
