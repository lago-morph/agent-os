# ADR 0045: Single owner per CloudEvent namespace

- **Status**: Accepted
- **Date**: 2026-05-31

## Context

ADR 0031 establishes the top-level CloudEvent taxonomy: ten `platform.*`
namespaces (lifecycle, gateway, policy, audit, tenant, capability, approval,
security, observability, evaluation) that carry every event on the bus. The
taxonomy fixes the namespaces but not who is responsible for the event-type
schemas within each one.

Without an explicit rule, more than one component can plausibly claim a
namespace — several components emit `platform.audit` events, both the policy
engine and the dashboards touch `platform.policy`, and many components produce
`platform.security` events. Shared ownership of a schema leads to divergent or
conflicting event-type definitions, ambiguity about who registers a type in the
schema registry (B12), and no clear party to consult when a contract changes.

## Decision

**Exactly one component owns each of the ten `platform.*` CloudEvent
namespaces.** The owner authors and owns the event-type schemas for that
namespace and registers them in the schema registry (B12). Any other component
that emits or consumes those events takes an **explicit dependency** on the
owner. There is **no co-ownership** of a namespace.

The owners are:

| Event family (namespace) | Single owner | Who depends |
|---|---|---|
| Agent lifecycle (`platform.lifecycle`) | the agent operator / ARK (A5) | the sandbox (A6), dashboards |
| Gateway (`platform.gateway`) | the LiteLLM gateway (A1) | dashboards, cost reporting |
| Policy (`platform.policy`) | the policy engine / OPA (A7) | dashboards, audit |
| Audit (`platform.audit`) | the audit endpoint (A18) | every audit-emitting component |
| Tenant (`platform.tenant`) | the tenant-onboarding reconciler (A21) | dashboards |
| Capability (`platform.capability`) | the kopf operator (B13) | dashboard, SDK, profile library |
| Approval (`platform.approval`) | the approval system (B19) | promotion pipeline, dashboard, self-healing agent |
| Security (`platform.security`) | the policy engine / OPA (A7) | dashboards, audit, external SIEM (out of scope) |
| Observability (`platform.observability`) | the metrics & tracing stack — Tempo + Mimir (A13) | external consumers (initially cloud-native logging) |
| Evaluation (`platform.evaluation`) | the test & evaluation framework (B14) | dashboards, promotion pipeline |

A component that needs to emit into a namespace it does not own — for example,
any component emitting `platform.audit` events, or any component emitting a
`platform.security` event when it detects something out of the norm — uses the
schema defined by the owner and depends on it rather than defining its own
event types in that namespace.

## Consequences

- Each namespace has a single authoritative source for its schemas, so
  event-type definitions cannot diverge across emitters.
- Schema-registry (B12) registration has an unambiguous responsible party per
  namespace.
- Emitters and consumers that are not owners carry an explicit, documented
  dependency on the owner; this dependency must be stated in each such
  component's spec.
- A schema change has one owner to coordinate with, and the set of dependents
  in the table above tells that owner who is affected.
- OPA (A7) owns two namespaces — `platform.policy` and `platform.security` —
  which is intentional: both are policy-domain concerns authored by the same
  component.

## References

- [ADR 0031](./0031-cloudevent-top-level-taxonomy.md) (CloudEvent top-level taxonomy — the ten `platform.*` namespaces this ADR assigns owners to)
- B12 (schema registry — where each owner registers its event-type schemas)
- [ADR 0050](./0050-security-events-on-the-event-bus.md) (security events on the event bus — builds on A7's ownership of `platform.security`)
- [ADR 0051](./0051-observability-alerts-via-event-bus.md) (observability alerts via the event bus — builds on A13's ownership of `platform.observability`)
- `_meta/reviews/DECISIONS-LOG.md` (QN-03 — the resolved decision this ADR records)
