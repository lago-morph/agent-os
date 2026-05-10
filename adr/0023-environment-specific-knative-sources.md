# ADR 0023: Knative broker is in the architecture; sources are environment-specific by design

- **Status**: Accepted
- **Date**: 2026-05-10

## Context

The platform commits to Knative Eventing as the event mesh and NATS JetStream as the broker backend (ADR 0004), with filtering at Triggers and thin Python adapters (component B8) that translate CloudEvents into `AgentRun` CRs or Argo Workflows (architecture-overview.md §6.7).

The broker, Triggers, channels, adapters, and the CloudEvent type taxonomy (ADR 0031) are the same in every install — they are part of the architecture and travel with the platform regardless of where it is deployed.

The *sources* that feed events into the broker, however, are not the same across environments. An EKS install has `AwsSqsSource` available; an AKS install has its Azure equivalent; a kind-based dev cluster has neither. Standardizing one set of sources across all environments would force cloud SDKs into dev clusters and webhook shims into prod, defeating the point of being CNI- and cloud-agnostic and conflicting with ADR 0026 (each cluster is an independent install).

The architecture-backlog calls out an explicit open question (§5): how dev environments handle the absence of cloud-specific event sources — webhook receivers for everything, synthetic event generators, or some mix. This ADR records the *boundary* (broker is in, sources are out) and leaves the dev-side pattern open per that question.

## Decision

The broker, Triggers, channels, CloudEvent taxonomy, and the two initial adapter flows (AlertManager → HolmesGPT, budget exceeded → email user) are part of the architecture and ship with every install.

Event *sources* are explicitly out of the architectural commitment: each install picks the sources appropriate to its environment — cloud-native sources on EKS / AKS, `PingSource` and `ApiServerSource` everywhere, webhook receivers or synthetic generators on kind. Platform components emit CloudEvents directly to the broker without caring how external events arrive; downstream Triggers and adapters are unchanged across environments because they only see CloudEvents on the broker.

## Consequences

- Triggers, adapters (B8), sinks, and the CloudEvent schema registry (B12) are portable across kind / EKS / AKS without modification — the broker contract is uniform and tests written against it run anywhere.
- Per-environment install responsibility: whoever installs the platform on EKS configures the AWS-specific sources; whoever installs on AKS configures the Azure-specific ones; whoever runs kind configures webhook receivers or synthetic generators. The architecture does not prescribe a single source set, and the install documentation must call out the per-environment source choices explicitly.
- The "AlertManager → HolmesGPT" flow works identically in every environment because AlertManager runs in-cluster and emits to the broker as a platform component, not via a cloud-specific source — this is the model new component flows should follow when possible.
- Components that rely on an external cloud source (e.g., an SQS-fed flow) must explicitly document a kind-equivalent path — webhook receiver or synthetic event generator — as part of their own deliverables, or be marked as cloud-only.
- Open question (architecture-backlog.md §5): the canonical kind-side pattern — "webhook receivers for everything, synthetic event generators, or both" — is deliberately left open until enough component flows have landed to show which pattern wins. This ADR records that the gap is intentional, not an omission.
- Cost: dev environments cannot exercise cloud-source-specific failure modes (SQS visibility timeouts, IAM auth on the source side, dead-letter behavior). Those characteristics surface only in EKS / AKS integration tests, which raises the bar on what counts as "tested before merge".
- Aligns with ADR 0033 (initial cloud target is AWS + GitHub): we ship and exercise the AWS source set in v1.0, document the Azure source set as supported by the architecture, and keep kind viable for development without forcing it to mimic a cloud.
- A new install on a previously-unsupported environment (a different cloud, a different on-prem stack) is a source-set exercise, not an architecture change — provided that environment can run Knative + NATS JetStream and emit CloudEvents from whatever sources it does have.
- GitOps stays clean: the broker, Triggers, adapters, and event-type schemas live in the platform manifests (one shape across installs), while the per-environment source manifests live in environment-specific overlays maintained by the install team.
- The CloudEvent taxonomy (ADR 0031) is the load-bearing contract that makes this split work; tightening it (new namespaces, schema versioning) is a platform-wide concern, while adding a new source is a local one.

## References

- architecture-overview.md § 6.7
- architecture-backlog.md § 5
- ADR 0004 (NATS JetStream broker), ADR 0031 (CloudEvent taxonomy), ADR 0033 (initial cloud target)
