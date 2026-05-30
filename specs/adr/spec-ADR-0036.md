# SPEC ADR-0036 — Two-way Mattermost integration via Knative Eventing as the v1.0 chat-platform pattern `[PROPOSED]`

> kind: ADR · workstream: — · tier: T2
> upstream: [A4; A1; A7] · downstream: [A19; A14; B12] · adrs: [0036] · views: [6.7]
> canon-glossary: FROZEN · canon-interface: FROZEN

## 1. Purpose & Problem Statement

ADR 0036 is a settled decision: **Mattermost Team Edition (OSS)** is the v1.0 platform-operator chat
surface, integrated **two-way through the Knative Eventing fabric** — outbound via a Trigger-filtered
subscription to a thin Mattermost adapter, inbound via a webhook source that normalizes into
CloudEvents on the broker. Routing is expressed as **OPA policy**; the adapter is a **driver-shaped**
thin Python sink; SSO is via a **GitLab-spoof Keycloak client** isolated from the platform JWT. This
SPEC states what honoring the decision requires of the adapter, the webhook source, and Keycloak
config — it does not re-argue the chat-platform pick.

The problem the decision solves: the platform needs an operator-facing chat surface distinct from the
end-user LibreChat UI (ADR 0007) — for alerts, HolmesGPT findings, approval prompts, and chatops —
without point-to-point coupling and without leaking those flows into LibreChat's threat model.

## 2. Scope

### 2.1 In scope
- Mattermost Team Edition as the v1.0 operator chat surface, distinct from LibreChat (different audience/policies/identity scope).
- Outbound: a Knative Trigger filtering by CloudEvent `type` → thin Mattermost adapter calling the Mattermost REST API, with per-event-type→channel routing expressed as OPA policy.
- Inbound: a Mattermost outgoing-webhook → Knative webhook source normalizing payloads into CloudEvents under the existing taxonomy (ADR 0031), attaching the authenticated Keycloak identity.
- Driver-shaped adapter/source interface; Mattermost is the first driver (Slack/Teams deferred, addable without topology change).
- SSO via a dedicated GitLab-spoof Keycloak client, isolated from the ADR 0029 platform claim schema.

### 2.2 Out of scope (and where it lives instead)
- Per-event-type CloudEvent names/schemas the adapter filters on — owned by component B12.
- The broker/Trigger mechanics — owned by A4 (Knative + NATS).
- The CloudEvent→AgentRun adapter that dispatches inbound chatops to agents — owned by B8 / ARK (A5).
- The OPA policy library the routing rules live in — owned by B16 (content) over B3 (framework).
- The Mattermost adapter component build — owned by component A19.
- Slack/Teams drivers — deferred to future-enhancements.

## 3. Context & Dependencies

Upstream consumed: A4 (Knative Eventing + NATS JetStream broker — the fabric carrying both
directions); A1 (LiteLLM — the gateway through which chatops agent invocations route); A7
(OPA/Gatekeeper — evaluates the routing policy on the adapter). Downstream consumers: A19 (builds the
Mattermost adapter + webhook source), A14 (HolmesGPT — gains a chatops surface and a findings sink),
B12 (registers the event types the adapter filters on and the inbound source mints).

ADR decisions honored:
- **ADR 0036** (this): Mattermost OSS as v1.0 chat surface; two-way over Knative; OPA routing; driver-shaped; GitLab-spoof SSO isolated from the platform JWT.
- **ADR 0004**: NATS JetStream broker carries chat traffic; routing by CloudEvent `type`.
- **ADR 0031**: outbound filters on `platform.observability.*` / `platform.approval.requested` / selected `platform.security.*`; inbound mints under the existing taxonomy (likely `platform.lifecycle.*` for chatops).
- **ADR 0023**: the inbound webhook source is in the same family as environment-specific Knative sources.
- **ADR 0002 / ADR 0018**: per-event-type→channel routing is OPA policy, GitOps-managed and audited.
- **ADR 0007**: LibreChat stays the end-user agent UI; Mattermost is the operator surface — different audiences and policies.
- **ADR 0028 / ADR 0029**: the GitLab-spoof mapper lives only on the Mattermost-dedicated Keycloak client and does NOT enter the platform JWT consumed by LiteLLM/OPA/Headlamp/LibreChat.

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs
N/A — this ADR imposes no new CRD/XRD. It consumes Knative Trigger / source primitives (A4) and OPA
policy (A7); the Mattermost-dedicated Keycloak client is provisioned by the A-track Keycloak config.

### 4.2 APIs / SDK surfaces
- **Outbound adapter** — a thin Python Knative sink (matching every other Knative sink in §6.7). It
  has no decision logic of its own: pure field-mapping + an OPA call for routing, then a Mattermost
  REST API call. Written against a chat-platform-agnostic **driver interface** (Mattermost = first driver).
- **Inbound webhook source** — a Knative source receiving Mattermost outgoing-webhook posts,
  normalizing into a CloudEvent, attaching the authenticated Keycloak identity (resolved via the
  Mattermost-dedicated client) so downstream OPA sees a real platform principal, then publishing to the broker.
- Driver responsibilities are limited to API-shape translation + webhook payload normalization.
  Concrete adapter method signatures are **not specified in source** `[PROPOSED — not in source]`.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
- **Consumed (outbound)**: a Trigger filters by `type` — e.g. `platform.observability.alert.*`,
  `platform.approval.requested`, selected `platform.security.*`. (Concrete sub-names deferred to B12.)
- **Emitted (inbound)**: the webhook source mints a CloudEvent under the existing taxonomy — likely
  `platform.lifecycle.*` for chatops agent invocations — carrying `specversion` + `schemaVersion` (ADR 0030).

### 4.4 Data schemas / connection-secret contracts
- The Mattermost-dedicated **Keycloak client** is configured with custom protocol mappers emitting a
  GitLab-shaped OAuth + `/api/v4/user`-shaped userinfo response; Mattermost is pointed at it as if it
  were a GitLab instance. This mapping lives **only** on this client; it does not alter the ADR 0029
  platform claim schema. No platform connection-secret surface beyond the client credentials.

## 5. OSS-vs-Custom Decision
N/A — ADR. The realizing component A19 wraps Mattermost Team Edition (OSS, in-cluster) with a custom
thin Python adapter + webhook source; the GitLab-spoof Keycloak client is custom configuration on the
A-track Keycloak. Those OSS-vs-custom calls live in the A19 / A8 SPECs.

## 6. Functional Requirements

- REQ-ADR-0036-01: Mattermost Team Edition MUST be the v1.0 operator chat surface, deployed in-cluster, and MUST NOT replace LibreChat; the two run with different audiences, access policies, and identity scopes.
- REQ-ADR-0036-02: Outbound platform→Mattermost traffic MUST flow as a Knative Trigger filtering by CloudEvent `type` to a thin adapter calling the Mattermost REST API — no point-to-point component→Mattermost coupling.
- REQ-ADR-0036-03: Per-event-type→channel routing MUST be expressed as OPA policy evaluated on the adapter; the adapter MUST carry no routing decision logic of its own (field-mapping + OPA call only).
- REQ-ADR-0036-04: Inbound Mattermost→platform traffic MUST flow as a Mattermost outgoing-webhook → Knative source that normalizes into a CloudEvent under the existing taxonomy (ADR 0031) and publishes onto the broker.
- REQ-ADR-0036-05: The inbound source MUST attach the authenticated Keycloak identity of the posting user to the CloudEvent so downstream OPA sees a real platform principal, not an opaque Mattermost user id.
- REQ-ADR-0036-06: The outbound adapter and inbound source MUST be written against a chat-platform-agnostic driver interface; adding Slack/Teams later MUST NOT require touching the Trigger topology, the OPA routing rules' shape, or the CloudEvent taxonomy.
- REQ-ADR-0036-07: SSO MUST use a dedicated GitLab-spoof Keycloak client with custom protocol mappers; Mattermost MUST be configured for GitLab OAuth pointed at that client.
- REQ-ADR-0036-08: The GitLab-spoof mapping MUST live only on the Mattermost-dedicated client and MUST NOT leak into the platform JWT consumed by LiteLLM, OPA, Headlamp, or LibreChat (ADR 0029); a Mattermost-side compromise MUST NOT be able to forge a platform JWT.
- REQ-ADR-0036-09: All outbound chat traffic MUST traverse the broker + Trigger so it inherits the same audit emission, observability, and policy gating as every other event flow.

## 7. Non-Functional Requirements
- Security/isolation: the Mattermost-dedicated Keycloak client is outside the ADR 0029 claim schema by construction — an explicit isolation property; routing-as-OPA-policy is reviewable and GitOps-managed.
- Multi-tenancy (§6.9): inbound events carry a real platform principal, so downstream OPA/tenant scoping applies normally.
- Observability (§6.7): chat traffic stays inside the same audit/observability/policy envelope as every event flow.
- Portability: the Knative-Trigger / OPA-routing / driver-interface pattern survives a move to Mattermost Enterprise or a different chat platform.
- Versioning (ADR 0030): inbound events carry `specversion` + `schemaVersion`; the driver interface is the stable seam for new drivers.

## 8. Cross-Cutting Deliverable Checklist
N/A — ADR. The realizing component A19 carries the §14.1 deliverables (adapter + webhook source
manifests, runbook, audit emission, Knative trigger flow, HolmesGPT chatops toolset, 3-layer tests);
A8 carries the Keycloak-client config; conformance to this pattern is verified per §9.

## 9. Acceptance Criteria

- AC-ADR-0036-01: Honored when Mattermost and LibreChat are shown running with distinct audiences, access policies, and identity scopes, neither replacing the other. (→ REQ-01)
- AC-ADR-0036-02: Honored when a `platform.observability.alert.*` / `platform.approval.requested` event on the broker is delivered to a Mattermost channel via the Trigger→adapter→REST path with no direct component→Mattermost call. (→ REQ-02, REQ-09)
- AC-ADR-0036-03: Honored when changing the event-type→channel mapping is shown to be an OPA-policy (GitOps) change requiring no adapter redeploy, and the adapter carries no routing logic. (→ REQ-03)
- AC-ADR-0036-04: Honored when a Mattermost outgoing-webhook post becomes a normalized CloudEvent on the broker under the existing taxonomy. (→ REQ-04)
- AC-ADR-0036-05: Honored when an inbound CloudEvent carries the posting user's authenticated Keycloak identity and a downstream OPA decision is shown to act on that principal. (→ REQ-05)
- AC-ADR-0036-06: Honored when a second (stub) driver is registered against the driver interface without touching the Trigger topology, OPA routing shape, or taxonomy. (→ REQ-06)
- AC-ADR-0036-07: Honored when Mattermost authenticates via the GitLab-spoof Keycloak client, and the platform JWT consumed by LiteLLM/OPA/Headlamp/LibreChat is shown unchanged by that client's mappers. (→ REQ-07, REQ-08)

## 10. Risks & Open Questions
- (med) Mattermost Team Edition's OSS ceiling (no SAML, no compliance exports, no clustering) is accepted for v1.0; a requirement for any of these reopens the pick (the pattern survives the swap).
- (low) The GitLab-spoof Keycloak client is custom config that must be maintained; if Mattermost ships generic OIDC in OSS it is replaced by a normal OIDC client with no other platform impact.
- (low) `[PROPOSED]` — the exact inbound namespace for chatops invocations ("likely `platform.lifecycle.*`") and adapter method signatures are not specified in source; final binding is design-time per the emitting component / B12.

## 11. References
- ADR 0036 (this decision). Enforcing/realizing components: A19 (Mattermost adapter + webhook source), A4 (broker/Trigger), A7 (OPA routing), A8 (Keycloak client), A14 (HolmesGPT chatops), B12 (event schemas).
- architecture-overview.md §6.7 (eventing), §6.10 (HolmesGPT), §6.11 (identity federation). architecture-backlog.md §5 (open chat-surface question, closed here). ADR 0002, 0004, 0007, 0018, 0023, 0028, 0029, 0031.
