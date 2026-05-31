# ADR 0048: Two-layer egress control

- **Status**: Accepted
- **Date**: 2026-05-31

## Context

A sandboxed agent's outbound traffic must be constrained on two distinct axes:
in-cluster reachability (which pods and services it may talk to at all) and the
identity and shape of its external calls (which external hosts, over which scheme,
with which HTTP methods). These are different concerns enforced at different layers
of the stack, and one mechanism does not cover both.

The Kubernetes agent-sandbox primitive exposes a `NetworkPolicySpec` that renders
to native Kubernetes NetworkPolicy. Verified against the agent-sandbox API, this is
**L3/L4 only**: it selects traffic by IP/CIDR and port. It cannot allowlist by FQDN,
by URL scheme, or by HTTP method. The platform's egress requirements are explicitly
L7 — an agent's resolved `CapabilitySet` carries `egressTargets[]`, where each
`EgressTarget` names an `fqdn`, `port`, `scheme`, and `allowedMethods`. Native
NetworkPolicy cannot express any of `fqdn`, `scheme`, or `allowedMethods`. A single
layer therefore cannot satisfy the requirement.

## Decision

Egress is controlled in **two layers**, each enforcing what it is suited to.

1. **L7 external egress — the Envoy egress proxy.** Envoy enforces FQDN, HTTP-method,
   and scheme allowlisting for an agent's external traffic. Its allowlist is configured
   from the agent's resolved `CapabilitySet.egressTargets[]`: each `EgressTarget`
   (`fqdn` / `port` / `scheme` / `allowedMethods`) is reconciled into the Envoy
   allowlist by the kopf operator (B13). This is the layer that does the L7 work the
   sandbox `NetworkPolicySpec` cannot. Envoy stays precisely because the agent-sandbox
   `NetworkPolicySpec` is L3/L4 only and the platform requires FQDN and HTTP-method
   allowlisting.

2. **L3/L4 in-cluster baseline — the agent-sandbox `NetworkPolicySpec`.** The
   agent-sandbox `NetworkPolicySpec` provides the L3/L4 **default-deny baseline** for
   in-cluster traffic, rendered to native Kubernetes NetworkPolicy.

**The default sandbox egress allow-set is a minimum baseline — a floor, not a closed
or mandatory ceiling.** It is explicitly extensible: it is a starting set that design
work adds to, never a closed list, and membership beyond what is stated here is neither
mandated nor forbidden. The confirmed initial members are:

- the LiteLLM gateway (A1),
- the Envoy egress proxy (A6),
- the audit endpoint (A18),
- cluster DNS (CoreDNS),
- the memory backend (A10),
- the telemetry / trace collector (A2 / OTel),
- the event broker (B8).

Further targets are added as the design surfaces them.

The **audit endpoint (A18) is a platform component, not a sandboxed agent**, so it is
not itself subject to sandbox egress control. Envoy's **deployment topology** — sidecar
versus shared gateway — is a build-time detail and is **not** part of this decision.

## Consequences

- The two concerns map cleanly to the two layers: external call shape (L7) is owned by
  Envoy; in-cluster reachability (L3/L4) is owned by the sandbox `NetworkPolicySpec`.
  Neither layer is asked to do what it cannot.
- L7 egress policy is fully declarative: changing an agent's external reach is an
  `egressTargets[]` change on its `CapabilitySet`, reconciled into Envoy by B13 — not a
  hand-edit of Envoy config.
- Because the default allow-set is a floor, components can be added to it as design work
  proceeds without revising this decision; readers must not treat the list as closed or
  as a prohibition on unlisted targets.
- Treating Envoy topology as a build-time detail keeps this decision stable across
  sidecar/shared-gateway choices.

## References

- [DECISIONS-LOG.md](../_meta/reviews/DECISIONS-LOG.md) — "Egress (D-08)", the authoritative decision this ADR records
- [ADR 0003](./0003-envoy-egress-proxy.md) (Envoy egress proxy as CNI-agnostic egress control) — the L7 chokepoint this ADR pairs with the sandbox baseline
- [ADR 0013](./0013-capability-crd-and-capabilityset-layering.md) (Capability CRD and CapabilitySet layering) — `EgressTarget` and `egressTargets[]` live on the `CapabilitySet`
- [ADR 0032](./0032-capabilityset-overlay-semantics.md) (CapabilitySet overlay semantics) — per-Agent `EgressTarget` overrides compose via the overlay rules
- B13 (kopf operator) — reconciles `EgressTarget` → Envoy allowlist
- A6 (agent-sandbox + Envoy egress proxy) — the sandbox `NetworkPolicySpec` baseline and the Envoy proxy
- [ADR 0049](./0049-tenant-quota-model.md) (tenant quota model) — a sibling tenant-scoped control that, like egress, is expressed through the `CapabilitySet`
