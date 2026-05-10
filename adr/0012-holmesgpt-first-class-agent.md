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
like any other agent, with broad **read-only** access to platform state (OPA
policies, audit, traces, metrics, Knowledge Base) and an A2A interface so
other Platform Agents can hand off troubleshooting questions. Write actions
are policy-controlled: v1.0 ships read-only diagnostics plus auto-PR /
Headlamp suggestion-card flows, with autonomous narrow-scope remediations
gated until trust is established. The Knowledge Base
(`platform-knowledge-base` RAGStore) is a separate primitive that HolmesGPT
uses, not part of HolmesGPT — any Platform Agent that includes it in its
CapabilitySet has the same access via the SDK's `rag.*` API.

## Consequences

- Establishes the **"every component contributes a HolmesGPT toolset"**
  invariant (architecture-backlog.md § 6) as a standard component deliverable
  alongside OPA contribution, audit emission, observability, Knative trigger
  flow design, tests, and how-to docs. By the time the platform is mature,
  HolmesGPT has a complete picture by construction rather than by retrofit.
- Wires the **AlertManager → HolmesGPT** Knative trigger flow as one of the
  two initial v1.0 trigger flows that exercise the eventing fabric end to
  end (overview § 6.7): AlertManager alerts route as CloudEvents, a Trigger
  filters for diagnostic-relevant ones (unschedulable pods, high error rates,
  sustained gateway latency) and dispatches to HolmesGPT, which produces
  findings, suggestions, or auto-PRs through its normal flow.
- v1.0 ships **read-only diagnostics + auto-PR / Headlamp suggestion cards**
  routed through the generalized Approval system (ADR 0017); HolmesGPT-
  suggested remediation actions are one of the two initial use cases the
  Approval system is built to support.
- Calls into and out of HolmesGPT traverse LiteLLM, are subject to OPA gating
  (ADR 0002), emit audit to OpenSearch (ADR 0009), and run inside a sandbox
  like any tenant agent — no parallel governance surface for the platform's
  own self-management agent.
- HolmesGPT is sequenced **early** in Workstream A (item A14) so other
  components can contribute toolsets and emit traces from the start; this
  also means it is useful as a diagnostic aid during platform implementation
  itself, with its toolset coverage growing as more components land.
- Accepts an explicit self-referential trade-off: HolmesGPT has read access
  to the OPA policies that gate it so it can analyze them for loopholes and
  contradictions. We would rather find and fix self-bypass paths than leave
  them undiscovered.
- **Revisit trigger** (architecture-backlog.md § 3.11): a remediation pattern
  we trust enough to gate behind OPA but execute autonomously. At that point
  HolmesGPT's autonomy expands from auto-PR/suggestion-card to OPA-gated
  autonomous narrow-scope remediation, and the SLO-breach response flow
  described in future-enhancements.md § 3 becomes the natural next step.

## References

- architecture-overview.md § 6.7, § 6.10
- architecture-backlog.md § 3.11, § 6
- future-enhancements.md § 3 (SLO breach response via HolmesGPT)
