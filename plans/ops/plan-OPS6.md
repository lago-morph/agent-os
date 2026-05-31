# Implementation Plan: OPS6 — RBAC Granularity (Trimmed)

> Status: Draft (trimmed scope)
> Workstream: OPS (Operations & Platform Engineering)
> Component ID: OPS6

---

## Scope

Only the RBAC-granularity slice survives (decision #23): distinct RBAC
permissions for budget actions (agent-type default / per-instance override /
budget-maintainer) and policy actions (policy-change vs policy-maintain), with
field-level rules native RBAC cannot express enforced via Gatekeeper (OPA)
admission.

Policy bundle ownership / signing / staged rollout (#24 / #25 / #26) are already
decided and implemented in B3 / B16 — cross-referenced, not implemented here.
Anti-lockout / break-glass (#27) is post-MVP (future version) — not in scope.

## Work Items

1. **Budget RBAC roles.** Define distinct, independently-grantable permissions:
   set agent-type default budget, per-instance budget override, budget-maintainer.
2. **Policy RBAC roles.** Define distinct permissions: policy-change vs
   policy-maintain.
3. **Gatekeeper field-level rules.** For distinctions native RBAC cannot express,
   author Gatekeeper (OPA) admission rules (engine already deployed).

## Tests

- Assert each budget permission (default / override / maintainer) is grantable
  independently of the others.
- Assert policy-change and policy-maintain are grantable independently.
- Assert a field-level distinction native RBAC cannot express is enforced at
  Gatekeeper admission (a native-RBAC-allowed but field-policy-forbidden request
  is rejected).

## References

DECISIONS-LOG #23 (survived). #24 / #25 / #26 (decided in B3 / B16). #27
(anti-lockout / break-glass — future version, not in scope). B3 / B16 (policy
bundle).
