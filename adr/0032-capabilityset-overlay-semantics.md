# ADR 0032: CapabilitySet overlay semantics

- **Status**: Accepted
- **Date**: 2026-05-10

## Context

`CapabilitySet` is the layered bundling primitive that grants Platform
Agents access to MCP servers, A2A peers, RAG stores, egress targets,
skills, OPA policy snippets, and LLM providers (ADR 0013). Agents
reference one or more `CapabilitySet`s and may add per-Agent overrides,
so the platform needs a single, unambiguous resolution rule that
produces the same effective capability set every time the same inputs
are presented. Predictability is a security property as much as an
ergonomics one: admission validation, OPA decisions, virtual-key claim
binding, and the Headlamp per-Agent unified view all read the resolved
set, and a non-deterministic merge would make security review and audit
unsound. Architecture-overview.md § 6.8 commits the platform to a
Helm-style values-overlay model and walks through a worked example;
this ADR records that commitment as a platform invariant.

## Decision

CapabilitySet resolution is a **field-level overlay**. The rule is
**add-if-not-there / replace-if-there**: for each top-level field in a
`CapabilitySet`, if the field is not yet present in the resolved state
it is added, and if it is present it is fully replaced (lists are not
concatenated — to extend a parent list, an overlay restates the full
list it wants). Referenced `CapabilitySet`s are processed **one at a
time, in declared order**, and **per-Agent overrides apply last** under
the same rule. This produces a single deterministic resolved capability
set per Agent.

## Consequences

- Resolution is deterministic and order-defined: given the same Agent
  spec and the same set of referenced `CapabilitySet`s, every consumer
  (kopf operator, OPA, Headlamp, audit) computes the same effective
  capability set, which is a precondition for sound security review.
- Replace-not-merge for lists is explicit, so authors of overlays must
  restate the full intended list (illustrated by the § 6.8 worked
  example); this trades a small ergonomic cost for elimination of merge
  ambiguity and surprising additive composition.
- The kopf operator (ADR 0006) reconciles the resolved state to
  LiteLLM, and Gatekeeper (ADR 0002) validates the resolved set at
  admission time; both rely on this rule being the single resolution
  algorithm in the platform.
- Several deeper sub-decisions are deferred to design-time per
  architecture-backlog.md § 1.1: whether a `CapabilitySet` may include
  other `CapabilitySet`s (and if so, recursion depth); validation rules
  for missing or deleted references; whether namespace boundaries apply
  to cross-references; and whether per-Agent overrides may grant
  capabilities not present in any referenced set, or must be a subset
  of what those sets provide.
- Per-Agent overrides applying **last** means an Agent author can
  always specialize the resolved set, subject to whatever subset/grant
  rule § 1.1 eventually pins down; OPA remains the authoritative
  restrictor at decision time (ADR 0018) regardless of what overlay
  authors write.
- Any future change to the overlay rule (switching list semantics,
  changing override ordering, formalizing recursion) is a breaking
  change to every consumer of the resolved set and MUST go through the
  CRD/API versioning discipline of ADR 0030.
- The Headlamp per-Agent unified view (§ 6.8) renders the resolved
  set, not the layered inputs, so operators reviewing an Agent see
  exactly what the platform enforces rather than having to mentally
  replay the overlay.

## References

- architecture-overview.md § 6.8
- architecture-backlog.md § 1.1, 6
- ADR 0013 (Capability CRD model), ADR 0018 (RBAC-floor / OPA-restrictor)
