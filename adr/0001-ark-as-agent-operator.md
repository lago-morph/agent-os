# ADR 0001: ARK as the agent operator

## Status
Accepted

## Context
The platform needs a Kubernetes-native operator that turns declarative agent definitions into running, governed Platform Agents. The runtime must reconcile CRDs into agent pods inside Sandboxes, integrate with LiteLLM, Letta, and the Knative eventing fabric, and present a CRD surface broad enough to model agents, teams, tools, memory, evaluations, and queries without bespoke glue per concern.

Two viable OSS operators were evaluated: kagent and ARK. Both target the same general problem space, but they differ in CRD coverage and surrounding tooling. Because the platform is built around declarative governance (CapabilitySet layering, OPA admission, Crossplane Compositions, GitOps via ArgoCD), the operator's CRD surface is load-bearing — every primitive that exists as a CRD becomes governable, auditable, and composable; every primitive that doesn't has to be reinvented.

The choice also constrains downstream components. HolmesGPT (ADR 0012) runs as a first-class Platform Agent on this operator, the v1.0 SDK choice (ADR 0019) operates inside its `Agent` pods, and the capability model (ADR 0013) layers values onto its `Agent` CRD spec.

## Decision
Adopt **ARK** as the agent operator. ARK provides the `Model`, `Agent`, `Team`, `Tool`, `Memory`, `Evaluation`, and `Query` CRDs that the platform reconciles together with `agent-sandbox`, LiteLLM, and the Platform SDK. kagent is rejected.

## Consequences
Positive:
- Broader CRD set out of the box (`Model`, `Agent`, `Team`, `Tool`, `Memory`, `Evaluation`, `Query`) — every agent-runtime concern has a declarative surface that OPA, RBAC, audit, and GitOps can act on uniformly.
- Ships with a CLI, SDKs, and a dashboard, reducing custom UX work.
- Better fit for the platform's declarative-governance model: CapabilitySets layer cleanly onto the `Agent` CRD, and `Memory` / `Tool` CRDs map to existing platform primitives (Letta, MCP) without parallel registries.
- Native primitives for `Team` and `Evaluation` align with multi-agent compositions (B18) and Langfuse/promptfoo evaluation flows without a separate framework.

Negative:
- Commits the platform to ARK's CRD shapes and release cadence; future divergence between ARK upstream and platform needs has to be managed (forks, contributions, or replacement).
- ARK's broader surface means more component-level documentation, runbook, and Headlamp-plugin work under Workstream A (component A5).
- Loses any kagent-specific features or community momentum; switching later is a non-trivial migration because Agent CRDs, capability layering, and HolmesGPT integration are all bound to ARK's schemas.

## Alternatives considered
- **kagent.** Rejected: narrower CRD set and weaker surrounding tooling (CLI, SDKs, dashboard), making declarative governance and operator UX more expensive to build on top.
- **Bespoke operator (kopf-based, in-tree).** Rejected implicitly: duplicates work that ARK already does, and the platform already commits kopf to LiteLLM reconciliation (ADR 0006); using it for the agent runtime as well would expand custom-controller scope without offsetting benefit.

## Related
- Architecture overview: §5 (Software added to baseline — ARK row), §6.2 Agent runtime architecture, §14.1 Workstream A (component A5).
- Architecture backlog: §2.1 (kagent vs ARK rationale), §7 ADR list.
- Other ADRs:
  - ADR 0012 — HolmesGPT runs as a first-class Platform Agent on this operator.
  - ADR 0013 — Capability CRD model and CapabilitySet layering on the `Agent` CRD.
  - ADR 0019 — Langchain Deep Agents as the v1.0 SDK, operating inside ARK `Agent` pods.
  - ADR 0005 — Letta as the memory backend, consumed via ARK's `Memory` CRD.
