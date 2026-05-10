# ADR 0018: RBAC as the floor, OPA as the restrictor

## Status
Accepted

## Context
The platform makes access decisions at two distinct surfaces: Kubernetes admission (does this principal have the right to create / read / update / delete this CR at all?) and runtime (given current spend, time of day, request shape, tenant pairing, and approval state, may this specific call proceed right now?). Two engines actually evaluate these decisions — Kubernetes RBAC at the API server, and OPA at admission via Gatekeeper plus runtime via LiteLLM callbacks, the Envoy egress proxy, and the kopf operator (see ADR 0002).

Without an explicit composition rule between them, the same decision can be made coherent in two contradictory ways: OPA could grant access RBAC denied (an OPA bug or an over-broad bundle would silently widen the API surface), or RBAC could be treated as advisory and overridden by Rego. Either direction breaks the least-privilege story and makes audits ambiguous: a reviewer cannot tell, from policy alone, who can do what.

The platform-wide invariant from architecture-backlog §6 — "RBAC is the floor; OPA may raise the floor on a per-decision basis. OPA never grants permissions RBAC didn't already give." — exists precisely to remove that ambiguity. This ADR formalizes the invariant as a binding architectural rule that all enforcement points obey.

## Decision
Adopt a strict two-layer composition for every access decision in the platform:

1. **Kubernetes RBAC is the floor.** It is the necessary condition. If RBAC denies the action — at API admission, on a get / list / watch / create / update / delete against the API server — the action is denied. Nothing downstream can revive it.
2. **OPA is the restrictor.** OPA Gatekeeper (admission policy) and OPA via LiteLLM callbacks, Envoy egress, and the kopf operator (runtime decisions) may further restrict, on a per-decision basis, what RBAC allowed. They take into account context RBAC cannot see: budget remaining, tenant pairing, time window, approval state, request payload shape, capability binding, egress destination.
3. **OPA never grants.** OPA bundles MUST NOT contain rules that synthesize permissions absent from RBAC. If OPA returns `allow`, that is a non-veto; the underlying right must already exist in RBAC. OPA's only output that changes outcomes is `deny` (or a more restrictive variant such as `allow_with_constraints`).

Concretely: every CR kind the platform defines (Agent, Sandbox, MemoryStore, MCPServer, A2APeer, RAGStore, CapabilitySet, VirtualKey, BudgetPolicy, Approval) ships with both a RoleBinding/ClusterRoleBinding template and a Gatekeeper constraint template. The RBAC binding establishes who may touch the kind at all; the constraint encodes the per-decision restrictions. Runtime decision points (LiteLLM, Envoy, kopf) follow the same shape: the caller must already hold the underlying right (virtual key bound to the capability, agent identity in the namespace), and OPA's job is to say "not right now" or "not for this request."

Policy authoring conventions enforce this: Rego bundles are reviewed against a checklist that disallows rules whose effect is to grant. Bundle CI includes a test that injects "RBAC denies" cases and asserts OPA cannot reverse them.

## Consequences
Positive:
- One mental model for every reviewer, auditor, and policy author: read RBAC for "what is possible," read OPA for "what is currently allowed."
- Per-decision context (budget, time, tenant, approval) lives in OPA where it belongs; static role grants live in RBAC where they belong.
- Defense in depth is real: a buggy OPA bundle cannot widen access beyond RBAC; a missing OPA constraint cannot bypass RBAC.
- The invariant is testable in CI — bundles that attempt to grant fail the policy lint.

Negative:
- Authors must remember to provision the RBAC binding alongside the OPA constraint; forgetting RBAC leaves a feature unreachable even with a correct constraint. This is mitigated by templating both together.
- Some "elevation" patterns (Approval-driven temporary access, ADR 0017) require RBAC bindings broad enough to cover the elevated case, with OPA closing the gap by default. This is intentional but demands careful RBAC scoping.

## Alternatives considered
- **OPA as both grant and restrict.** Rejected: duplicates RBAC, gives Rego unilateral authority over the API surface, weakens least privilege, and makes audit non-local (you must read every Rego bundle to know who can do what).
- **RBAC only, no OPA composition.** Rejected: RBAC has no per-decision context. Budgets, time windows, tenant pairing, and request-shape checks have nowhere to live; runtime governance disappears.
- **RBAC for admission, OPA for runtime, with no composition rule.** Rejected: leaves the grant/restrict relationship implicit; the same ambiguity returns the moment a single decision touches both surfaces (e.g., dynamic agent registration).

## Related
- Architecture overview §6.6 (Security and policy architecture), §6.9 (namespace tenancy enforcement layers).
- Architecture backlog §6 (invariant: RBAC is the floor; OPA never grants).
- ADR 0002 (OPA + Gatekeeper as the single policy engine — provides the OPA layer this ADR governs).
- ADR 0016 (namespace tenancy — its three enforcement layers compose under this model).
- ADR 0017 (generalized Approval system — elevation expressed as OPA loosening its own restriction within RBAC's floor, never crossing it).
