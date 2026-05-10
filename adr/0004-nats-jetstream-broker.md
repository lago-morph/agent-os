# ADR 0004: NATS JetStream as the Knative broker backend

## Status
Accepted

## Context
Knative Eventing is the platform's event mesh (see ADR 0023 and architecture-overview §6.7). It standardizes on CloudEvents and a Source / Broker / Trigger model: sources emit CloudEvents into a Broker, and Triggers declaratively filter and route them to sinks (the two thin Python adapters that produce `AgentRun` CRs or Argo Workflows, plus external subscribers).

Knative Eventing requires a concrete broker implementation. The candidates considered were Kafka, NATS JetStream, RabbitMQ, and AWS SNS+SQS. Constraints driving the choice:

- The broker must run in **every environment** the platform installs into — including local kind clusters used for development and conformance — without operator-heavy infrastructure or cloud lock-in. Each cluster is an independent install (ADR 0026); there is no shared cross-cluster bus.
- The broker must implement the **Knative Channel / Broker contract natively**, so Triggers, filters, and dead-letter sinks behave as documented.
- Durability is required so that a CloudEvent emitted by a platform component (LiteLLM callback, audit emitter, AlertManager bridge) is not lost if a Trigger sink is briefly unavailable.
- The broker is plumbing, not a product surface. We want minimal operational footprint and no separate cluster of stateful nodes to babysit.

## Decision
Use **NATS JetStream** as the Knative Eventing broker backend, in **both dev and prod**, on every cluster the platform installs into. JetStream provides the durable streams that back the Knative Broker; Knative Triggers attach to the Broker and perform CloudEvent attribute filtering before dispatch to sinks.

The broker is environment-uniform. What differs per environment is the set of **event sources** feeding it (`PingSource`, `ApiServerSource`, `AwsSqsSource` on AWS, an Azure equivalent, webhook receivers in kind). That source-vs-broker split is recorded in ADR 0023.

## Consequences
- One broker technology to learn, install, monitor, and back up across every cluster. Helm-installed alongside Knative Eventing; stream configuration is declarative.
- Local development on kind has the same Broker and Trigger semantics as production. Trigger flow design — required of every component (architecture-backlog §6 invariants) — can be exercised on a laptop.
- JetStream's durability gives at-least-once delivery to Triggers; sinks (the adapters, ARK, Argo Workflows, external subscribers) must remain idempotent. The CloudEvent → `AgentRun` and CloudEvent → Argo Workflow adapters are pure field-mapping and naturally idempotent on event id.
- We do not get Kafka's ecosystem (Connect, Streams, Schema Registry). We accept that: the platform's schema registry is a JSON schema set in Git (architecture-overview §6.7), not Confluent-style infrastructure.
- Operational scope stays small: NATS clusters are lightweight compared to Kafka or RabbitMQ, and there is no managed-service dependency as with SNS+SQS.

## Alternatives considered
- **Kafka.** Heaviest of the options. Strong ecosystem and replay story, but operationally expensive in every cluster including kind, and the platform does not need stream-processing semantics — Knative Triggers cover the routing model.
- **RabbitMQ.** Mature Knative integration, but adds a second stateful component class with its own clustering model for no capability we need beyond what JetStream provides.
- **AWS SNS+SQS.** Rejected on fit, not just on cloud lock-in: it does not match Knative's Broker / Trigger pub-sub semantics (Triggers expect a Broker they can declaratively filter), and it cannot run in dev or on-prem clusters. SNS+SQS remains relevant only as an event **source** on AWS via `AwsSqsSource` (ADR 0023).
- **Argo Events** as the whole eventing layer (replacing Knative Eventing). Rejected at the layer above this ADR (architecture-backlog §2.7) in favor of Knative Eventing's CloudEvents-native source / broker / trigger model.

## Related
- ADR 0023 — Knative event sources are environment-specific while the broker is uniform.
- ADR 0026 — Independent-cluster install topology; each cluster runs its own NATS JetStream.
- architecture-overview §5 (component table), §6.7 (Eventing architecture).
- architecture-backlog §2.7, §2.8, §6 (invariants), §7 (ADR candidates).
