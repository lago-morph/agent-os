# ADR 0013: Capability CRD model and CapabilitySet layering

## Status

Accepted

## Context

A Platform Agent's reachable surface — which MCP servers it may call, which
tools those servers expose, which RAG stores it can query, which egress
targets it may reach, which OPA policy snippets gate its requests, which
LLM providers it may invoke, which secret references it carries — touches
every cross-cutting concern in the platform: gateway routing, egress
allowlists, observability tagging, admission policy, secret distribution,
budget scoping, Headlamp visualization. Treating that surface as
gateway-internal config (a LiteLLM-only concern) would scatter the same
list of "what this agent is allowed to do" across many components with no
single declarative source.

The architecture overview §6.8 establishes a CRD-based capability model
reconciled by the kopf operator (ADR 0006), referenced by Agent CRDs
(ADR 0001). What remains is to lock the model itself — the CRDs and the
composition primitive — separately from the many merge-rule details that
are properly design-time decisions.

Backlog §1.1 enumerates those deferred details: recursive vs one-level
includes, per-field merge semantics (additive vs replace vs per-element),
admission validation, cross-namespace references, and whether per-Agent
overrides may grant capabilities not present in any referenced set. This
ADR does not resolve them.

## Decision

Capabilities are first-class Kubernetes CRDs, not gateway-internal config.
The set is: `MCPServer`, `A2APeer`, `RAGStore`, `EgressTarget`, `Skill`,
plus `CapabilitySet` as the composition primitive and `Agent` as the
consumer.

A **CapabilitySet** bundles references to MCP servers (with their tools),
A2A peers, RAG stores, OPA policy snippets, egress targets, secret
references, and LLM providers. It is the unit of capability composition.

CapabilitySets compose using a **Helm-style values overlay** model:

- An Agent CRD references an ordered list of CapabilitySets.
- Sets apply in order; later sets override earlier ones.
- **Per-Agent overrides** on the Agent CRD itself take final precedence.
- The resolved capability surface for an agent is the merged result.

Reconciliation is one-way: the kopf operator (ADR 0006) projects resolved
capability state into LiteLLM, the Envoy egress proxy allowlist, OPA
policy bundles, and the Headlamp per-agent unified view. Kubernetes CRDs
remain the source of truth.

Detailed merge semantics — the syntax for additive vs replace markers,
per-field rules across MCP servers, tools, RAG stores, OPA policy
snippets, egress targets, secret references, and LLM providers, recursive
inclusion, cross-namespace references, override-grants-new-capability
rules, deletion behaviour — are deferred to design (backlog §1.1). This
ADR locks the model and the primitive, not every merge rule.

## Consequences

Positive:

- One declarative source describes an agent's full capability surface;
  every downstream component (gateway, egress, policy, observability,
  Headlamp) reads from the same CRDs.
- A library of CapabilitySet profiles (B17) becomes a real reuse surface
  rather than copy-paste between Agent specs.
- Admission, OPA, and audit have a stable schema to bind to.
- Per-agent overrides preserve the escape hatch without forcing every
  variant into the shared library.

Negative / accepted:

- Resolving the effective capability set requires merge logic; every
  reader must agree on the resolved view (the kopf operator owns this).
- Layering invites subtle precedence bugs; mitigated by Headlamp showing
  the resolved per-agent view and by admission validation once §1.1 is
  designed.
- Several real questions (recursive includes, per-field merge, deletion,
  cross-namespace) are deliberately left open here and must be closed
  before implementation.

## Alternatives considered

- **Capability config lives inside LiteLLM only.** Rejected: it is not
  only a gateway concern — egress, policy, observability, Headlamp all
  need the same declarative view.
- **Flatten everything onto the Agent CRD with no CapabilitySet
  primitive.** Rejected: every agent re-declares the same baselines;
  no reuse surface; profile library (B17) becomes unworkable.
- **Strict inheritance (single parent, no layering).** Rejected: real
  agents combine orthogonal concerns (a baseline + a domain overlay +
  an environment overlay); single-parent inheritance forces duplication.
- **Free-form patches (JSON Patch / strategic merge per Agent).**
  Rejected: too low-level for a profile library; defeats the point of
  named, reviewable CapabilitySets.

## Related

- ADR 0001 — ARK Agent CRDs reference CapabilitySets.
- ADR 0006 — kopf operator reconciles CapabilitySet state to LiteLLM,
  Envoy egress, and OPA bundles.
- ADR 0018 — OPA policy snippets contributed via capabilities.
- ADR 0019 — SDK selection lives inside CapabilitySets.
- ADR 0025 — memory access modes referenced from CapabilitySets.
- Architecture overview §6.1 (gateway), §6.2 (agent runtime), §6.8
  (capability registries and approved primitives).
- Architecture backlog §1.1 (deferred merge semantics), §6 (invariants),
  §7 item 13 (this ADR).
