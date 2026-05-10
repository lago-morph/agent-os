# ADR 0002: OPA + Gatekeeper as the policy engine

## Status
Accepted

## Context
The platform needs policy enforcement at two distinct points: Kubernetes admission (admit/deny on Agent, Sandbox, Memory, MCPServer, A2APeer, RAGStore, CapabilitySet, VirtualKey, BudgetPolicy CRs) and runtime decisions on every LiteLLM call, every egress hop through the Envoy egress proxy, every dynamic agent registration, every Approval, and every virtual key issuance. These decisions are core to the platform's "governed, observable, policy-controlled" mandate.

OPA can serve both: Gatekeeper handles admission, and OPA's runtime decision API is consumed by LiteLLM callbacks, the Envoy egress proxy, and the kopf operator that reconciles VirtualKey, BudgetPolicy, and CapabilitySet state. A single policy language (Rego) and a single bundle distribution (Rego bundles reconciled from Git by ArgoCD) covers both surfaces.

The realistic alternative was Kyverno for admission alongside OPA for runtime. Kyverno offers a friendlier admission DSL and a built-in image signature verification path. Splitting admission from runtime would, however, leave us writing and operating two policy languages, two bundle pipelines, and two test harnesses for what is conceptually one decision plane.

## Decision
Use OPA + Gatekeeper as the single policy engine for the platform. Gatekeeper enforces all Kubernetes admission. OPA is consulted at runtime by LiteLLM callbacks, the Envoy egress proxy, the kopf operator, and the Approval system. Policies are authored as Rego bundles, versioned in Git, reconciled by ArgoCD, and unit-tested in CI.

## Consequences
Positive:
- One policy language (Rego), one bundle pipeline, one test harness across admission and runtime.
- LiteLLM callbacks, Envoy egress hooks, and Gatekeeper admission share policy data, making the RBAC-as-floor / OPA-as-restrictor model (ADR 0018) coherent end-to-end.
- Cost budgets, virtual key issuance, Approval elevation, and capability composition all consult the same OPA, with Headlamp as the unified editing surface.
- HolmesGPT can read the same Rego bundles it would otherwise have to model indirectly.

Negative:
- Rego is code; we commit to engineers who can write and test it, not to YAML editors.
- We lose Kyverno's built-in image signature verification path. Image signature verification is a known gap (backlog §3.9).
- Two control loops touch policy artifacts: Gatekeeper for admission and the kopf operator for runtime data; both must stay in sync via the same Git source of truth.

## Alternatives considered
- Kyverno for admission + OPA for runtime. Rejected: two policy languages and two pipelines for one decision plane.
- Kyverno alone. Rejected: no usable runtime decision surface for LiteLLM callbacks or the egress proxy.
- OPA without Gatekeeper, using a custom admission webhook. Rejected: Gatekeeper is the well-trodden Kubernetes admission integration for OPA; rebuilding it adds risk for no gain.

## Related
- Architecture overview §5 (software list — OPA / Gatekeeper entry), §6.6 (Security and policy architecture), §14 (component matrix).
- Architecture backlog §2.2 (this decision), §3.9 (image signature verification trigger to add Kyverno, Sigstore policy-controller, or Connaisseur), §6 (invariants), §7 (decision register).
- ADR 0003 (Envoy egress proxy — calls OPA on every egress).
- ADR 0017 (Generalized Approval system — OPA elevation only).
- ADR 0018 (RBAC-as-floor / OPA-as-restrictor — the enforcement model that depends on OPA).
