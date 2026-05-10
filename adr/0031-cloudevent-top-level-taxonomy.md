# ADR 0031: CloudEvent top-level type taxonomy

- **Status**: Accepted
- **Date**: 2026-05-10

## Context

All platform events — agent and resource lifecycle, audit emissions, gateway events, policy decisions, evaluation runs, approvals — flow through Knative Eventing as CloudEvents on the NATS JetStream broker (ADR 0004). Knative Triggers route events declaratively by `type`, so a stable, agreed-upon top-level taxonomy is what makes filtering, fan-out, side-effect wiring, and external subscription tractable. Without a fixed namespace set, each component would invent its own naming and Triggers would have to match on ad-hoc patterns, breaking predictable consumption and audit-category alignment with the durable audit pipeline (ADR 0034) and its downstream subscribers. The architecture therefore commits to a closed set of top-level type namespaces; specific event names within each namespace are design-time per component.

## Decision

The platform commits to the top-level CloudEvent type namespaces enumerated in architecture-overview.md § 6.7 as an architectural invariant: `platform.lifecycle.*`, `platform.audit.*`, `platform.gateway.*`, `platform.policy.*`, `platform.capability.*`, `platform.evaluation.*`, `platform.approval.*`, `platform.observability.*`, `platform.tenant.*`, and `platform.security.*`. Every CloudEvent emitted by a platform component MUST fall under exactly one of these namespaces. Per-event-type schemas under each namespace are design-time per component and are **deferred** per architecture-backlog.md § 4 ("Per-event-type CloudEvent schemas under each top-level namespace"); they live in component B12's schema registry as they are authored.

## Consequences

- Architectural invariant (architecture-backlog.md § 6): all platform CloudEvents fall under one of the ten committed top-level namespaces. A component that needs a category not on this list MUST request an architecture amendment and a new ADR rather than minting an ad-hoc top-level type.
- Per-event-type schema design under each namespace is deferred to component design time (architecture-backlog.md § 4); B12's registry is the single source of truth for the concrete schemas as they land.
- Every CloudEvent carries CloudEvents-native `specversion` plus a per-event-type `schemaVersion` field (ADR 0030, architecture-overview.md § 6.13); backward-compatible additions bump minor and breaking changes mint a new event type rather than breaking subscribers.
- Knative Trigger filtering becomes straightforward: subscribers match on the namespace prefix (e.g., `platform.audit.*` for an audit sink, `platform.approval.*` for the approval-decision dispatcher) without coupling to component-internal event names.
- The taxonomy aligns with consumption boundaries already present in the platform: `platform.audit.*` is the namespace consumed by the platform audit adapter into its Postgres + S3 system of record (ADR 0034), with OpenSearch as an advisory fanout for query and dashboards rebuildable from those primaries; `platform.security.*` is intentionally distinct from audit; and `platform.observability.*` covers threshold-crossing and alert-routing flows such as the v1.0 budget-exceeded notification.
- The two initial v1.0 trigger flows (AlertManager → HolmesGPT, budget-exceeded → email user) and every subsequent component's flow design (architecture-overview.md § 6.7) author their event types under these namespaces.
- External subscribers can rely on the namespace set as a stable contract for declarative subscription; introducing a new top-level namespace is a breaking change to that contract.

## References

- [architecture-overview.md § 6.7](../architecture-overview.md#67-eventing-architecture-knative--nats-jetstream) (Eventing architecture — top-level type namespaces)
- [architecture-backlog.md](../architecture-backlog.md) [§ 4](../architecture-backlog.md#4-topics-that-need-further-design-before-implementation) (deferred per-event-type schemas), [§ 6](../architecture-backlog.md#6-architecture-level-invariants-worth-documenting-as-adrs) (invariant)
- [ADR 0004](./0004-nats-jetstream-broker.md) (NATS JetStream broker), [ADR 0030](./0030-crd-and-api-versioning-policy.md) (CRD/API versioning), [ADR 0034](./0034-audit-pipeline-durable-adapter.md) (audit pipeline — Postgres + S3 system of record, OpenSearch advisory fanout)
