# ADR 0006: Python kopf operator for LiteLLM reconciliation

## Status
Accepted

## Context
LiteLLM is the gateway chokepoint (architecture overview §6.1) and exposes its registry — MCP servers, A2A peers, virtual keys, capability bundles, RAG store wirings, budget policies — through an admin HTTP API rather than as Kubernetes objects. The platform needs that registry to be GitOps-driven and Headlamp-editable like every other platform surface, which means a controller has to translate Kubernetes CRs (`MCPServer`, `A2APeer`, `VirtualKey`, `CapabilitySet`, `RAGStore`, `BudgetPolicy`) into LiteLLM admin API calls and keep them reconciled.

There are two realistic shapes for that controller:

1. A Go-based Crossplane v2 provider, consistent with how the platform handles cloud-shaped resources (`AgentEnvironment`, `MemoryStore`, `SyntheticMCPServer`, `GrafanaDashboard` — see ADR 0021).
2. A Python kopf operator, sitting alongside Crossplane rather than inside it.

The team has no Go experience. Python is the platform's default custom-code language (every other custom controller surface — LiteLLM callbacks, HolmesGPT toolsets, Crossplane composition functions where used — is Python). Crossplane's value-add for this particular reconciliation is modest: there is no cloud account to authenticate against, no managed-resource lifecycle that benefits from Crossplane's connection-secret handling, and no composition story we'd lose because LiteLLM CRs aren't claims for cloud infrastructure — they're already the user-facing shape.

The backlog invariant (§6) requires that all controllers either use Crossplane (cloud-shaped resources) or kopf (application-API reconciliation) — no bespoke controller frameworks. This decision draws that line.

## Decision
Build a custom Python kopf operator that reconciles `MCPServer`, `A2APeer`, `VirtualKey`, `CapabilitySet`, `RAGStore`, and `BudgetPolicy` CRDs to the LiteLLM admin API. Crossplane v2 stays in place for cloud-shaped resources and dashboard XRs (ADR 0021); the two control loops coexist.

The operator runs as a standard Deployment, watches the listed CRD kinds, drives LiteLLM's admin API on create/update/delete, surfaces status back to the CRs, and consults OPA for runtime decisions where the policy invariants require it (virtual key issuance, dynamic registration, budget enforcement — see ADR 0002).

## Consequences
Positive:
- Keeps Python as the single custom-code language across the platform; no Go skill ramp required.
- kopf's programming model (decorated handlers, idempotent reconcile, finalizers, status patches) is simple enough to onboard onto in days.
- Same operator can host the runtime OPA bridges for the CRs it owns, keeping the policy seam clean.
- Functionally equivalent reconciliation outcome to a Crossplane provider for this scope.

Negative:
- We do not get Crossplane's connection-secret handling, package model, or composition integration for these CRs. Secret material (e.g., issued virtual key strings) needs an explicit ESO-compatible delivery path designed alongside the operator (backlog §1.3).
- Two controller frameworks live side-by-side. Engineers must know which framework owns which kind. The invariant in backlog §6 is the rule that keeps this from sprawling — anything not Crossplane is kopf, full stop.
- kopf's ecosystem is smaller than controller-runtime / Crossplane; some patterns (leader election, sharding) we'd inherit for free in Go we may have to reach for explicitly.

## Alternatives considered
- Go Crossplane v2 provider. Rejected: team has no Go experience, and the LiteLLM-specific reconciliation does not benefit enough from Crossplane's managed-resource model to justify the language split.
- Bespoke controller using controller-runtime in Go or a hand-rolled Python reconciler. Rejected by the §6 invariant — no bespoke controller frameworks.
- Reconcile via Crossplane Composition functions (Python) over a generic provider. Rejected: stretches Compositions beyond cloud-shaped resources and does not cleanly model the LiteLLM admin-API call surface.

## Related
- Architecture overview §5 (Custom kopf operator for LiteLLM row), §6.1 (Gateway), §14 (component B13).
- Architecture backlog §2.5 (this decision), §6 (controller-framework invariant), §7 item 6.
- ADR 0002 (OPA + Gatekeeper — the operator calls OPA on virtual key issuance, registration, budget decisions).
- ADR 0013 (CapabilitySet CRD — one of the kinds this operator reconciles).
- ADR 0020 (Initial MCP services — registered via `MCPServer` CRs reconciled by this operator).
- ADR 0021 (Dashboards as Crossplane XRs — the other side of the Crossplane / kopf split).
