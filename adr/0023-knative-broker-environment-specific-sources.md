# ADR 0023: Knative broker uniform; event sources are environment-specific by design

## Status
Accepted

## Context
The platform uses Knative Eventing as its event mesh, with NATS JetStream as the broker backend in every environment (ADR 0004, architecture-overview §6.7). Knative's model splits cleanly into three parts: **sources** that produce CloudEvents, a **Broker** that durably accepts them, and **Triggers** that declaratively filter and dispatch to sinks (the two thin Python adapter services — CloudEvent → `AgentRun` CR and CloudEvent → Argo Workflow — plus external subscribers).

The platform installs into very different environments: developer kind clusters, on-prem clusters, and cloud-managed clusters (AWS, Azure, etc.). Each has a different set of "things that emit events worth reacting to." S3 object drops exist on AWS but not on kind. AlertManager exists everywhere but emits via webhook. CI lives outside Knative entirely (ADR 0010) and reaches in via webhook receivers.

A natural temptation is to make the eventing layer "look the same" by either swapping brokers per environment or rewriting Trigger schemas per environment. Both options were rejected, but the rejection needs to be recorded explicitly so component owners do not relitigate it while designing their per-component trigger flows.

## Decision
- **The broker is uniform.** NATS JetStream is the Knative broker backend in dev, on-prem, and every cloud (per ADR 0004). Same Helm chart, same Broker resource shape, same durability semantics, same operational runbook everywhere.
- **CloudEvent schemas are uniform.** The JSON schema registry in Git (architecture-overview §6.7) defines event types once. A `com.example.platform.budget.exceeded` event has the same shape on every cluster.
- **Trigger filters are uniform.** Triggers filter by CloudEvent attributes (`type`, `source`, extensions) declaratively. A Trigger written for prod works in dev and vice versa, because it filters on the event, not on its origin.
- **Event sources are environment-specific by design.** The set of `Source` resources installed into a cluster reflects what that cluster can actually observe:
  - **Cloud (AWS):** `AwsSqsSource`, `AwsS3Source`, native cloud-event integrations.
  - **Cloud (Azure / GCP):** equivalent native sources.
  - **On-prem / cluster-local:** `ApiServerSource`, `PingSource` (cron), AlertManager webhook receivers.
  - **kind / dev:** webhook receivers and synthetic event generators that produce the same CloudEvent shapes the cloud sources would produce, so downstream Triggers and adapters are exercised identically.
- **Each component owns its trigger flow design.** Per the standard component deliverables (architecture-overview §6.7, architecture-backlog §6, §7), every component declares "what CloudEvents I emit, what events I consume, what Triggers route between them." That design is environment-agnostic because it operates on event types, not on sources.

## Consequences
- Trigger flow design and adapter logic are written once and run unchanged in every environment. Chainsaw tests that assert "this CRD applied → these CloudEvents on the broker" run identically on a laptop and in prod.
- The dev-environment story for cloud-only sources is explicitly carried as an open backlog item (architecture-backlog §5): we will provide webhook receivers and synthetic event generators that emit the same CloudEvent shapes as the cloud sources they stand in for. The exact set is decided as components land and surface their needs, not up front.
- CI (ADR 0010, GitHub Actions outside the cluster) bridges into the eventing layer through webhook receivers when it needs to inject events — same mechanism as any other external producer.
- Component owners are explicitly responsible for the "what events does this emit and consume" Knative trigger flow design as part of their standard deliverables. The eventing fabric grows organically rather than via big-bang central design.
- Source resources become a per-environment GitOps overlay rather than part of any component's core manifest.

## Alternatives considered
- **Different broker per environment** (e.g. an in-memory channel for kind, NATS for prod). Rejected: this was already decided in ADR 0004, and divergent broker semantics defeat the point of being able to test trigger flows on a laptop.
- **Per-environment Trigger schemas** (rewriting filters or event types per cluster to match what local sources happen to emit). Rejected: it pushes environment knowledge into every component, breaks the schema registry, and makes trigger flow design a per-environment exercise instead of a per-component one.
- **Force every event through a single normalization layer before the broker** (a "universal source adapter"). Rejected as premature: Knative's source ecosystem already normalizes to CloudEvents, and a custom funnel would duplicate that work while adding a single point of failure.

## Related
- ADR 0004 — NATS JetStream as the Knative broker backend (uniform across environments).
- ADR 0026 — Independent-cluster install topology; sources are configured per cluster.
- ADR 0010 — GitHub Actions CI lives outside Knative; webhooks may bridge in.
- architecture-overview §6.7 (Eventing architecture).
- architecture-backlog §5 (dev environment open questions), §7 (ADR candidates).
