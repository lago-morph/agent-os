# ADR 0050: Security events on the event bus

- **Status**: Accepted
- **Date**: 2026-05-31

## Context

Security-relevant signals arise across the platform: the policy engine (OPA, A7)
denies an admission, the egress proxy blocks a destination, the gateway rejects a
malformed request, a reconciler observes a resource in an unexpected state. These
base signals originate in many subsystems, in many formats, and are collected and
handled in many different places. Today each subsystem handles its own security
signal locally — it denies, blocks, logs, alarms — but there is no single place a
dashboard, an audit consumer, or an external tool can subscribe to in order to see
security events across the whole platform.

ADR 0031 fixes the `platform.security.*` namespace as one of the ten committed
top-level CloudEvent namespaces on the bus. ADR 0045 then assigns exactly one owner
to each namespace, with no co-ownership: the policy engine (OPA, A7) owns
`platform.security`. That ownership rule says who authors the schema, but it does not
by itself say how a security signal detected in some *other* component reaches the
bus. Many components detect security-relevant events; only one owns the schema. This
ADR records how those two facts fit together.

## Decision

**Security alerts are treated like policy violations — "we caught something out of
the norm worth capturing" — and are published to the event bus under
`platform.security`, whose schema is owned by the policy engine (OPA, A7).**

- The `platform.security` event schema has a **single owner: the policy engine (OPA,
  A7)** (per ADR 0045). A7 authors the event-type schemas and registers them in the
  schema registry (B12). No other component defines its own event types in this
  namespace.
- **Cross-cutting requirement on every component:** when a component detects a
  security-relevant event, it performs its existing local handling **and
  additionally emits the event to the bus** under `platform.security`, using the
  schema owned by A7. The local handling is unchanged; the bus emission is added on
  top of it. This requirement must land in each component's spec, and each such
  component takes an explicit dependency on A7's schema.
- The bus emission is an **export/notification surface, not the primary response
  path.** A component does not wait on the bus to decide what to do — it still denies,
  blocks, or alarms locally. The bus carries the event so that interested parties can
  observe it.
- Consumers subscribe **via the bus**: dashboards, the audit pipeline, and external
  SIEM tooling. The bus is the single publish point, so an emitter publishes once and
  consumers choose what to consume, rather than each emitter wiring point-to-point to
  each consumer.
- **External SIEM integration is out of scope** for this ADR and for v1; it is named
  only as an example of a bus consumer that the single publish point makes possible.

## Consequences

- There is one place to subscribe for security events across the platform. Base
  signals come from many subsystems in many formats and are collected in many places;
  the bus normalizes the *publish* point so consumers do not have to integrate with
  each subsystem individually.
- Every component that can detect a security-relevant event acquires a new
  obligation — emit to `platform.security` in addition to handling locally — that
  must be written into its component spec and traced to A7's schema dependency.
- Because emission is additive and not on the primary response path, adding or
  changing bus consumers (a new dashboard panel, audit category, or future SIEM
  export) does not change how any component responds to the underlying signal.
- A7 carries the schema for events it does not itself originate. Schema changes are
  coordinated with A7 as the single owner, and the set of emitters (every component
  that detects security events) and consumers (dashboards, audit, external SIEM) is
  the affected set.

## References

- [ADR 0045](./0045-single-owner-per-cloudevent-namespace.md) (single owner per CloudEvent namespace — the ownership rule this ADR instantiates for `platform.security`)
- [ADR 0031](./0031-cloudevent-top-level-taxonomy.md) (CloudEvent top-level taxonomy — the `platform.security` namespace)
- A7 / OPA (policy engine, owner of the `platform.security` schema) and B16 (policy/admission)
- `_meta/reviews/DECISIONS-LOG.md` (event-namespace ownership QN-03 — the cross-cutting security requirement this ADR records)
