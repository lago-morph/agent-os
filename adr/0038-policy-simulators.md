# ADR 0038: Policy simulators for OPA and RBAC

- **Status**: Accepted
- **Date**: 2026-05-10

## Context

The platform stacks multiple enforcement layers on every privileged
action: Gatekeeper evaluates OPA at admission for every CRD in the
§6.12 inventory (ADR 0002); LiteLLM consults OPA at runtime for tool,
model, MCP, A2A, and budget decisions; the Envoy egress proxy
consults OPA on every outbound connection (ADR 0003); Kubernetes
RBAC, mirrored by the §6.9 Keycloak claim schema, is the floor under
all of it (ADR 0018); and the approval system (ADR 0017) lets OPA
elevate the required approval level on top of that floor. HolmesGPT
(ADR 0012, §6.10) reads across these policies and routinely needs to
explain *why* a request was denied.

This composition is the architecturally desired behavior, but has a
predictable failure mode: a policy authored in one layer interacts
unexpectedly with a policy authored in another, and the failure is
hard to debug after the fact. A LiteLLM rule may permit a tool call
that Gatekeeper would have blocked at admission; an Envoy allow-list
may exclude an FQDN the runtime layer assumed reachable; an RBAC
narrowing may render an OPA elevation moot. Audit logs record the
eventual decision but not the counterfactual, and ADR 0027 names
"OPA bypass" and "capability escape" as primary attack patterns that
benefit from answering "what would happen for this exact request"
before an incident, not after.

## Decision

The platform ships a **policy simulator** surface that, given a
synthetic request — subject (Keycloak claims), action, resource,
context — returns the decision at every applicable enforcement layer
(admission, runtime, egress, RBAC, approval elevation) with the
matching rule cited. It is exposed two ways:

- A **Headlamp simulator panel** for interactive use, alongside the
  policy editing surfaces in ADR 0039.
- A **HolmesGPT skill** with the same semantics, callable inside
  diagnostic flows so HolmesGPT can answer "simulate the request
  that just got denied and tell me why" without a human in the loop.

Implementation is a thin **aggregator service** that fans the
synthetic request out to layer-specific dry-run paths and composes
the verdicts. OPA-backed components evaluate `data.allow` against
the supplied input bundle; Gatekeeper exposes its existing
audit-mode evaluation; Envoy exposes an ext-authz dry-run path; the
approval system evaluates elevation without creating an `Approval`.
Headlamp and HolmesGPT consume the same aggregator API.

The simulator is a **"what would happen for this case"** surface.
It is not formal verification, does not enumerate the policy space,
and does not replace the security review ADR 0027 requires.
Comprehensive policy verification is out of v1.0 scope.

## Consequences

- **Per-component deliverable.** Every component hosting a policy
  decision point — Gatekeeper, LiteLLM (callbacks and dynamic
  registration), Envoy egress, the approval system, Headlamp plugin
  actions, agent-platform CLI gates — must expose a structured
  dry-run, alongside audit emission and OPA hookup.
- **One API, two consumers.** Headlamp and HolmesGPT call the same
  aggregator, so a HolmesGPT explanation is reproducible in Headlamp.
- **Composes with ADR 0018.** RBAC is reported as a separate layer
  rather than collapsed into OPA, preserving the "RBAC is the
  ceiling, OPA narrows" contract; citations name which layer denied.
- **Composes with ADR 0017.** For approval-bearing actions the
  simulator reports the resolved required level (post-OPA-elevation)
  without creating an `Approval`.
- **Audit but not enforcement.** Simulator runs are audited under
  the `platform.policy.*` taxonomy but never enter the enforcement
  path — a simulator call never causes a real admission, runtime,
  or egress decision.
- **Bounded scope.** Point queries only; cross-layer conflict
  analysis and drift detection are future enhancements.

## References

- architecture-overview.md § 6.6 (security and policy architecture)
- architecture-overview.md § 6.10 (HolmesGPT self-management)
- ADR 0002 (OPA + Gatekeeper)
- ADR 0003 (Envoy egress proxy)
- ADR 0012 (HolmesGPT as a first-class Platform Agent)
- ADR 0017 (Generalized approval system)
- ADR 0018 (RBAC-floor / OPA-restrictor)
- ADR 0027 (Threat-model scope and B22)
- ADR 0039 (Headlamp policy editors)
