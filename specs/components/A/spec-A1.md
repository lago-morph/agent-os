# SPEC A1 — LiteLLM (gateway)

> kind: COMPONENT · workstream: A · tier: T0
> upstream: [] · downstream: [A16, A17, A19, A20, B1, B2, B16, B6, B9, B13, B17, B18] · adrs: [0006, 0001, 0030, 0031, 0034, 0013, 0002, 0003, 0035, 0020] · views: [6.1, 6.8]
> canon-glossary: 6aadcc2a4f383a26 · canon-interface: 54f5ede58e5f8c05

## 1. Purpose & Problem Statement

LiteLLM is the single chokepoint for every LLM, MCP, and A2A call a Platform Agent makes (architecture-overview.md §6.1). Beyond LLM routing it brokers MCP tool calls, peers A2A (internal and approved external), issues virtual keys, and performs **OpenAI-compatible ↔ A2A translation** so frontends like LibreChat see Platform Agents as models. A1 is the install-+-configure-+-operate package for the gateway: the LiteLLM Helm chart (with the kopf operator as a subchart per ADR 0006), the callbacks pipeline configuration, the MCP/A2A/RAG registry wiring, and the dynamic-registration path.

The problem it solves is **a single governed perimeter for all model/tool/agent traffic**. Without a single chokepoint, authentication, OPA policy, audit emission, PII scrubbing, and Langfuse tracing would have to be re-implemented at every call site. By forcing all such traffic through LiteLLM's callbacks pipeline, the platform gets one place to enforce policy and emit audit, and capability composition becomes declarative (CRDs → kopf → LiteLLM registry) rather than gateway-internal config.

A1 is **T0, contract-owning, highest blast radius** — almost every other component depends on it: the agent runtime (A6), MCP services (A17), Mattermost (A19), the policy simulator (A20), the kopf operator (B13), the SDK (B6), the CLI (B9), and the agent profile/composition libraries (B17/B18).

## 2. Scope

### 2.1 In scope

- LiteLLM proxy install via Helm with `replicas >= 2` Deployment.
- The **callbacks pipeline** configuration (hook chain at `pre-call`, `post-call`, `success`, `failure`) and registration of the platform callback classes.
- **OpenAI-compatible ↔ A2A translation** so Platform Agents appear as models; stream-chunk preservation end-to-end.
- The **A2A endpoint** (internal calls + outbound to approved external `A2APeer`s).
- The **MCP gateway** (tool namespacing, OAuth flows) consuming the `MCPServer` registry.
- The **MCP/A2A/RAG registry** populated by B13 from CRDs.
- **Virtual key auth + Keycloak claims** verification on inbound requests (Platform JWT claim schema).
- **Dynamic agent registration** of A2A/MCP interfaces at agent startup, OPA-gated.
- **LLM provider failover** behavior (single/multiple provider rules per §6.1).
- **MCP server health** signal surfacing (metrics, audit, traces) to the extent MCP exposes it.
- ConfigMap-driven config with **Reloader-triggered rolling restart** per ADR 0035 (no in-process reload).
- Helm packaging of B13 kopf operator as a **subchart** (ADR 0006).
- Emission of `platform.gateway.*` and `platform.audit.*` CloudEvents and audit via the audit adapter (ADR 0034).
- The §14.1 standard deliverable set (docs, runbook, alerts, dashboard, Headlamp plugin integration, OPA integration, audit, Knative trigger flow, HolmesGPT toolset, 3-layer tests, tutorials).

### 2.2 Out of scope (and where it lives instead)

- **Callback implementations** (PII/Presidio, OPA bridge, audit, guardrails) — owned by **B2** (LiteLLM custom callbacks); A1 ships the pipeline + registration surface and the *mandatory* audit + OPA-bridge callbacks ride here only if A1 lands first (ADR 0002, §14.2 B2 note). A1 defines the registration contract; B2 supplies logic.
- **CRD reconciliation into LiteLLM** (`MCPServer`, `A2APeer`, `RAGStore`, `EgressTarget`, `Skill`, `CapabilitySet`, `VirtualKey`, `BudgetPolicy`) — owned by **B13** (kopf operator), packaged as A1's subchart.
- **OPA engine + Rego bundles** — **A7** (engine), **B3/B16** (policy content).
- **The audit endpoint + audit adapter library** — **A18** (A1 *links* the adapter, does not implement it).
- **Langfuse** install — **A2**.
- **Envoy egress** (non-LiteLLM HTTP) — **A6**.
- **OpenAI ↔ A2A adapter if LiteLLM OSS lacks translation** — fallback lives in **A8** (LibreChat) per §14.1 A8 note.
- **CapabilitySet overlay resolution semantics** — ADR 0032 + B13 design.
- **CloudEvent per-event-type schemas** — B12 registry.

## 3. Context & Dependencies

**Upstream consumed:** None (W0 foundation; no internal deps). LiteLLM is an OSS product installed onto the k8-platform baseline.

**Downstream consumers (what they consume):**
- A16 Interactive Access Agent, A17 MCP services, A19 Mattermost, B17/B18 agents — the gateway call path (LLM/MCP/A2A) + virtual keys.
- A20 policy simulator — the LiteLLM OPA-callback **dry-run** decision surface.
- B1 oauth2-proxy — fronts LiteLLM for SSO.
- B2 callbacks — register into A1's callbacks pipeline.
- B6 Platform SDK — terminates `memory.*`, `rag.*`, model invocation, A2A registration helpers at LiteLLM.
- B9 CLI, B13 kopf operator — admin API.

**ADR decisions honored:**
- **ADR 0006** — kopf operator ships as a **subchart of the LiteLLM Helm chart**; single ArgoCD release; no gateway/operator version skew. The gateway exposes an HTTP admin API versioned `/v1/...` (§6.13).
- **ADR 0001** — ARK is the agent operator; agents reach LiteLLM through the SDK call path; this constrains A1 to present the OpenAI-compat ↔ A2A surface ARK-run agents expect.
- **ADR 0031** — every CloudEvent A1 emits falls under exactly one of the ten top-level namespaces (`platform.gateway.*` for routing/failover/MCP-health/A2A-handoffs; `platform.audit.*` for audit; `platform.observability.*` for budget-exceeded).
- **ADR 0034** — audit emission is **only** via the platform audit adapter to the audit endpoint; A1 **never** writes audit directly to Postgres/S3/OpenSearch.
- **ADR 0013** — capability registry shape; capability changes emit `platform.capability.changed` (emitted by B13, not A1).
- **ADR 0002 / 0018** — OPA callbacks are restrictors over an RBAC-bounded identity (RBAC-as-floor / OPA-as-restrictor); budgets enforced through OPA, not LiteLLM admin UI.
- **ADR 0003** — non-LiteLLM HTTP goes through Envoy, not LiteLLM (perimeter invariant).
- **ADR 0035** — config reload = rolling restart via Reloader (no in-process reload).
- **ADR 0030** — CRD/CloudEvent/HTTP-API versioning policy.
- **ADR 0020** — the v1.0 initial MCP service set brokered by LiteLLM.

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)

A1 does **not own** any CRD reconciler (B13 owns the LiteLLM-facing CRDs). A1 **consumes** the registry these CRDs produce. Owned-by-B13 CRDs A1 reads via the registry (versioned `v1alpha1` → `v1beta1` → `v1` per ADR 0030, owner B13):

| CRD | Scope | Fields A1 relies on (source-stated) |
|---|---|---|
| `MCPServer` | namespaced | `endpoint`, `authMode` (system/user-cred), `credentialsRef`, `tags`, `scopes`, `visibility` |
| `A2APeer` | namespaced | `endpoint`, `direction` (internal/external), `auth`, `tags` |
| `RAGStore` | namespaced | `backend`, `indexes[]`, `contentSourceRefs[]`, `ingestionPipelineRef` |
| `Skill` | namespaced | `gitRef`, `versionPin`, `schemaRef` (served via LiteLLM's skill gateway) |
| `VirtualKey` | namespaced | `ownerIdentity`, `capabilitySetRef`, `budgetRef`, `environment`, `allowedModels[]`, `ttl` |
| `BudgetPolicy` | namespaced | `scope`, `period`, `limits`, `thresholdActions[]` (consumed by OPA + LiteLLM) |
| `CapabilitySet` | namespaced | `mcpServers[]`, `a2aPeers[]`, `ragStores[]`, `egressTargets[]`, `skills[]`, `llmProviders[]`, `opaPolicyRefs[]` |
| `Agent` (ARK, A5) | namespaced | `exposes` (A2A/MCP) drives dynamic-registration intent; `modelRef`, `capabilitySetRefs[]` |

CRDs owned/reconciled here: **N/A — A1 owns no reconciler; B13 (subchart) reconciles all LiteLLM-facing CRDs (ADR 0006).**

### 4.2 APIs / SDK surfaces

- **OpenAI-compatible HTTP API** (inbound) — LiteLLM's OSS surface; Platform Agents and frontends call it; translated to A2A where the target is a Platform Agent. Stream chunks preserved.
- **A2A endpoint** — internal + outbound-to-approved-external. External orgs do **not** self-register (§6.1).
- **MCP gateway** — tool namespacing + OAuth flows against the `MCPServer` registry.
- **LiteLLM admin API** — `/v1/...` URL-path versioned (§6.13); driven by B13, not by hand.
- **Callbacks pipeline** — Python hook chain firing at `pre-call`, `post-call`, `success`, `failure`; each hook is a registered class. Registration contract owned here; logic in B2. `[PROPOSED — not in source]` the exact class registration signature / ConfigMap key names beyond "configured in the LiteLLM ConfigMap" (§6.1).
- **Dry-run surface for the policy simulator (A20)** — the LiteLLM OPA callback exposes a structured dry-run input and emits the same decision shape with `simulated: true`, with no enforcement-path side effects (§6.6, ADR 0038). `[PROPOSED — not in source]` exact dry-run request/response field names (A20/ADR 0038 own the composition; A1 owns the per-layer shape — to reconcile with A20).

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)

Emitted (each under exactly one closed namespace; **per-event-type names deferred to B12 registry**):
- `platform.gateway.*` — routing decisions, **provider failover**, **MCP health changes**, **A2A handoffs**.
- `platform.audit.*` — every request/response incl. MCP calls and A2A handoffs (via audit adapter, ADR 0034).
- `platform.observability.*` — **budget-exceeded** notification (the v1.0 budget-exceeded → email trigger flow, §6.7).
- `platform.policy.*` — OPA decision/violation events surfaced from the callback path (decision authored by OPA; A1 is the emit point at the gateway). `[PROPOSED — not in source]` exact event-type names — B12.

Consumed:
- `platform.capability.changed` (ADR 0013) — emitted by B13; A1's registry is updated by B13 reconcile, not by A1 subscribing. (The SDK consumes the event for agent notification; A1 enforcement is request-time.)

Every event carries CloudEvents `specversion` + per-event-type `schemaVersion` (ADR 0030/0031).

### 4.4 Data schemas / connection-secret contracts

- A1 holds no system-of-record state. Virtual-key and budget tracking state is LiteLLM-internal (spend tracking is built-in, §6.6); the **declarative source of truth is the CRDs** (B13), not LiteLLM's store.
- Provider/MCP/A2A credentials arrive as Kubernetes Secrets via **ESO** (referenced by `credentialsRef` on `MCPServer`); A1 does not mint secrets.
- Audit records are posted to the audit endpoint via the adapter library (no direct DB writes). `[PROPOSED — not in source]` LiteLLM internal Postgres usage for spend/key state is implied by "tracks spend per virtual key" but the backing store shape is not specified in source — treat as LiteLLM-internal, not a platform connection-secret contract.

## 5. OSS-vs-Custom Decision

- **Upstream project:** LiteLLM (the gateway). **Decision: config + wrap** — install OSS LiteLLM via Helm, configure the callbacks pipeline + registry, wrap with the B13 kopf operator subchart (ADR 0006). Do **not** fork.
- **Custom code that rides here:** the callbacks pipeline registration surface; B2 supplies callback logic; B13 supplies reconciliation. The OpenAI ↔ A2A translation is expected from LiteLLM OSS; **if OSS does not ship it, the adapter is built in A8** (§14.1 A8), not forked into A1.
- **Enterprise-feature caveat (§6.6):** basic budget tracking + per-virtual-key budgets are OSS; advanced budget features (alert routing, automated suspensions) **may be enterprise-only — to be confirmed during implementation**. If needed-and-enterprise-only, extend via callbacks. The architectural commitment is **OPA = policy layer, LiteLLM = spend-tracking layer** regardless.
- **Version pinning** per §9 OSS-limitations mitigation; ADR 0006 binds operator/gateway versions via the subchart.

## 6. Functional Requirements

- **REQ-A1-01:** LiteLLM SHALL be installed via Helm as a Deployment with `replicas >= 2`.
- **REQ-A1-02:** The kopf operator (B13) SHALL ship as a **subchart of the LiteLLM Helm chart**, deployed as a single ArgoCD release with lifecycle bound to LiteLLM (ADR 0006).
- **REQ-A1-03:** The gateway SHALL authenticate every inbound request via virtual-key auth and verify Keycloak Platform JWT claims (`platform_tenants`, `platform_namespaces`, `platform_roles`, `tenant_roles`, `capability_set_refs`).
- **REQ-A1-04:** The gateway SHALL provide OpenAI-compatible ↔ A2A translation so Platform Agents appear as models to OpenAI-compatible clients, preserving stream chunks end-to-end.
- **REQ-A1-05:** The A2A endpoint SHALL accept internal Platform-Agent-to-Platform-Agent calls and outbound calls to approved external `A2APeer`s, and SHALL NOT permit external parties to self-register.
- **REQ-A1-06:** The MCP gateway SHALL broker MCP tool calls against the `MCPServer` registry with tool namespacing and OAuth flows.
- **REQ-A1-07:** The callbacks pipeline SHALL fire registered callback classes at `pre-call`, `post-call`, `success`, and `failure`, configured via the LiteLLM ConfigMap.
- **REQ-A1-08:** Configuration changes SHALL take effect via a **Reloader-triggered graceful rolling restart** against the `replicas >= 2` Deployment; A1 SHALL NOT rely on in-process reload (ADR 0035).
- **REQ-A1-09:** Platform Agents declaring `exposes` (A2A/MCP) SHALL register dynamically at startup, gated by an OPA check on whether the agent may expose that interface in its namespace; the entry SHALL be deregistered on agent termination.
- **REQ-A1-10:** LLM provider failover SHALL behave: one-provider-available → use it; one-provider-unavailable → error; multiple-all-unavailable → error; multiple-≥1-available → silent failover to next-priority working provider.
- **REQ-A1-11:** Admission validation SHALL reject an Agent CapabilitySet declaring zero LLM providers.
- **REQ-A1-12:** On MCP server health degradation, the gateway SHALL emit metrics, audit events, and traces, and surface unhealthy status to the calling agent's error path; A1 SHALL NOT build a synthetic health layer beyond MCP's signals.
- **REQ-A1-13:** Audit emission for every request/response (incl. MCP and A2A handoffs) SHALL go through the platform audit adapter to the audit endpoint; A1 SHALL NOT write audit directly to Postgres/S3/OpenSearch (ADR 0034).
- **REQ-A1-14:** All CloudEvents A1 emits SHALL fall under exactly one of the ten top-level namespaces and carry `specversion` + `schemaVersion` (ADR 0031/0030).
- **REQ-A1-15:** The LiteLLM OPA callback SHALL enforce budgets by consulting OPA per request (given current spend + `BudgetPolicy`), and SHALL act as a restrictor only (never grant beyond RBAC; ADR 0018).
- **REQ-A1-16:** The gateway SHALL broker the ADR 0020 v1.0 MCP service set (GitHub, Google Drive, Context7, OpenSearch system-mediated + external-credentialed, Postgres, MongoDB, web-search, web-scrape) as registered `MCPServer`s.
- **REQ-A1-17:** The LiteLLM OPA callback SHALL honor a structured **dry-run** input and emit the same decision shape with `simulated: true` and no enforcement-path side effects, for the policy simulator (A20/ADR 0038).
- **REQ-A1-18:** The admin HTTP API SHALL be URL-path versioned `/v1/...` with deprecated versions reachable ≥1 platform release after replacement (§6.13).

## 7. Non-Functional Requirements

- **Security / multi-tenancy (§6.9):** virtual-key claims bind requests to tenant/namespace; OPA restricts per-decision; RBAC-as-floor (ADR 0018). Cross-tenant A2A blocked at both gateway authz and Envoy egress (defense-in-depth, §6.9). Secrets via ESO only.
- **Availability:** `replicas >= 2`; rolling restart must be graceful (no dropped in-flight streams beyond unavoidable reconnect). Provider failover keeps agent-visible behavior predictable.
- **Observability (§6.5):** OTel emission to the collector → Tempo; Langfuse for prompts/costs/evals; Mimir metrics; per-component Grafana dashboard delivered as a Crossplane `GrafanaDashboard` XR.
- **Streaming:** stream chunks preserved through LibreChat ↔ LiteLLM ↔ Platform Agent for interactive and long-running agents.
- **Versioning (ADR 0030):** admin HTTP API `/v1/...`; CloudEvent `schemaVersion`; consumed CRDs follow B13 versioning.
- **Scale:** sized for v1.0 event/request volumes; budget/spend tracking is per-virtual-key.

## 8. Cross-Cutting Deliverable Checklist (§14.1)

| Deliverable | Status |
|---|---|
| Helm values / manifests in Git | applicable — LiteLLM chart + B13 subchart |
| Per-product docs (10.5) | applicable |
| Operator runbook (10.7) | applicable — incl. provider-failover, rolling-restart, MCP-health triage |
| Backup / restore | applicable — LiteLLM-internal spend/key state; source of truth is CRDs in Git (rebuild by re-reconcile) |
| Alert rules | applicable — gateway down, callback failures, provider-all-unavailable, MCP-unhealthy |
| Grafana dashboard (Crossplane `GrafanaDashboard` XR) | applicable |
| Headlamp plugin | applicable — A1 owns making the capability/per-agent view work in our env (cross-cutting plugin owned by B5; A1 ensures integration) |
| OPA/Rego integration | applicable — runtime tool/model authz, budget enforcement, dynamic-registration approval (Rego content in B16) |
| Audit emission (ADR 0034) | applicable — via adapter; the densest emission point in the platform |
| Knative trigger flow | applicable — budget-exceeded → email (v1.0); gateway-event triggers |
| HolmesGPT toolset | applicable — gateway metrics/log queries, MCP-health, provider-status tools |
| 3-layer tests (Chainsaw / Playwright / PyTest) | applicable |
| Tutorials & how-tos | applicable — virtual-key issuance, registering an MCP server, A2A peering |

## 9. Acceptance Criteria

- **AC-A1-01** (REQ-A1-01): `kubectl get deploy litellm` shows `replicas >= 2` and all pods Ready. *(Chainsaw)*
- **AC-A1-02** (REQ-A1-02): Installing the LiteLLM Helm chart yields exactly one ArgoCD release containing both the gateway and the B13 operator pods. *(Chainsaw)*
- **AC-A1-03** (REQ-A1-03): A request with an invalid/absent virtual key is rejected; a request with a valid key whose JWT lacks the target namespace claim is denied. *(PyTest)*
- **AC-A1-04** (REQ-A1-04): An OpenAI-format chat-completion request targeting a Platform Agent returns a valid streamed response; stream chunk boundaries are preserved vs the agent's emitted chunks. *(PyTest)*
- **AC-A1-05** (REQ-A1-05): An internal A2A call between two Platform Agents succeeds; an attempt by an unregistered external endpoint to register itself is rejected. *(PyTest)*
- **AC-A1-06** (REQ-A1-06): An MCP tool call routed through the gateway resolves to the namespaced tool from the registered `MCPServer`. *(PyTest)*
- **AC-A1-07** (REQ-A1-07): A registered test callback observes invocations at all four hook points for a single request. *(PyTest)*
- **AC-A1-08** (REQ-A1-08): Editing the LiteLLM ConfigMap triggers a Reloader rolling restart; pods cycle gracefully and the new config is active; no in-process reload path is used. *(Chainsaw)*
- **AC-A1-09** (REQ-A1-09): An Agent with `exposes` set and an allowing OPA policy becomes discoverable in the registry after startup; with a denying policy it does not; on termination the entry is removed. *(Chainsaw + PyTest)*
- **AC-A1-10** (REQ-A1-10): For each of the four failover cases the gateway returns the specified outcome; multi-provider silent failover is invisible to the agent. *(PyTest)*
- **AC-A1-11** (REQ-A1-11): Admission rejects an Agent whose resolved CapabilitySet has zero LLM providers. *(Chainsaw)*
- **AC-A1-12** (REQ-A1-12): A simulated MCP outage emits a metric + audit event + trace and surfaces an error to the agent's error path. *(PyTest)*
- **AC-A1-13** (REQ-A1-13): For a sampled request, an audit record reaches the audit endpoint via the adapter and there is **no** direct gateway write to Postgres/S3/OpenSearch. *(PyTest)*
- **AC-A1-14** (REQ-A1-14): Every CloudEvent emitted in a test run matches one of the ten namespaces and carries `specversion` + `schemaVersion`. *(PyTest)*
- **AC-A1-15** (REQ-A1-15): With a `BudgetPolicy` over its limit, the OPA callback denies the next call; the callback never returns allow for an action RBAC would deny. *(PyTest)*
- **AC-A1-16** (REQ-A1-16): Each ADR 0020 MCP service is reachable as a registered `MCPServer` through the gateway in an integration run. *(PyTest)*
- **AC-A1-17** (REQ-A1-17): A dry-run OPA-callback request returns a decision with `simulated: true` and produces no audit-enforcement side effect (only the `platform.policy.*` simulator-run audit). *(PyTest)*
- **AC-A1-18** (REQ-A1-18): The admin API responds under `/v1/...`; a deprecated prior version (when one exists) remains reachable. *(PyTest)*

## 10. Risks & Open Questions

- **R1 (high):** Advanced budget features may be enterprise-only (§6.6). *Reconciliation:* commit to OPA-as-policy + callbacks-extension; confirm during implementation. Blast radius high (budget enforcement is platform-wide).
- **R2 (high):** Does LiteLLM OSS ship OpenAI ↔ A2A translation? If not, the adapter lands in A8 (§14.1 A8). *Open question* — must be settled before A8/A16 build.
- **R3 (med):** Mandatory audit + OPA-bridge callbacks "ship with LiteLLM (A1)" but logic is B2; if A1 lands before B2/A7, the OPA-bridge callback moves into A7 (§14.2 B2/ADR 0002). Ownership handoff must be explicit to avoid a gap. *[PROPOSED]* the exact callback-registration interface.
- **R4 (med):** Rolling-restart on config change (ADR 0035) can drop in-flight streams; runbook must define graceful drain. Blast radius med.
- **R5 (low):** Dry-run decision-shape field names for A20 are `[PROPOSED — not in source]`; reconcile with A20/ADR 0038 before either freezes its contract.
- **OQ1:** ConfigMap key names / callback class registration signature are not in source → `[PROPOSED — not in source]`.

## 11. References

- architecture-overview.md §6.1 Gateway architecture (line ~170), §6.8 Capability registries (~632), §6.6 (~436, audit/OPA hook points, budget-through-OPA), §6.7 (~553, budget-exceeded trigger flow), §6.5 (~370, observability), §6.13 (~981, versioning).
- ADR 0006 (kopf operator subchart), ADR 0001 (ARK), ADR 0031 (CloudEvent taxonomy), ADR 0034 (audit pipeline), ADR 0013 (capability CRDs / `platform.capability.changed`), ADR 0002 (OPA/Gatekeeper), ADR 0018 (RBAC-floor/OPA-restrictor), ADR 0003 (Envoy egress — perimeter invariant), ADR 0035 (dynamic toggle / rolling restart), ADR 0020 (initial MCP services), ADR 0030 (versioning).
- Related pieces: B13 (kopf operator), B2 (callbacks), A7 (OPA), A18 (audit), A6 (Envoy/sandbox), A20 (policy simulator), A8 (LibreChat).
