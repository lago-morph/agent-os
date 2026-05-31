# Implementation Plan: OPS4 — Isolation (Trimmed)

> Status: Draft (trimmed scope)
> Workstream: OPS (Operations & Platform Engineering)
> Component ID: OPS4

---

## Scope

Only the agent-sandbox isolation slice survives (per D-08): the NetworkPolicy
L3/L4 default-deny baseline for sandbox pods, Envoy L7 external egress, and the
extensible sandbox egress floor. Container runtime class (#6), node-level
isolation (#7), and general tenant network isolation (#8) are baseline/cluster
concerns and are not implemented here. The `AgentEnvironment.quotas` field is
owned by OPS1 (cross-reference only).

## Work Items

1. **Default-deny NetworkPolicy.** Apply an L3/L4 default-deny `NetworkPolicy`
   to sandbox pods.
2. **Envoy L7 egress.** Route sandbox external egress through Envoy; deny direct
   external connections that bypass Envoy.
3. **Sandbox egress floor.** Permit the minimum baseline egress set (LiteLLM
   gateway, Envoy, audit endpoint, CoreDNS, memory backend, OTel collector,
   event broker) and make the floor explicitly extensible.

## Tests

- Assert default-deny: a sandbox connection to an off-floor, non-added
  destination is denied.
- Assert external egress traverses Envoy; a direct bypass is denied.
- Assert each egress-floor destination is reachable under the baseline policy.
- Assert an explicitly-added egress destination is honored without weakening
  default-deny for the rest.

## References

D-08 (sandbox egress baseline). `plan-OPS1.md` (quota field). DECISIONS-LOG #6,
#7, #8 (out-of-scope baseline/cluster concerns).
