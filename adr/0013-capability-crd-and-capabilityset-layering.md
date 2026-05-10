# ADR 0013: Capability CRD model and CapabilitySet layering

- **Status**: Accepted
- **Date**: 2026-05-10

## Context

The platform needs a single, declarative way to describe what a Platform Agent
is allowed to reach: MCP servers, A2A peers, RAG stores, egress targets,
skills, OPA policy snippets, and LLM providers. Treating each of these as
gateway-internal configuration would scatter governance across LiteLLM,
Envoy, and ad-hoc admin UIs, and would prevent reuse, audit, and policy
enforcement from working off the same source of truth (architecture-overview.md
§6.8). The architecture commits instead to capabilities as first-class
Kubernetes CRDs reconciled from Git, with a `CapabilitySet` primitive that
bundles and layers them so Agents compose from a library of profiles rather
than restating capabilities per-agent. The model also needs to be the same
surface that OPA, RBAC, Headlamp, observability tagging, and the audit
pipeline read from (architecture-backlog.md §6 invariants).

## Decision

Each approved capability — `MCPServer`, `A2APeer`, `RAGStore`,
`EgressTarget`, `Skill` — is a namespaced CRD reconciled by the custom
Python kopf operator (B13) into LiteLLM and adjacent components.
`CapabilitySet` is a namespaced CRD that bundles references to those
capabilities plus `llmProviders[]` and `opaPolicyRefs[]`, and is itself
layerable. An `Agent` references one or more CapabilitySets via
`capabilitySetRefs[]` and may carry per-Agent `overrides`. The existence
and shape of these CRDs is the architectural commitment; detailed overlay
resolution semantics are deferred to ADR 0032 and design specs.

## Consequences

- **One reusable primitive across the platform.** The same CRD set is the
  source of truth for LiteLLM registry contents, Envoy egress allowlists,
  OPA admission and runtime decisions, Headlamp's per-agent unified view,
  audit tagging, and observability labels (architecture-overview.md §6.8,
  §5 component table B5/B13).
- **Governance hooks at well-defined points.** Gatekeeper admits all
  capability CRDs; OPA gates dynamic agent registration and virtual-key
  issuance against `capability_set_refs` JWT claims; RBAC scopes who may
  create or bind sets — RBAC-floor / OPA-restrictor applies uniformly
  (architecture-overview.md §6.8, §6.9; ADR 0018).
- **Composition over duplication.** Agent profiles (B17) ship as
  CapabilitySets that downstream Agents reference and overlay rather than
  copy, keeping policy and budget surfaces consistent across the fleet.
- **Headlamp resolved-view requirement.** Because capabilities arrive
  via layered sets plus overrides, the operator UI must render the
  resolved per-Agent view from CRDs and LiteLLM applied state — this is
  load-bearing for operability, not optional (architecture-overview.md
  §6.8, B5).
- **Cross-tenant sharing is opt-in.** A CapabilitySet defined in one
  namespace is referenced from another only with explicit OPA-checked
  publication; the default is namespace-private
  (architecture-overview.md §6.9).
- **Deferred sub-semantics.** Recursive CapabilitySet inclusion,
  validation on missing or deleted references, namespace-boundary
  rules for cross-references, and whether Agent overrides may grant
  capabilities not present in any referenced set are all design-time
  per architecture-backlog.md §1.1 and resolved in ADR 0032 and
  follow-on design specs.
- **Versioning and ownership.** Capability CRDs follow the platform's
  CRD/API versioning policy (ADR 0030) and are owned by the kopf
  operator component (B13) for reconciliation lifecycle.
- **Capability-change notification model.** When an Agent definition or
  any referenced CapabilitySet changes (overlay rules per ADR 0032),
  enforcement points update immediately, but running agents may already
  be in a turn that assumed prior permissions. The platform handles this
  with two mechanisms working together:
  - *CloudEvent fanout (primary):* kopf reconciliation emits a
    `platform.capability.changed` CloudEvent (under the existing
    taxonomy in ADR 0031) carrying the affected Agent / CapabilitySet
    identifiers. The platform SDK (ADR 0019) subscribes and surfaces a
    callback / interrupt / next-turn notification to the agent code.
  - *Poll fallback (always available):* the SDK exposes a
    `refresh_capabilities()` API for runtimes that can't subscribe to
    CloudEvents. Worst-case staleness is one turn.

  This is **best-effort**, not a hard correctness guarantee —
  enforcement still happens at LiteLLM / OPA / Envoy regardless. The
  point is to avoid the agent thinking it has access to something,
  then trying to use it, and getting denied with a confusing error.

## References

- architecture-overview.md § 6.8
- architecture-backlog.md § 1.1, 6
- ADR 0019 (platform SDK is the agent-facing surface for capability
  notifications)
- ADR 0031 (CloudEvent taxonomy carrying `platform.capability.changed`)
- ADR 0032 (CapabilitySet overlay semantics)
- ADR 0018 (RBAC-floor / OPA-restrictor)
- ADR 0039 (Headlamp surfaces the resolved capability set live)
