# SPEC ADR-0023 — Knative broker is in the architecture; sources are environment-specific by design [PROPOSED]

> kind: ADR · workstream: — · tier: T1
> upstream: [A4] · downstream: [B8, B12, A19, A14] · adrs: [0023] · views: [6.7]
> canon-glossary: cf2d1a754a58 · canon-interface: 45ee7b798c47

## 1. Purpose & Problem Statement

ADR 0023 is a settled decision: the Knative broker, Triggers, channels, the CloudEvent taxonomy (ADR 0031), and the two initial adapter flows (AlertManager → HolmesGPT, budget-exceeded → email user) are part of the platform architecture and travel with every install unchanged. Event *sources* are explicitly outside the architectural commitment — each install picks the sources appropriate to its environment (cloud-native on EKS/AKS, `PingSource`/`ApiServerSource` everywhere, webhook receivers or synthetic generators on kind). This SPEC states what honoring that boundary obliges of the eventing components: the broker contract is uniform and portable, sources live in environment-specific overlays, and components that depend on a cloud source must document a kind-equivalent path or be marked cloud-only. It does not re-argue the broker choice (ADR 0004) or the source-portability rationale.

The problem the decision solves: standardizing one source set across all environments would force cloud SDKs into dev clusters and webhook shims into prod, defeating CNI/cloud-agnosticism and conflicting with the independent-cluster topology (ADR 0026). Drawing the boundary at "broker in, sources out" keeps Triggers, adapters (B8), sinks, and the schema registry (B12) testable once and runnable everywhere.

## 2. Scope

### 2.1 In scope
- The obligation that broker, Triggers, channels, CloudEvent taxonomy, and the two initial adapter flows ship identically in every install.
- The obligation that event sources live in environment-specific overlays, not platform manifests, and are an install-team responsibility.
- The obligation that platform components emit CloudEvents directly to the broker and never depend on how external events arrive.
- The obligation that any component relying on a cloud-specific source documents a kind-equivalent path (webhook receiver or synthetic generator) or is marked cloud-only.
- The obligation that install documentation calls out per-environment source choices explicitly.

### 2.2 Out of scope (and where it lives instead)
- Knative Eventing + NATS JetStream broker install — component **A4** SPEC.
- Knative event adapter services (CloudEvent → `AgentRun`/Argo Workflow) — component **B8** SPEC.
- CloudEvent schema registry — component **B12** SPEC; per-event-type names/schemas deferred (ADR 0031, backlog §4).
- CloudEvent top-level taxonomy itself — **ADR 0031**.
- The canonical kind-side source pattern ("webhook receivers for everything, synthetic generators, or both") — deliberately left **open** (backlog §5); not fixed here.
- Substrate abstraction — **ADR 0041**; ADR 0023's source boundary is a documented exception to that pattern.

## 3. Context & Dependencies

Upstream consumed: **A4** (Knative Eventing + NATS JetStream broker) provides the broker, Triggers, channels the decision declares portable.
Downstream consumers: **B8** (event adapters) and **B12** (schema registry) are portable across kind/EKS/AKS by virtue of this boundary; **A19** (Mattermost adapter) and **A14** (HolmesGPT, target of the AlertManager flow) consume broker events without source coupling.

ADR decisions honored:
- **ADR 0023** (this) — broker portable; sources environment-specific; cloud-source consumers document kind-equivalent or are cloud-only.
- **ADR 0004** — NATS JetStream is the broker backend the portable contract sits on.
- **ADR 0031** — the closed CloudEvent taxonomy is the load-bearing contract making the broker uniform across installs.
- **ADR 0026** — each cluster is an independent install owning its own eventing; sources are per-cluster.
- **ADR 0033** — AWS source set is shipped/exercised in v1.0; Azure documented-as-supported; kind kept viable without mimicking a cloud.
- **ADR 0041** — the source shape difference is a documented exception to substrate abstraction.

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs
N/A — ADR 0023 introduces no CRD/XRD. It governs the placement of Knative `Trigger`/source resources (Knative-native kinds, not platform CRDs) across platform manifests vs environment overlays. Source kinds named in the ADR (`AwsSqsSource`, `PingSource`, `ApiServerSource`) are upstream Knative kinds, not Canon platform CRDs.

### 4.2 APIs / SDK surfaces
- HTTP APIs of the Knative adapter services (B8) version per URL-path versioning (`/v1/...`, ADR 0030). Method surfaces beyond that are B8 design-time; `[PROPOSED — not in source]` if detailed.

### 4.3 CloudEvents emitted / consumed
- The broker carries the closed set of ten top-level namespaces (ADR 0031): `platform.lifecycle.*`, `platform.audit.*`, `platform.gateway.*`, `platform.policy.*`, `platform.capability.*`, `platform.evaluation.*`, `platform.approval.*`, `platform.observability.*`, `platform.tenant.*`, `platform.security.*`. These are uniform across every install.
- The two initial v1.0 trigger flows (AlertManager → HolmesGPT; budget-exceeded → email user) are exercised in every environment because their emitters run in-cluster, not via a cloud source.
- Per-event-type names within each namespace are deferred to B12's registry; `[PROPOSED — not in source]` any concrete event-type name.

### 4.4 Data schemas / connection-secret contracts
N/A — no connection secret introduced. Cloud sources (e.g. an SQS-fed flow) configure their own credentials in environment overlays; this is outside the uniform connection-secret contract (ADR 0041) by design.

## 5. OSS-vs-Custom Decision
N/A — ADR. (Enforcement note: the ADR names upstream **Knative** Eventing, **NATS JetStream**, and upstream source kinds; the custom pieces are the **B8** adapters and **B12** registry, which are unchanged across environments. No fork.)

## 6. Functional Requirements
- REQ-ADR-0023-01: The broker, Triggers, channels, and CloudEvent taxonomy MUST ship in the platform manifests as one shape across all installs (kind/EKS/AKS), with no per-environment variation.
- REQ-ADR-0023-02: Event sources MUST live in environment-specific overlays maintained by the install team, NOT in the platform manifests.
- REQ-ADR-0023-03: Platform components MUST emit CloudEvents directly to the broker and MUST NOT depend on how external events arrive at the broker.
- REQ-ADR-0023-04: Triggers, adapters (B8), sinks, and the schema registry (B12) MUST be byte-identical across kind/EKS/AKS — tests written against the broker contract MUST run unchanged in every environment.
- REQ-ADR-0023-05: Any component whose flow relies on a cloud-specific source MUST document a kind-equivalent path (webhook receiver or synthetic generator) as part of its own deliverables, OR be explicitly marked cloud-only.
- REQ-ADR-0023-06: Install documentation MUST call out the per-environment source choices explicitly (AWS source set on EKS, Azure set on AKS, webhook/synthetic on kind).
- REQ-ADR-0023-07: The AlertManager → HolmesGPT flow MUST work identically in every environment by emitting from an in-cluster platform component, not via a cloud-specific source.

## 7. Non-Functional Requirements
- Portability: the broker contract is the portability boundary; no source-specific logic may leak into Triggers/adapters/registry.
- Multi-tenancy (§6.9): events crossing the broker carry identity/tenant context consistent with the taxonomy; source placement does not alter tenancy enforcement.
- Observability (§6.5): cloud-source-specific failure modes (SQS visibility timeouts, source-side IAM, dead-letter) are only observable in EKS/AKS integration tests — raising the bar on "tested before merge".
- Versioning (ADR 0030): adapter HTTP APIs URL-path-versioned; CloudEvent schemas versioned per ADR 0031.

## 8. Cross-Cutting Deliverable Checklist
N/A — ADR (verification map lives in the PLAN). The §14.1 standard set applies to the enforcing components (A4, B8, B12), not to this decision record.

## 9. Acceptance Criteria
- AC-ADR-0023-01: Honored when the platform manifests contain the broker/Triggers/channels/taxonomy with zero per-environment forks, and no source manifests are present in them. (REQ-01/02)
- AC-ADR-0023-02: Honored when source resources are found only in environment-specific overlays for each of kind/EKS/AKS. (REQ-02)
- AC-ADR-0023-03: Honored when a platform component emits to the broker with no reference to any source type, and a downstream Trigger fires identically regardless of source. (REQ-03)
- AC-ADR-0023-04: Honored when the same B8/B12 test suite passes unchanged on kind, EKS, and AKS broker contracts. (REQ-04)
- AC-ADR-0023-05: Honored when each cloud-source-dependent component's deliverables either include a documented kind-equivalent path or carry a cloud-only marker. (REQ-05)
- AC-ADR-0023-06: Honored when install docs enumerate the per-environment source set for kind/EKS/AKS. (REQ-06)
- AC-ADR-0023-07: Honored when the AlertManager → HolmesGPT flow succeeds on kind with no cloud source present. (REQ-07)

## 10. Risks & Open Questions
- OQ-1 (med): The canonical kind-side source pattern is deliberately open (backlog §5) until enough component flows land; ACs that assume a single kind pattern are partial until then. `[PROPOSED]`
- R-1 (med): Dev environments cannot exercise cloud-source failure modes; the only safety net is EKS/AKS integration tests gating merge — a process control, not an architectural one.
- R-2 (low): Source manifests drifting into platform manifests would silently break portability; mitigated by AC-01's "no sources in platform manifests" check.

## 11. References
- ADR 0023 (`adr/0023-environment-specific-knative-sources.md`) — the decision enforced here.
- architecture-overview.md §6.7 (eventing architecture).
- architecture-backlog.md §5 (open question: kind-side source pattern).
- Enforcing/related components: A4 (broker), B8 (adapters), B12 (schema registry), A19 (Mattermost adapter), A14 (HolmesGPT).
- ADR 0004 (broker backend), 0026 (independent cluster), 0031 (taxonomy), 0033 (AWS target), 0041 (substrate exception).
