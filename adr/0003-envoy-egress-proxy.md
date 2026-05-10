# ADR 0003: Envoy egress proxy as CNI-agnostic egress control

- **Status**: Accepted
- **Date**: 2026-05-10

## Context

The platform must run on EKS, AKS, and kind, so it cannot depend on any specific CNI being installed. It still needs uniform L7 egress control for all non-LiteLLM HTTP that Platform Agents emit: FQDN allowlisting, mTLS, audit emission to OpenSearch, and an OPA hook at the egress point. LiteLLM already covers LLM, MCP, and A2A traffic; everything else needs an equivalent chokepoint that is portable across the supported clusters and consistent with the rest of the policy and observability story (see architecture-overview.md §6.1 and §6.6).

## Decision

We deploy an Envoy-based egress proxy as the single egress chokepoint for all non-LiteLLM agent outbound HTTP. Per-agent-class configuration enforces the FQDN allowlist derived from each Platform Agent's `EgressTarget` capabilities, and Envoy calls OPA for runtime decisions and emits structured audit events on every connection. Envoy is installed via Helm and configured per agent class; CNI choice remains free.

## Alternatives considered

- **Cilium L7 NetworkPolicy** — Cilium isn't available across EKS, AKS, and kind, so adopting it would either constrain the supported cluster set or force a per-environment fork of egress policy. Envoy is CNI-agnostic and gives equivalent L7 control plus first-class audit and OPA hooks at the egress point (architecture-backlog.md §2.3).

## Consequences

- Portability: identical egress enforcement on EKS, AKS, and kind; no CNI dependency, consistent with the independent-cluster install model.
- Uniform policy surface: OPA decides at admission, at the LiteLLM gateway, and at the Envoy egress proxy — same Rego authoring story for all three.
- Uniform audit: every egress connection emits to the OpenSearch audit index alongside LiteLLM and admission events.
- Reinforces the invariant that **every Platform Agent outbound call traverses LiteLLM or the Envoy egress proxy — there are no other paths to external services** (architecture-backlog.md §6).
- New operational surface: an Envoy install plus per-agent-class config to own, monitor, and upgrade.
- Egress policy is driven by the `EgressTarget` CRD and CapabilitySet composition; adding a destination is a declarative capability change, not an Envoy-config edit by hand.
- Defense-in-depth network-layer enforcement for cross-tenant A2A: even if gateway authorization fails, Envoy blocks the call (architecture-overview.md §6.9).

## References

- architecture-overview.md §6.1 (gateway), §6.6 (security and policy), §6.8 (capabilities), §6.9 (multi-tenancy)
- architecture-backlog.md §2.3, §6, §7 item 3
- ADR 0002 (OPA + Gatekeeper) — Envoy uses OPA for runtime egress decisions
- ADR 0013 (Capability CRD) — `EgressTarget` is the declarative input to Envoy config
- ADR 0026 (independent-cluster install) — motivates CNI-agnostic choice
- ADR 0033 (AWS + GitHub initial targets) — EKS is the v1.0 target; AKS and kind must remain supported
