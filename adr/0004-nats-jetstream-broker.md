# ADR 0004: NATS JetStream as the Knative broker backend

- **Status**: Accepted
- **Date**: 2026-05-10

## Context

Knative Eventing was chosen as the platform's event mesh (architecture-backlog.md § 2.7), and Knative requires a broker backend for CloudEvent transport, durability, and trigger dispatch (architecture-overview.md §6.7). The platform installs into kind for dev and into managed Kubernetes (initially AWS) for prod, and we want the same broker in both environments so that trigger flows, ordering semantics, and operational runbooks are identical end to end. Event sources differ per environment (AwsSqsSource on AWS, webhook receivers in kind), but everything downstream of the source — broker, triggers, adapters — must be uniform. Four backends were evaluated (architecture-backlog.md § 2.8).

## Decision

The platform adopts **NATS JetStream** as the Knative Eventing broker backend (component **A4**, architecture-overview.md §5, §6.7, §14.1). The same NATS JetStream install runs in dev (kind) and prod (managed Kubernetes), provisioned by Helm with stream configuration managed declaratively. All CloudEvents emitted under the `platform.*` namespaces (§6.7) flow through this broker and are dispatched by Knative Triggers to adapters and external subscribers.

## Alternatives considered

- **Kafka** — Rejected: heaviest operational footprint of the four, overkill for the event volumes the platform targets, and awkward to run in a kind dev cluster with the same topology as prod.
- **RabbitMQ** — Rejected: viable but heavier than NATS, and the Knative RabbitMQ channel ecosystem is less actively maintained than the NATS JetStream channel.
- **SNS + SQS** — Rejected: doesn't fit Knative's pub/sub semantics (no native Knative channel, awkward trigger fan-out), AWS-only so it would force a different backend in dev, and would split runbooks between environments.

## Consequences

- A single NATS JetStream install (component A4) is the broker backend in dev and prod, removing an entire class of "works in dev, breaks in prod" eventing bugs.
- JetStream provides at-least-once delivery and durability for CloudEvents in flight; consumers (Knative Triggers and their sinks) must remain idempotent, which is already a CloudEvent-handling requirement.
- NATS runs anywhere (kind, EKS, Azure equivalents) with a small footprint, keeping the dev cluster lightweight and the prod cluster install tractable for the independent-cluster model (ADR 0026).
- Knative Trigger filtering and CloudEvent taxonomy (ADR 0031) are unaffected by the backend choice — broker selection is encapsulated below the Trigger layer.
- Operating NATS JetStream (stream sizing, retention, monitoring, upgrade) is a Workstream A responsibility; the Knative broker is observed via the standard observability stack (architecture-overview.md §6.5, broker arrival and dispatch metrics).
- Environment-specific event sources (ADR 0023) feed this single broker, preserving uniform downstream behavior across environments; the Mattermost integration (ADR 0036) is the first non-trivial cross-broker consumer to validate this.
- If event volumes ever outgrow NATS JetStream, the Knative abstraction lets us swap the backend without rewriting triggers or adapters; this is captured as a future evolution path rather than a current concern.

## References

- architecture-overview.md §6.7, §5, §14.1
- architecture-backlog.md § 2.7, 2.8
- ADR 0023 (environment-specific Knative sources), ADR 0031 (CloudEvent taxonomy), ADR 0036 (Mattermost integration via Knative Eventing)
