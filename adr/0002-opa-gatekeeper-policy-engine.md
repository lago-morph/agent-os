# ADR 0002: OPA + Gatekeeper as the policy engine

- **Status**: Accepted
- **Date**: 2026-05-10

## Context

The platform needs both Kubernetes admission control (Agent, AgentRun, Sandbox, MCPServer, A2APeer, CapabilitySet, VirtualKey, BudgetPolicy, Approval, and Crossplane XRs) and runtime authorization decisions for LiteLLM callbacks (per-request tool/model authorization, dynamic A2A/MCP registration, budget enforcement against `BudgetPolicy` CRDs). Runtime decisions inside the LiteLLM Python callback chain are unavoidable regardless of which engine handles admission. The architecture also commits to a defense-in-depth model where policy is consulted at admission, at the gateway, at the Envoy egress proxy, and at Headlamp action surfaces — all of which benefit from a single decision API and a single policy language.

## Decision

The platform uses Open Policy Agent as the policy decision engine and Gatekeeper as its Kubernetes admission integration. All admission policies, LiteLLM callback decisions, Envoy egress decisions, Headlamp action gating, approval-level elevation, and virtual-key issuance checks evaluate against OPA. Policies are authored as Rego bundles, versioned in Git, reconciled by ArgoCD, and shipped through the OPA policy library (framework in B3, initial content in B16).

## Alternatives considered

- **Kyverno** — Rejected because OPA's runtime decision capability is required for LiteLLM callbacks regardless of admission choice; adopting Kyverno would force the platform to maintain two policy languages and two authoring/testing toolchains for the same governance surface (backlog § 2.2).

## Consequences

- One policy language (Rego) and one decision API across admission, gateway runtime, egress, Headlamp, and approvals — consistent authoring, testing, and audit.
- LiteLLM callbacks (PII, audit, OPA bridge, guardrails, budget enforcement) consult OPA directly; the OPA bridge callback is mandatory and ships with A7 if it lands before A1 (per backlog component plan).
- The RBAC-floor / OPA-restrictor invariant (architecture-overview.md § 6.6, § 11.7, and ADR 0018) is enforceable in one place: OPA may only further restrict what RBAC has granted, never grant additional permissions.
- Image signature verification — Kyverno's strong out-of-the-box path — is **not** available. The platform accepts this gap for v1.0; supply-chain image-signature requirements trigger a revisit per backlog § 3.9 (add Kyverno specifically for image verification, or adopt Sigstore policy-controller, or Connaisseur).
- Rego is treated as code: Rego unit tests run in CI (architecture-overview.md § 9), bundles are SHA-pinned, and the OPA policy library has its own framework component (B3) and initial-content component (B16).
- No central policy management UI ships with OPA/Gatekeeper; the platform compensates with a Headlamp plugin for policy review and bundle management (architecture-overview.md § 10 vendor-gap table).
- Terminology: Gatekeeper *is* OPA's Kubernetes admission controller. When this architecture refers to "OPA admission control", that means Gatekeeper specifically — there is no separate OPA-only admission path.
- Gatekeeper's audit mode (continuous re-evaluation of cluster state against constraints) is one of the data sources consumed by the policy simulator in ADR 0038, alongside dry-run admission and the LiteLLM callback shadow channel.
- Detail of the Rego policy library structure, bundle layout, and per-component contribution conventions is deferred to design (backlog § 4, B3/B16 scope).

## References

- [architecture-overview.md](../architecture-overview.md) [§ 6.6](../architecture-overview.md#66-security-and-policy-architecture) (security and policy architecture, defense in depth, audit and OPA hook points), § 11.7 (RBAC-floor / OPA-restrictor for approvals), [§ 14.2](../architecture-overview.md#142-workstream-b--custom-platform-development) (components A7, B2, B3, B16)
- [architecture-backlog.md](../architecture-backlog.md) [§ 2.2](../architecture-backlog.md#22-policy-engine-kyverno-vs-opa-gatekeeper), [§ 3.9](../architecture-backlog.md#39-image-signature-verification), [§ 6](../architecture-backlog.md#6-architecture-level-invariants-worth-documenting-as-adrs)
- [ADR 0018](./0018-rbac-floor-opa-restrictor.md) (RBAC-floor / OPA-restrictor enforcement)
- [ADR 0003](./0003-envoy-egress-proxy.md) (Envoy egress proxy — consumes OPA decisions at the egress hook)
- [ADR 0017](./0017-generalized-approval-system.md) (Approval system — OPA elevation of required approval level)
- [ADR 0038](./0038-policy-simulators.md) (Policy simulators — uses Gatekeeper audit mode as a data source)
