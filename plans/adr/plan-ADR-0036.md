# PLAN ADR-0036 — Two-way Mattermost integration via Knative Eventing as the v1.0 chat-platform pattern `[PROPOSED]`

> spec: SPEC-ADR-0036 · kind: ADR · tier: T2
> wave: authoring-parallel · estimate: M
> upstream-pieces: [A4; A1; A7] · downstream-pieces: [A19; A14; B12]

## 1. Implementation Strategy
Enforcement/verification map. ADR 0036 is enforced at **A19**, which builds the thin driver-shaped
outbound adapter and inbound webhook source against the Knative fabric (A4) — the natural gate for
"no point-to-point coupling," "routing-as-OPA-policy," and "driver interface." SSO isolation is
enforced at **A8** (the Mattermost-dedicated GitLab-spoof Keycloak client kept outside the ADR 0029
claim schema). HolmesGPT (A14) gains chatops via the existing CloudEvent→AgentRun path; B12 registers
the filtered/minted event types. Conformance is verified by broker round-trip tests, an OPA-routing
GitOps test, an identity-attachment test, and a JWT-isolation test.

## 2. Ordered Task List
- TASK-01: Verify outbound Trigger→adapter→Mattermost REST delivery with no direct component→Mattermost call — produces: Chainsaw/integration eventing test — depends-on: []
- TASK-02: Verify routing is OPA policy: change channel mapping via Rego/GitOps with no adapter redeploy; adapter carries no routing logic — produces: integration test — depends-on: [TASK-01]
- TASK-03: Verify inbound webhook→normalized CloudEvent on broker under existing taxonomy — produces: Chainsaw eventing test — depends-on: []
- TASK-04: Verify inbound event carries authenticated Keycloak identity and downstream OPA acts on it — produces: integration test — depends-on: [TASK-03]
- TASK-05: Verify a stub second driver registers without touching topology/OPA-shape/taxonomy — produces: PyTest driver-interface test — depends-on: [TASK-01, TASK-03]
- TASK-06: Verify GitLab-spoof Keycloak client authenticates Mattermost and platform JWT is unchanged by its mappers — produces: integration/JWT-isolation test — depends-on: []
- TASK-07: Verify Mattermost/LibreChat run with distinct audiences/policies/identity scopes — produces: config-conformance check — depends-on: []

## 3. Dependency Map
### 3.1 Upstream pieces that must ship first (HARD)
- A4 — Knative + NATS broker (carries both directions). A1 — LiteLLM (chatops invocation route). A7 — OPA (routing policy evaluation).
### 3.2 Downstream pieces blocked on this
- A19 (adapter + webhook source build), A14 (HolmesGPT chatops surface), B12 (filtered/minted event schemas).
### 3.3 Continuous (non-blocking) inputs
- B16 OPA policy content (routing rules); B8 CloudEvent→AgentRun adapter; A8 Keycloak config; B14 test framework.

## 4. Parallelizable Subtasks
TASK-01, TASK-03, TASK-06, TASK-07 run concurrently. TASK-02 follows TASK-01; TASK-04 follows TASK-03; TASK-05 follows both TASK-01 and TASK-03.

## 5. Test Strategy
- AC-01 → config-conformance (TASK-07): distinct Mattermost/LibreChat audiences/policies.
- AC-02, AC-09(broker-envelope) → Chainsaw/integration outbound delivery (TASK-01).
- AC-03 → integration OPA-routing GitOps change (TASK-02).
- AC-04 → Chainsaw inbound normalization (TASK-03).
- AC-05 → integration identity-attachment + downstream OPA (TASK-04).
- AC-06 → PyTest stub-driver registration (TASK-05).
- AC-07 → integration GitLab-spoof auth + JWT-isolation (TASK-06); fake Keycloak client fixture until A8 lands.

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/authoring`
### 6.2 PR — `piece/ADR-0036-mattermost-chat` → base `wave/authoring`; carries spec + plan.
### 6.3 Merge order — independent of sibling ADR specs; rolls up to main.

## 7. Effort Estimate
TASK-01 S · TASK-02 S · TASK-03 S · TASK-04 S · TASK-05 S · TASK-06 M · TASK-07 S. Rollup M. Critical path: TASK-03 → TASK-04 (inbound identity path).

## 8. Rollback / Reversibility
Reverting backs out the Mattermost adapter + webhook source and the GitLab-spoof Keycloak client;
operators lose the chat surface but the broker, LibreChat, and platform JWT are unaffected (the
adapter is a leaf sink/source, no other component depends on it). The driver interface means a future
chat-platform swap is a driver change, not a re-architecture. No data migration.
