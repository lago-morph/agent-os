# PLAN A19 — Mattermost adapter

> spec: SPEC-A19 · kind: COMPONENT · tier: T1
> wave: W1 · estimate: L
> upstream-pieces: [A4, A1] · downstream-pieces: []

## 1. Implementation Strategy

Build the bidirectional Mattermost integration as a thin, driver-shaped pair on the Knative fabric: an outbound Trigger→adapter→REST path with OPA-as-routing, and an inbound webhook-receiver→CloudEvent path that attaches a real Keycloak identity and republishes on the broker for the existing B8 adapters to consume. SSO is solved with a dedicated Keycloak GitLab-OAuth-spoof client whose mappers are strictly isolated from the platform JWT (ADR 0029). Develop driver-interface-first so Slack/Teams are later drop-ins, keeping all decision logic in OPA (GitOps-managed, audited) rather than the adapter. Critical path runs through the GitLab-spoof Keycloak client (the riskiest, most version-brittle piece) and the inbound identity-attachment (the security-load-bearing step). Mock the A18 audit adapter, the B8 CloudEvent→AgentRun adapter, and the routing Rego until they land.

## 2. Ordered Task List

- **TASK-01:** Install Mattermost Team Edition in-cluster (Helm) — produces: Mattermost deployment — depends-on: [] (A-track baseline).
- **TASK-02:** Keycloak GitLab-OAuth-spoof client + protocol mappers (GitLab-shaped OAuth + `/api/v4/user` userinfo); point Mattermost at it — produces: dedicated Keycloak client + Mattermost OAuth config — depends-on: [TASK-01] (Keycloak from A8/A4). Verify isolation from platform JWT.
- **TASK-03:** Define the chat-platform-agnostic **driver interface** (API-shape translation + payload normalization) — produces: driver interface — depends-on: [].
- **TASK-04:** Outbound adapter (Mattermost driver): consume Trigger CloudEvents → OPA routing call → Mattermost REST — produces: outbound adapter — depends-on: [TASK-03]. (Mock routing Rego + A18.)
- **TASK-05:** Outbound Knative Trigger filtering by CloudEvent `type` (`platform.observability.alert.*`, `platform.approval.requested`, selected `platform.security.*`) — produces: outbound Trigger — depends-on: [TASK-04] (A4 broker).
- **TASK-06:** Inbound webhook-receiver source: receive Mattermost outgoing-webhook → normalize → CloudEvent → attach resolved Keycloak identity → publish — produces: inbound source — depends-on: [TASK-02, TASK-03] (A4 broker).
- **TASK-07:** OPA routing-as-policy bundle (per-event-type → channel) + dry-run for the simulator (ADR 0038); admission Rego for A19 resources — produces: routing Rego — depends-on: [TASK-04] (B3 contract).
- **TASK-08:** Wire inbound chatops events to the existing B8 CloudEvent→AgentRun adapter + HolmesGPT chatops — produces: chatops dispatch wiring — depends-on: [TASK-06]. (Mock B8 until landed.)
- **TASK-09:** Audit emission via A18 adapter for outbound/inbound flows — produces: audit wiring — depends-on: [TASK-04, TASK-06].
- **TASK-10:** Cross-cutting — `GrafanaDashboard` XR, alerts, runbook (Keycloak re-provision, webhook rotation, misroute triage), HolmesGPT chatops toolset — depends-on: [TASK-07, TASK-08, TASK-09].
- **TASK-11:** 3-layer tests mapping all ACs — depends-on: [TASK-07, TASK-08, TASK-09].
- **TASK-12:** Tutorials & how-tos ("route an alert to a channel"; "invoke an agent from Mattermost chatops") — depends-on: [TASK-10].

## 3. Dependency Map

### 3.1 Upstream that must ship first (HARD)
- **A4 (Knative + NATS broker)** — Trigger/source/sink machinery for outbound + inbound.
- **A1 (LiteLLM)** — gateway path for chatops agent invocations + HolmesGPT findings.

### 3.2 Downstream blocked on this
- None in CSV. De-facto: HolmesGPT (A14) chatops surface; operators receiving alerts/approvals/findings.

### 3.3 Continuous (non-blocking) inputs
- **Keycloak** (A8/A4 config) — the dedicated Mattermost client is provisioned by that same Keycloak config. **B8** CloudEvent→AgentRun adapter (consumes inbound events). **A7/B3** OPA for routing + dry-run. **A18** audit adapter. **A20** simulator (dry-runs routing before rollout). **B12** event schemas. **B14** test framework. **B22** threat model (claim-schema isolation is a named security property).

## 4. Parallelizable Subtasks
- TASK-03 (driver interface) is independent and can start day one.
- After TASK-03: TASK-04 (outbound) and TASK-06 (inbound) are independent fan-out (different drivers' halves).
- TASK-02 (Keycloak spoof) parallel with TASK-03/04.
- TASK-10 (cross-cutting) + TASK-12 (docs) parallelize once flows exist.

## 5. Test Strategy
- **Chainsaw (operator/CRD):** AC-A19-04 (Trigger reconcile + delivery), -06 (webhook source reconcile + normalized event on broker), -08 (inbound dispatches `AgentRun` via B8).
- **PyTest (logic):** AC-A19-05 (routing-as-OPA, no adapter redeploy), -07 (identity attachment → real claims), -09 (driver-interface stub compiles), -10 (audit + policy events), -11 (simulator dry-run sends no message).
- **Playwright (UI/e2e):** AC-A19-01 (Mattermost reachable), -02 (GitLab-spoof SSO login round-trip), -03 (platform JWT unchanged; GitLab-shape only on Mattermost client), -12 (LibreChat/Mattermost distinct scopes).
- **Fakes/fixtures:** mock A18 adapter, mock B8 adapter, mock routing Rego/A20, a Mattermost test instance + Keycloak realm fixture. Inbound webhook is an ADR 0023 env-specific source — kind uses the webhook receiver; cloud-source failure modes untestable on kind.

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/1` (contains A4, A1 specs A19 builds on).
### 6.2 PR — `piece/A19-mattermost-adapter` → base `wave/1`; carries SPEC-A19 + PLAN-A19.
### 6.3 Merge order — independent of W1 siblings; wave/1 rolls up to main.

## 7. Effort Estimate
- Install + Keycloak spoof (TASK-01, -02): M (mapper config is fiddly and version-brittle).
- Driver + outbound + inbound (TASK-03..06): M (two thin halves; identity attachment is the careful part).
- Routing Rego + chatops wiring + cross-cutting + tests + docs (TASK-07..12): M.
- **Rollup: L** (matches CSV). **Critical path:** TASK-01 → 02 (GitLab-spoof) → 06 (inbound identity) → 08 (chatops dispatch) → 11.

## 8. Rollback / Reversibility
Back out by deleting the outbound Trigger, inbound webhook source, adapters, and the Mattermost install; remove the dedicated Keycloak client. Because routing lives in OPA (not the adapter) and identity isolation is confined to the Mattermost client, rollback does not touch the platform JWT or any other event flow. **Downstream break:** operators lose the Mattermost alert/approval/findings surface and HolmesGPT loses its chatops entry point — but LibreChat and all other event flows are unaffected (the LibreChat/Mattermost split is clean). Mattermost data store is deleted with the install (back up first if message history matters).
