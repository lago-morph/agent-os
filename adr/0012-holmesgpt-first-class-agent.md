# ADR 0012: HolmesGPT as a first-class Platform Agent

- **Status**: Accepted
- **Date**: 2026-05-10

## Context

The platform needs a self-management story for diagnostics, root-cause
analysis, and remediation suggestions, and it needs one from day one —
diagnostic value during platform implementation itself is real, even before
the full observability stack is wired. Running HolmesGPT as an external
operator tool would create a parallel surface that bypasses LiteLLM, OPA,
audit, and the sandbox model — exactly the governance the platform exists
to provide. Running it on-platform forces the architecture to eat its own
dog food: every constraint that applies to a tenant agent (gateway routing,
policy gating, audit emission, capability registries, identity claims)
applies to the platform's own diagnostic agent. This also lets HolmesGPT
grow with the platform — each new component contributes a toolset as it
lands, rather than HolmesGPT being retrofitted with knowledge later.

## Decision

HolmesGPT runs on the platform as a Platform Agent (Agent CRD), administered
like any other agent, with an A2A interface so other Platform Agents can
hand off troubleshooting questions. The Knowledge Base
(`platform-knowledge-base` RAGStore) is a separate primitive that HolmesGPT
uses, not part of HolmesGPT — any Platform Agent that includes it in its
CapabilitySet has the same access via the SDK's `rag.*` API.

### Three-state permission model via OPA

Every HolmesGPT action resolves to exactly one of three states at a single,
centrally-managed OPA decision point:

- **allowed** — execute autonomously.
- **upon-approval** — route through the generalized Approval system
  (ADR 0017) and execute only after a human approves.
- **denied** — refuse and emit audit.

OPA expresses the model with fine-grained conditions (time-of-day, resource
scope, target namespace, identity claims, change-window state, etc.). For
example: "allowed unless touching configurations X / Y / Z during business
hours, in which case upon-approval." All HolmesGPT decisions go through
**one** OPA decision point, edited **centrally** — no per-deployment
policy drift, no parallel allowlist embedded in HolmesGPT itself. Policies
are authored and maintained as OPA bundles via the Headlamp graphical
editors (ADR 0039), and changes are validated with the policy simulator
(ADR 0038) before submission.

### Phased deployment trajectory

HolmesGPT is sequenced **very early** in v1.0 implementation — deployed
into the raw Kubernetes cluster before most platform components exist, so
it can help with the implementation and pilot work itself. During this
bootstrap phase, the guardrails are blunt and external rather than
platform-mediated:

- A read-only ServiceAccount on the cluster.
- A read-only AWS IAM role with **no** access to secret stores or other
  sensitive data planes.
- No write paths.

As platform components land (LiteLLM, the audit adapter, the OPA decision
point, the Approval system, the sandbox runtime), HolmesGPT's access is
**rewired** through those normal layers rather than the bootstrap shortcut.
The endpoint moves behind LiteLLM; calls become subject to OPA gating
(ADR 0002); every action emits audit through the standard adapter
(ADR 0034); A2A handoffs go through the gateway. Read-only stays the
default throughout. Expanding into write actions happens **incrementally**,
one capability at a time, as trust is established (cross-ref
architecture-backlog.md § 3.11).

## Consequences

- Establishes the **"every component contributes a HolmesGPT toolset"**
  invariant (architecture-backlog.md § 6) as a standard component
  deliverable alongside OPA contribution, audit emission, observability,
  Knative trigger flow design, tests, and how-to docs. By the time the
  platform is mature, HolmesGPT has a complete picture by construction
  rather than by retrofit.
- Wires the **AlertManager → HolmesGPT** Knative trigger flow as one of
  the two initial v1.0 trigger flows that exercise the eventing fabric
  end to end (overview § 6.7): AlertManager alerts route as CloudEvents,
  a Trigger filters for diagnostic-relevant ones (unschedulable pods,
  high error rates, sustained gateway latency) and dispatches to
  HolmesGPT, which produces findings, suggestions, or auto-PRs through
  its normal flow.
- v1.0 ships **read-only diagnostics + auto-PR / Headlamp suggestion
  cards**; any action OPA classifies as upon-approval is routed through
  the generalized Approval system (ADR 0017). HolmesGPT-suggested
  remediation actions are one of the two initial use cases the Approval
  system is built to support.
- HolmesGPT **drives the dynamic diagnostics toggle** (ADR 0035) — when a
  diagnostic session needs deeper signal, HolmesGPT requests the toggle
  through the same OPA-gated path as any other write action.
- Calls into and out of HolmesGPT traverse LiteLLM, are subject to OPA
  gating (ADR 0002), emit audit through the platform audit adapter to
  the durable Postgres + S3 system of record (ADR 0034; OpenSearch is
  an advisory fanout per ADR 0009), and run inside a sandbox like any
  tenant agent — no parallel governance
  surface for the platform's own self-management agent. The platform
  threat model (ADR 0027) treats HolmesGPT as in-scope, not privileged.
- Accepts an explicit self-referential trade-off: HolmesGPT has read
  access to the OPA policies that gate it so it can analyze them for
  loopholes and contradictions. We would rather find and fix self-bypass
  paths than leave them undiscovered. The policy simulator (ADR 0038)
  is HolmesGPT's primary tool for this analysis.
- **Revisit trigger** (architecture-backlog.md § 3.11): each time a
  remediation pattern earns enough trust to graduate from upon-approval
  to allowed, OPA is updated centrally via the Headlamp editor
  (ADR 0039) and HolmesGPT's autonomy expands one increment. The
  SLO-breach response flow described in future-enhancements.md § 3 is
  the natural endpoint of this trajectory.

## References

- ADR 0002 (OPA gating)
- ADR 0017 (Generalized Approval system)
- ADR 0027 (Platform threat model)
- ADR 0034 (Audit emission adapter)
- ADR 0035 (Dynamic diagnostics toggle)
- ADR 0038 (OPA policy simulator)
- ADR 0039 (Headlamp graphical policy editors)
- architecture-overview.md § 6.7, § 6.10
- architecture-backlog.md § 3.11, § 6
- future-enhancements.md § 3 (SLO breach response via HolmesGPT)
