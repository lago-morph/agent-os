# ADR 0006: Python kopf operator for LiteLLM reconciliation

- **Status**: Accepted
- **Date**: 2026-05-10

## Context

LiteLLM is the single gateway for every LLM, MCP, and A2A call (architecture-overview.md §6.1) and exposes an admin API rather than a Kubernetes-native interface. The platform needs a CRD-driven reconciler to keep LiteLLM in sync with the capability surface — `MCPServer`, `A2APeer`, `RAGStore`, `EgressTarget`, `Skill`, `CapabilitySet`, `VirtualKey`, and `BudgetPolicy` (§6.8, §6.6, CRD catalog in §6.12) — so that capabilities are declarative, GitOps-reconciled, and OPA-gated like every other platform surface. There is no native Crossplane provider for LiteLLM (§9), so the choice is between writing one in Go or writing a custom controller in Python with kopf (architecture-backlog.md § 2.5).

## Decision

The platform implements a **custom Python kopf operator** (component **B13**, architecture-overview.md §5, §6.8) that reconciles the LiteLLM-facing CRDs above into LiteLLM's admin API and into OPA data where applicable (§6.6). Kopf is used for application-API reconciliation against LiteLLM; **Crossplane v2 remains the controller for cloud-shaped resources** (component B4) — `AgentEnvironment`, `MemoryStore`, `SyntheticMCPServer`, and `GrafanaDashboard` XRs (§6.12, §11). The split is the platform-wide invariant: every controller is either Crossplane (cloud-shaped) or kopf (app-API), with no bespoke controller frameworks (architecture-backlog.md § 6).

The kopf operator is packaged as a **subchart of the LiteLLM Helm chart**, so a single ArgoCD-applied chart deploys both, the operator's lifecycle is bound to LiteLLM, and version pinning is automatic. Crossplane is deliberately not used for this packaging: Crossplane's value is composing cloud-shaped resources, not packaging a Kubernetes Deployment for an in-cluster reconciler.

## Alternatives considered

- **Go Crossplane provider for LiteLLM** — Rejected: the team has no Go experience; a kopf operator achieves the same reconciliation outcome and keeps Python as the platform's default language (architecture-backlog.md § 2.5). The trade-off is losing Crossplane's connection-secret handling, package model, and Composition integration for the LiteLLM-specific surface; these losses are bounded to the LiteLLM-facing CRDs and do not affect the Crossplane track.

## Consequences

- The kopf operator (B13) owns reconciliation for `MCPServer`, `A2APeer`, `RAGStore`, `EgressTarget`, `Skill`, `CapabilitySet`, `VirtualKey`, and `BudgetPolicy`, including the dependency edges between them and LiteLLM's runtime registry (architecture-overview.md §6.8, §6.6, §6.12).
- The controller-framework invariant is reinforced: **all controllers either use Crossplane (cloud-shaped) or kopf (application-API). No bespoke controller frameworks** (architecture-backlog.md § 6).
- Python remains the default language for custom platform code, alongside the explicit Headlamp/TypeScript exception (architecture-backlog.md § 6); the operator joins LiteLLM callbacks, Crossplane composition functions, glue services, and HolmesGPT toolsets in the Python "platform glue" workstream (architecture-overview.md §9).
- Crossplane's connection-secret handling, package model, and Composition integration are unavailable for LiteLLM-specific CRDs; the operator implements equivalent secret wiring (via ESO / Kubernetes Secrets) and CRD lifecycle directly, and does not participate in Crossplane Compositions.
- Crossplane v2 (B4) remains authoritative for cloud-shaped XRs — `AgentEnvironment`, `MemoryStore`, `SyntheticMCPServer`, and `GrafanaDashboard` (architecture-overview.md §6.12, §11; ADR 0021); the kopf operator does not encroach on that surface.
- The operator emits capability-registry reconcile events to the audit pipeline (architecture-overview.md §6.7) and exposes an HTTP admin API versioned under `/v1/...` per the API-versioning policy (§6.13).
- The operator inherits the standard custom-component deliverables: HolmesGPT toolset, OPA contribution, audit emission, observability, Knative trigger flow, tests, and tutorial / how-to documentation (architecture-backlog.md § 6).
- The kopf reconciler ships and upgrades together with LiteLLM as a subchart of the LiteLLM Helm chart, giving a single ArgoCD-applied release, automatic version pinning between gateway and operator, and a shared lifecycle.

## References

- [architecture-overview.md](../architecture-overview.md) [§6.1](../architecture-overview.md#61-gateway-architecture), [§6.6](../architecture-overview.md#66-security-and-policy-architecture), [§6.8](../architecture-overview.md#68-capability-registries-and-approved-primitives), [§6.12](../architecture-overview.md#612-crd-inventory), [§6.13](../architecture-overview.md#613-versioning-policy), [§9](../architecture-overview.md#9-oss-limitations-and-required-custom-development), [§5](../architecture-overview.md#5-software-added-to-baseline) (component B13), [§11](../architecture-overview.md#11-grafana-dashboards)
- [architecture-backlog.md](../architecture-backlog.md) [§ 2.5](../architecture-backlog.md#25-litellm-provider-go-crossplane-provider-vs-python-kopf-operator), [§ 6](../architecture-backlog.md#6-architecture-level-invariants-worth-documenting-as-adrs)
- [ADR 0001](./0001-ark-as-agent-operator.md) (ARK as the agent operator) — sibling reconciler in the controller landscape
- [ADR 0021](./0021-grafanadashboard-xrs.md) (`GrafanaDashboard` XRs use Crossplane Compositions) — Crossplane side of the kopf/Crossplane split
