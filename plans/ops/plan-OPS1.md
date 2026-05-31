# Implementation Plan: OPS1 — Resource Quotas (Trimmed)

> Status: Draft (trimmed scope)
> Workstream: OPS (Operations & Platform Engineering)
> Component ID: OPS1

---

## Scope

Only the resource-quota slice survives (decisions #5, #9). Autoscaling (#10,
OOS), cost showback (#11, future), and SLI promotion (#12, OOS) are not
implemented.

## Work Items

1. **TenantOnboarding quota fields.** Add the extensible, typed `quotas` list
   with a `quotaType` discriminator. Enforce mandatory presence of `cpu` and
   `memory` entries, each requiring exactly one of `limit` or `unlimited: true`
   (admission validation).
2. **AgentEnvironment composition.** Emit a `ResourceQuota` and a `LimitRange`
   in the tenant namespace, derived from the declared `quotas` list.

## Tests

- Reject `TenantOnboarding` missing a `cpu` or `memory` quota entry.
- Reject a quota entry with neither / both of `limit` and `unlimited`.
- Assert the `AgentEnvironment` composition emits exactly one `ResourceQuota`
  and one `LimitRange`, with values matching the declared quotas.
- Assert an added `quotaType` value does not break existing-entry validation.

## References

DECISIONS-LOG #5, #9 (survived); #10, #11, #12 (not in scope).
