# ADR 0036: Two-way Mattermost integration via Knative Eventing as the v1.0 chat-platform pattern

- **Status**: Accepted
- **Date**: 2026-05-10

## Context

The platform needs a chat surface that is distinct from the LibreChat end-user agent UI (ADR 0007): a place where the platform itself talks to operators — alert dispatch, HolmesGPT findings (architecture-overview.md § 6.10), approval prompts, chatops-style agent invocation — and where humans can talk back. LibreChat is purposely scoped to user-to-agent conversations; it is the wrong audience for operator-facing alerts and team-channel chatops, and bolting that role onto it would muddy its threat model and access controls.

The eventing fabric (architecture-overview.md § 6.7, ADR 0004) already standardizes on Knative Eventing over NATS JetStream with environment-specific sources (ADR 0023) and a fixed CloudEvent top-level taxonomy (ADR 0031). The cleanest integration shape for any chat platform is therefore: outbound traffic is a Trigger-filtered subscription to the broker, and inbound traffic is a webhook source that publishes normalized CloudEvents back onto the broker. This keeps chat traffic inside the same audit, policy, and observability envelope as every other event flow and avoids point-to-point coupling between components and a chat client.

A specific v1.0 pick is needed so the pattern is exercised end-to-end. Mattermost Team Edition is OSS, deployable in-cluster, has a stable webhook and REST API, and is the chat platform most commonly used in regulated / air-gapped environments where this platform is targeted. Architecture-backlog.md § 5 leaves open how non-LibreChat chat surfaces should integrate; this ADR closes that question for v1.0 in a way that does not foreclose Slack or Teams later.

A complication for SSO: Mattermost Team Edition only ships GitLab OAuth in OSS — SAML, generic OIDC, and direct Keycloak integration are Enterprise-only. Web research (DevOpsTales walkthrough, Big Bang configuration templates, Mattermost issue #15606) confirms this is still the case as of 2026 and that the canonical OSS workaround is to configure a Keycloak client with custom protocol mappers that mimic GitLab's OAuth and `/api/v4/user` userinfo response shape. Mattermost is then pointed at Keycloak as if it were a GitLab instance. Without an explicit decision here, the integration would either drift toward an Enterprise-license assumption or invent a per-deployment auth scheme inconsistent with ADR 0028 (identity federation) and ADR 0029 (JWT claim schema).

## Decision

1. **v1.0 chat platform.** Mattermost Team Edition (free / OSS) is the v1.0 platform-operator chat surface. It is deployed in-cluster as part of the A-track install. It is **not** a replacement for LibreChat (ADR 0007); the two have different audiences (operators / chatops vs. end users / agent conversations) and run with different access policies, different identity scopes, and different visibility into platform internals.

2. **Outbound (platform → Mattermost).** A Knative Trigger on the broker filters CloudEvents by `type` (e.g., `platform.observability.alert.*`, `platform.approval.requested`, selected `platform.security.*`) and routes them to a thin Mattermost adapter that calls the Mattermost REST API. Per-event-type → channel routing is expressed as **OPA policy** evaluated on the adapter, since the platform already standardizes on OPA for runtime decisions (ADR 0002, ADR 0018) and this keeps routing rules GitOps-managed and audited like every other policy decision. The adapter has no decision logic of its own; it is pure field-mapping plus the OPA call, matching the "thin Python adapter" shape of every other Knative sink in the architecture (architecture-overview.md § 6.7).

3. **Inbound (Mattermost → platform).** A Mattermost outgoing-webhook posts to a Knative source (a webhook receiver in the same family as the env-specific sources of ADR 0023). The source normalizes the payload into a CloudEvent under the existing taxonomy (ADR 0031) — likely `platform.lifecycle.*` for chatops agent invocations — and publishes it onto the broker. Downstream consumers (HolmesGPT chatops, Platform Agent invocation via the existing CloudEvent → AgentRun adapter) subscribe through normal Triggers. The webhook receiver attaches the authenticated Keycloak identity of the posting user (resolved via the Mattermost-dedicated client described in (5)) to the CloudEvent so OPA decisions downstream see a real platform principal rather than an opaque Mattermost user id.

4. **Driver-shaped adapter.** The outbound adapter and the inbound webhook source are written against a chat-platform-agnostic interface; Mattermost is the first driver. Slack and Teams drivers are deferred to future-enhancements and can be added without touching the Trigger topology, the OPA routing rules' shape, or the CloudEvent taxonomy. Driver responsibilities are limited to API shape translation (Mattermost REST vs. Slack Web API vs. Teams Graph) and webhook payload normalization.

5. **SSO via GitLab-spoof Keycloak client.** The platform installs a dedicated Keycloak client for Mattermost configured with custom protocol mappers that emit a GitLab-shaped OAuth and `/api/v4/user`-shaped userinfo response. Mattermost is configured for GitLab OAuth pointed at that client. The platform's canonical Keycloak claim schema (ADR 0029) is **unaffected** — the GitLab-spoof mapping lives only on this Mattermost-dedicated client and does not leak into the platform JWT consumed by LiteLLM, OPA, Headlamp, or LibreChat. The Mattermost client is provisioned by the same A-track Keycloak configuration (A8 / A4) that handles the other federation work in ADR 0028.

## Consequences

- The "chat platform" role becomes a generic, driver-pluggable integration sitting on the Knative fabric rather than a bespoke per-tool wiring; adding Slack or Teams later is a driver swap, not a re-architecture.
- All outbound chat traffic flows through the broker and through a Trigger, so it inherits the same audit emission, observability, and policy gating as every other event flow (architecture-overview.md § 6.7).
- Routing-as-OPA-policy means "which alert type goes to which channel" is reviewable, GitOps-managed, and changeable without an adapter redeploy — consistent with ADR 0002 / ADR 0018.
- HolmesGPT (architecture-overview.md § 6.10) gains a chatops surface for free: an inbound message becomes a CloudEvent that the existing AgentRun adapter can dispatch to it, and HolmesGPT findings can be routed back out via the outbound Trigger.
- Architecture-backlog.md § 5's open question about non-LibreChat chat surfaces is closed for v1.0; the LibreChat / Mattermost split (end-user vs. operator) becomes the documented pattern.
- The GitLab-spoof Keycloak client is custom configuration that has to be maintained; if Mattermost ever ships generic OIDC in OSS, this client is replaced by a normal OIDC client with no other platform impact.
- The Mattermost-dedicated Keycloak client is **outside** the ADR 0029 platform claim schema, so a Mattermost-side compromise cannot forge a platform JWT. This is an explicit isolation property, not an accident of the design.
- Mattermost Team Edition's OSS feature ceiling (no SAML, no compliance exports, no clustering) is accepted for v1.0; the same Knative-Trigger / OPA-routing / driver-interface pattern survives a future move to Mattermost Enterprise or to a different chat platform entirely.
- Operators get one place to receive `platform.observability.*` alerts, approval prompts, and HolmesGPT findings without those flows leaking into the end-user LibreChat surface.

## References

- [architecture-overview.md](../architecture-overview.md) [§ 6.7](../architecture-overview.md#67-eventing-architecture-knative--nats-jetstream) (Eventing — Knative + NATS), [§ 6.10](../architecture-overview.md#610-platform-self-management-with-holmesgpt) (HolmesGPT), [§ 6.11](../architecture-overview.md#611-identity-federation) (identity federation)
- [architecture-backlog.md § 5](../architecture-backlog.md#5-open-questions-to-leave-open) (open questions about events and chat surfaces)
- [ADR 0002](./0002-opa-gatekeeper-policy-engine.md) (OPA + Gatekeeper as policy engine)
- [ADR 0004](./0004-nats-jetstream-broker.md) (NATS JetStream broker)
- [ADR 0007](./0007-librechat-locked-down-frontend.md) (LibreChat locked-down frontend — different audience)
- [ADR 0018](./0018-rbac-floor-opa-restrictor.md) (RBAC-as-floor / OPA-as-restrictor)
- [ADR 0023](./0023-environment-specific-knative-sources.md) (environment-specific Knative sources)
- [ADR 0028](./0028-identity-federation.md) (identity federation chain)
- [ADR 0029](./0029-keycloak-jwt-claim-schema.md) (Keycloak JWT claim schema)
- [ADR 0031](./0031-cloudevent-top-level-taxonomy.md) (CloudEvent top-level taxonomy)
- DevOpsTales: "Mattermost SSO with Keycloak via GitLab OAuth" (2024, still current 2026)
- Big Bang `mattermost` package values (Keycloak-as-GitLab mapper templates)
- Mattermost issue #15606 (OSS SSO via Keycloak GitLab spoof)
