# SPEC A19 — Mattermost adapter

> kind: COMPONENT · workstream: A · tier: T1
> upstream: [A4, A1] · downstream: [] · adrs: [0002, 0004, 0007, 0018, 0023, 0028, 0029, 0030, 0031, 0034, 0036] · views: [6.7, 6.10, 6.11]
> canon-glossary: 6aadcc2a4f38 · canon-interface: 54f5ede58e5f

## 1. Purpose & Problem Statement

A19 integrates **Mattermost Team Edition** as the platform's **operator-facing, bidirectional chat surface** via Knative Eventing (ADR 0036). It is deliberately distinct from LibreChat (ADR 0007): LibreChat is the end-user / agent-conversation UI; Mattermost is where the platform talks to operators (alert dispatch, HolmesGPT findings, approval prompts, chatops agent invocation) and operators talk back. Both run with different audiences, identity scopes, and access policies.

The integration follows the architecture's eventing shape: **outbound** (platform → Mattermost) is a Knative **Trigger** filtering CloudEvents by `type` to a thin Mattermost adapter that calls the Mattermost REST API, with **per-event-type → channel routing expressed as OPA policy** on the adapter; **inbound** (Mattermost → platform) is a Mattermost outgoing-webhook posting to a Knative webhook-receiver source that normalizes the payload into a CloudEvent and republishes it on the broker. Both halves are written against a **chat-platform-agnostic driver interface** (Mattermost is the first driver; Slack/Teams are future drop-ins). Because Team Edition lacks native OIDC/SAML, SSO is provided via a **dedicated Keycloak client configured with GitLab-OAuth-spoof protocol mappers**, isolated from the platform JWT claim schema (ADR 0029) so a Mattermost compromise cannot forge a platform token.

## 2. Scope

### 2.1 In scope
- **Mattermost Team Edition install** (in-cluster, A-track).
- **Keycloak GitLab-OAuth-spoof client + protocol mappers** emitting a GitLab-shaped OAuth + `/api/v4/user`-shaped userinfo response; Mattermost configured for GitLab OAuth pointed at that client.
- **Outbound adapter** — a thin Knative sink calling the Mattermost REST API; **no decision logic** beyond an OPA call for per-event-type → channel routing.
- **Inbound webhook-receiver source** — receives Mattermost outgoing-webhooks, normalizes payload → CloudEvent (under the existing taxonomy), attaches the authenticated Keycloak identity of the posting user, republishes on the broker.
- **OPA-driven Trigger-filter library** for channel routing (routing-as-policy, GitOps-managed, audited).
- **Generic chat-platform driver interface** — Mattermost as first driver; Slack/Teams deferred.
- Standard §14.1 deliverables: Helm/manifests, per-product docs, runbook, alerts, Grafana dashboard (XR), Headlamp plugin (where useful), OPA/Rego integration (routing + admission), audit emission (via A18 adapter), Knative trigger-flow design, HolmesGPT toolset (chatops surface), 3-layer tests, tutorials/how-tos.

### 2.2 Out of scope (and where it lives instead)
- **The Knative broker + NATS JetStream + Trigger machinery** → **A4**. A19 adds Triggers/sinks/sources but does not own the broker.
- **Environment-specific event *sources*** generally → ADR 0023 / install team; A19's inbound webhook-receiver is one source in that family but is a documented A19 deliverable.
- **CloudEvent per-type names + schemas** → **B12**. A19 commits to existing namespaces only.
- **The CloudEvent → AgentRun adapter / CloudEvent → Argo Workflow adapter** → **B8**. A19's inbound events are consumed by those existing adapters; A19 does not reimplement them.
- **OPA engine / framework / content** → A7 / B3 / B16; A19 contributes its routing + admission Rego.
- **The platform JWT claim schema (ADR 0029)** → unchanged by A19; the GitLab-spoof mapping lives only on the Mattermost-dedicated client.
- **The rest of Keycloak federation (ADR 0028)** → A8 / A4 Keycloak config; A19 adds only the Mattermost client.
- **LibreChat** (different audience) → **A8** / ADR 0007.
- **Slack / Teams drivers** → future-enhancements.

## 3. Context & Dependencies

**Upstream consumed (HARD):**
- **A4 (Knative Eventing + NATS JetStream broker)** — the broker, Triggers, and source/sink machinery the outbound Trigger and inbound webhook-receiver attach to.
- **A1 (LiteLLM)** — the gateway path; relevant because chatops agent invocations and HolmesGPT findings route through Platform Agents that call out via LiteLLM. (CSV lists A1 as upstream.)

**Effective dependencies (named, not all in CSV):**
- **Keycloak** (ADR 0028/0029, configured by A8/A4) — the dedicated Mattermost client is provisioned by that same A-track Keycloak configuration.
- **A18 audit adapter** — A19 emits audit through it (the adapter is the only audit path).
- **A7 / B3** — OPA for routing-as-policy and admission.
- **B8** — existing CloudEvent→AgentRun adapter consumes A19's inbound chatops events.

**Downstream consumers:** none in CSV. De-facto: HolmesGPT (A14) gains a chatops surface; operators receive `platform.observability.*` alerts, approval prompts, HolmesGPT findings.

**ADRs honored:**
- **ADR 0036** — Mattermost Team Edition; bidirectional via Knative; outbound Trigger→adapter→REST with OPA channel routing; inbound webhook→source→CloudEvent with attached Keycloak identity; driver-shaped adapter; GitLab-spoof Keycloak client isolated from the platform JWT.
- **ADR 0004 / 0023** — NATS JetStream broker; the webhook-receiver is an environment-specific source in the ADR 0023 family.
- **ADR 0002 / 0018** — OPA as the runtime decision engine for routing; RBAC-as-floor / OPA-as-restrictor.
- **ADR 0028 / 0029** — identity federation; the Mattermost client is **outside** the platform claim schema (explicit isolation property).
- **ADR 0007** — Mattermost is NOT a LibreChat replacement; different audience/policies.
- **ADR 0031 / 0030** — CloudEvent taxonomy + versioning. **ADR 0034** — audit via the adapter.

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
N/A — A19 defines no platform CRD/XRD. It deploys Knative resources: a **Trigger** (outbound, filtered by CloudEvent `type`), a **webhook-receiver source** (inbound), and **sink** services — all Knative-native kinds owned by A4's install, not A19 CRDs. The Keycloak client + protocol mappers are Keycloak config artifacts, not CRDs. `[PROPOSED — not in source]` if any A19 config is later modeled as a CRD; none is required by source.

### 4.2 APIs / SDK surfaces
- **Outbound adapter** — consumes CloudEvents off a Trigger; calls the **Mattermost REST API**; performs one **OPA call** for channel routing; otherwise pure field-mapping (no decision logic). Conforms to the "thin Python adapter" shape (§6.7).
- **Inbound webhook-receiver source** — HTTP endpoint receiving Mattermost **outgoing-webhook** posts; normalizes → CloudEvent; attaches the resolved Keycloak identity of the posting user; publishes to the broker. Likely `platform.lifecycle.*` for chatops agent invocations (ADR 0036).
- **Chat-platform driver interface** — `[PROPOSED — not in source]` method surface; source fixes only that driver responsibilities are limited to **API-shape translation** (Mattermost REST vs Slack Web API vs Teams Graph) + **webhook payload normalization**. A19 defines the interface; exact method signatures `[PROPOSED]`.
- HTTP APIs follow URL-path versioning `/v1/...` (interface-contract §3.3).

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
- **Consumed (outbound routing):** filtered by `type` — e.g. `platform.observability.alert.*`, `platform.approval.requested`, selected `platform.security.*` (the example set from ADR 0036; exact filters are design-time / B12).
- **Emitted (inbound normalization):** chatops posts normalized to **`platform.lifecycle.*`** (ADR 0036's stated likely namespace) and published to the broker; consumed by the existing CloudEvent→AgentRun adapter (B8) and HolmesGPT chatops.
- **`platform.policy.*`** — OPA routing decisions (and dry-runs for the simulator) are audited under this namespace.
- **`platform.audit.*`** — outbound/inbound chat traffic inherits audit emission like every other event flow, via the A18 adapter.
- Per-event-type names/schemas → **B12**; `[PROPOSED — not in source]` for any concrete A19 event name; A19 commits only to namespaces.

### 4.4 Data schemas / connection-secret contracts
- **Mattermost REST API credentials** + **outgoing-webhook secret** materialized via ESO (`[PROPOSED — not in source]` exact secret shape).
- **Keycloak GitLab-spoof client** config: protocol mappers producing GitLab-shaped OAuth + `/api/v4/user` userinfo. This mapping is **isolated** to the Mattermost client and MUST NOT alter the ADR 0029 platform JWT. Field-level mapper config `[PROPOSED — not in source]` (source cites the GitLab `/api/v4/user` shape and the DevOpsTales/Big Bang templates as the pattern).
- No substrate connection secret (no DB owned by A19).

## 5. OSS-vs-Custom Decision
- **Mattermost Team Edition** — **adopt OSS + config**. Free/OSS tier explicitly chosen (ADR 0036). No fork. Version pin `[PROPOSED — not in source]`.
- **Outbound adapter + inbound webhook-receiver source** — **build-new** thin Python services (the standard Knative adapter/source shape, §6.7), written against the driver interface.
- **Driver interface** — **build-new** chat-platform-agnostic abstraction; Mattermost the first driver.
- **Keycloak GitLab-spoof client** — **config** (custom protocol mappers on a dedicated Keycloak client); the canonical workaround for Team Edition's OSS SSO ceiling (ADR 0036, DevOpsTales / Big Bang / Mattermost #15606).
- **OPA routing** — **config/content** (Rego in the policy library) rather than adapter-internal logic, so routing is GitOps-managed and audited.
- Rationale: keep the adapter thin and driver-pluggable so Slack/Teams are driver swaps, not re-architecture (ADR 0036).

## 6. Functional Requirements
- **REQ-A19-01:** A19 MUST install **Mattermost Team Edition** in-cluster as part of the A-track install.
- **REQ-A19-02:** A19 MUST provision a **dedicated Keycloak client** with **GitLab-OAuth-spoof protocol mappers** emitting a GitLab-shaped OAuth + `/api/v4/user`-shaped userinfo response, and configure Mattermost for GitLab OAuth pointed at it.
- **REQ-A19-03:** The GitLab-spoof mapping MUST be **isolated to the Mattermost client** and MUST NOT alter or leak into the ADR 0029 platform JWT consumed by LiteLLM/OPA/Headlamp/LibreChat.
- **REQ-A19-04:** The **outbound** path MUST be a Knative **Trigger** filtering CloudEvents by `type` routed to a thin Mattermost adapter that calls the Mattermost REST API.
- **REQ-A19-05:** Per-event-type → channel routing MUST be expressed as **OPA policy** evaluated on the adapter; the adapter MUST contain **no routing decision logic** of its own beyond field-mapping + the OPA call.
- **REQ-A19-06:** The **inbound** path MUST be a Mattermost outgoing-webhook posting to a Knative **webhook-receiver source** that normalizes the payload into a CloudEvent under the existing taxonomy and publishes it on the broker.
- **REQ-A19-07:** The inbound source MUST **attach the authenticated Keycloak identity** of the posting user to the CloudEvent so downstream OPA sees a real platform principal, not an opaque Mattermost user id.
- **REQ-A19-08:** Inbound chatops events MUST be consumable by the **existing CloudEvent → AgentRun adapter (B8)** and HolmesGPT chatops without A19 reimplementing those adapters.
- **REQ-A19-09:** The outbound adapter and inbound source MUST be written against a **chat-platform-agnostic driver interface** (Mattermost = first driver); adding Slack/Teams MUST NOT require changing the Trigger topology, OPA-routing shape, or CloudEvent taxonomy.
- **REQ-A19-10:** All outbound and inbound chat traffic MUST flow through the broker/Trigger so it inherits the platform's **audit, observability, and policy** envelope (via the A18 adapter and `platform.policy.*`/`platform.audit.*`).
- **REQ-A19-11:** The OPA channel-routing bundle MUST expose the structured **dry-run** mode (ADR 0038) so the policy simulator (A20) can evaluate routing decisions without sending a message.
- **REQ-A19-12:** A19 MUST NOT position Mattermost as a LibreChat replacement; the two MUST run with separate identity scopes and access policies (ADR 0007).

## 7. Non-Functional Requirements
- **Security:** Mattermost-side compromise **cannot forge a platform JWT** (claim-schema isolation, REQ-A19-03) — an explicit isolation property, not an accident. Mattermost egress (REST API calls out, if external) rides the Envoy egress proxy. Inbound webhook authenticates the posting user to a real Keycloak identity. Team Edition's OSS ceiling (no SAML/compliance-export/clustering) is accepted for v1.0.
- **Multi-tenancy (§6.9):** OPA routing decisions read Keycloak claims (`platform_tenants`/`platform_namespaces`/`platform_roles`) so channel routing and chatops invocation respect tenant scope; the resolved identity (REQ-A19-07) carries those claims downstream.
- **Observability (§6.5):** Grafana dashboard (XR) for outbound delivery rate/failures, inbound webhook rate, routing-deny counts. Alerts on adapter delivery failure, webhook-source errors, Keycloak client misconfiguration.
- **Scale:** adapter + source are stateless/horizontally scalable Knative services; bounded by broker throughput (A4).
- **Versioning (ADR 0030):** adapter/source HTTP APIs URL-path-versioned; driver interface versioned so Slack/Teams enroll additively.

## 8. Cross-Cutting Deliverable Checklist
- Helm/manifests — **applicable** (Mattermost install, Keycloak client config, outbound Trigger+adapter, inbound webhook-receiver source, routing Rego).
- Per-product docs (10.5) — **applicable** (Mattermost integration + GitLab-spoof setup; LibreChat-vs-Mattermost audience split).
- Runbook (10.7) — **applicable** (Keycloak client re-provision, webhook secret rotation, routing-policy change, channel-misroute triage).
- Backup/restore — **applicable** (Mattermost data store; standard Mattermost backup, `[PROPOSED]` specifics).
- Alerts — **applicable** (delivery failure, webhook error rate, SSO failure). `[PROPOSED]` thresholds.
- Grafana dashboard (Crossplane XR) — **applicable**.
- Headlamp plugin — **where useful** — `[PROPOSED]` minimal (routing-policy inspection); primary editing is via A22/OPA editors.
- OPA/Rego integration — **applicable** (channel-routing-as-policy + admission of A19 resources).
- Audit emission (ADR 0034) — **applicable, via A18 adapter** (chat traffic inherits audit).
- Knative trigger flow — **applicable (core deliverable)** — outbound Trigger + inbound webhook source are A19's defining flows.
- HolmesGPT toolset — **applicable** (chatops surface; HolmesGPT findings routed out, inbound messages dispatched to it via B8).
- 3-layer tests — **applicable** (Chainsaw: Trigger/source reconcile; PyTest: adapter field-mapping + OPA routing + identity attachment; Playwright: end-to-end message round-trip incl. SSO login).
- Tutorials & how-tos — **applicable** ("route a platform alert to a Mattermost channel"; "invoke an agent from Mattermost chatops").

## 9. Acceptance Criteria
- **AC-A19-01:** Mattermost Team Edition installs in-cluster and is reachable. (→ REQ-A19-01)
- **AC-A19-02:** A user logs into Mattermost via the GitLab-spoof Keycloak client (GitLab OAuth flow) and receives a session. (→ REQ-A19-02)
- **AC-A19-03:** Inspecting a platform JWT (LiteLLM/OPA/Headlamp/LibreChat) shows the ADR 0029 claim schema is unchanged by the Mattermost client; the GitLab-shaped userinfo appears only on the Mattermost client. (→ REQ-A19-03)
- **AC-A19-04:** A `platform.observability.alert.*` CloudEvent on the broker is filtered by a Trigger and delivered to a Mattermost channel via REST. (→ REQ-A19-04)
- **AC-A19-05:** Changing the OPA routing policy reroutes an event type to a different channel with **no adapter redeploy**; the adapter contains no hardcoded routing. (→ REQ-A19-05)
- **AC-A19-06:** A Mattermost outgoing-webhook post produces a normalized CloudEvent on the broker under the existing taxonomy. (→ REQ-A19-06)
- **AC-A19-07:** The normalized inbound CloudEvent carries the posting user's resolved Keycloak identity; an OPA decision downstream reads real platform claims. (→ REQ-A19-07)
- **AC-A19-08:** An inbound chatops message dispatches an `AgentRun` via the existing B8 adapter without A19-specific adapter code. (→ REQ-A19-08)
- **AC-A19-09:** A stub second driver compiles against the driver interface without touching Trigger topology, OPA-routing shape, or the taxonomy. (→ REQ-A19-09)
- **AC-A19-10:** An outbound delivery and an inbound message each produce audit events via the A18 adapter and a `platform.policy.*` routing decision. (→ REQ-A19-10)
- **AC-A19-11:** The policy simulator (A20) evaluates the channel-routing bundle in dry-run, returning the routed channel with `simulated: true` and sending no message. (→ REQ-A19-11)
- **AC-A19-12:** Mattermost and LibreChat run with distinct identity scopes/policies; no shared end-user session crosses them. (→ REQ-A19-12)

## 10. Risks & Open Questions
- **R1 (med):** GitLab-spoof protocol-mapper config is `[PROPOSED — not in source]` at field level and **brittle** to Mattermost/Keycloak version drift; if Mattermost ships OSS OIDC, the client is replaced (ADR 0036). Blast radius med (SSO only). Mitigation: pin versions; document mapper templates.
- **R2 (med):** Exact outbound **CloudEvent `type` filter set** and channel-routing rule content are design-time / B12 — `[PROPOSED]`. Risk of mis-routing sensitive `platform.security.*` events to wrong channels. Mitigation: simulator (A20) dry-run before rollout (REQ-A19-11).
- **R3 (low):** Driver-interface method surface is `[PROPOSED — not in source]`; over-fitting to Mattermost REST could leak into the interface and complicate Slack/Teams. Mitigation: limit driver scope to API-shape translation + payload normalization (ADR 0036).
- **R4 (low):** Inbound webhook-receiver is an ADR 0023 env-specific source; on kind it is the webhook receiver, but cloud-source failure modes are untestable on kind (ADR 0023 cost note).
- **OQ1:** Which CloudEvent namespace do non-chatops inbound messages normalize to if not `platform.lifecycle.*`? Deferred to B12 / design.
- **OQ2:** Is the Mattermost REST credential a system identity or per-channel bot token? `[PROPOSED]` system bot token via ESO.
- **OQ3:** Does A19 need a CRD to model channel-routing config, or is Rego sufficient? `[PROPOSED]` Rego-only (no CRD), per ADR 0036 routing-as-OPA-policy.

## 11. References
- architecture-overview.md §6.7 (line 553; eventing — Knative+NATS, Triggers/adapters/sinks; line 630 Mattermost integration paragraph), §6.10 (HolmesGPT chatops), §6.11 (identity federation), §14.1 (line 1685; A19 deliverable).
- ADR 0036 (Mattermost Team Edition; bidirectional Knative; outbound Trigger+OPA routing; inbound webhook→CloudEvent+identity; driver-shaped adapter; GitLab-spoof Keycloak client isolated from platform JWT). ADR 0004/0023 (NATS broker; env-specific sources). ADR 0002/0018 (OPA; RBAC-floor). ADR 0028/0029 (identity federation; claim schema). ADR 0007 (LibreChat — different audience). ADR 0031/0030 (event taxonomy / versioning). ADR 0034 (audit). ADR 0038 (simulator dry-run for routing bundle).
- Related pieces: A4 (Knative broker), A1 (LiteLLM), A8 (Keycloak/LibreChat), B8 (CloudEvent→AgentRun adapter), A7/B3 (OPA), A18 (audit), A14 (HolmesGPT), A20 (simulator).
