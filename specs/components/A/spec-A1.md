# SPEC A1 — LiteLLM (gateway)

> kind: COMPONENT · workstream: A · tier: T0
> upstream: [] · downstream: [A16, A17, A19, A20, B1, B2, B6, B9, B13, B16, B17, B18] · adrs: [0006, 0001] · views: [6.1, 6.8]
> canon-glossary: b0edae10 · canon-interface: 0ce201d5

## 1. Purpose & Problem Statement

LiteLLM is the **single mandatory chokepoint** for all model, tool, and inter-agent traffic on the platform (component A1; §6.1). No Platform Agent talks to a model provider, an MCP server, or another Platform Agent directly; every such call is brokered through LiteLLM. This is the architectural property that makes governance, observability, and cost control uniform across the platform — it is the trust boundary between Platform Agents and everything external (providers, MCP servers, A2A peers) and is therefore a tier-0 availability and security concern (§6.1, lines 172–175, 211).

The problem A1 solves is *uniform external enforcement of every agent egress*: model routing/failover, virtual-key issuance scoped to a CapabilitySet + identity, MCP server brokering (tool calls proxied and policy-checked), A2A brokering (inter-agent calls), skill-gateway management, and emission of callbacks/hooks that feed audit, observability, and policy. OpenAI-compatible HTTP requests are translated to A2A at the gateway so agents written against the OpenAI API can call peers natively (§6.1, lines 174, 188). The gateway is **never hand-configured**: capability state is reconciled into it by the kopf operator (B13) from CRDs (ADR 0006; §6.8), and request-time behaviour is governed by B2 callbacks + OPA.

A1 is the install + configure + operate package for upstream LiteLLM: it pins a tested version (Helm), **packages the B13 kopf operator as a subchart of the LiteLLM Helm chart so gateway and operator ship as a single ArgoCD release with no version skew (ADR 0006; §6.1 line 245)**, exposes the admin/virtual-key surface that B13 is the sole writer of, ships the callback/hook registration surface that B2 fills, provisions backing Postgres via the `Postgres` substrate XR (Crossplane v2 namespace-scoped composite resource), and delivers the standard Workstream A deliverables (§6.1; §14.1). A1 ships the hook *registration*; B2 ships the *handlers* — the split is deliberate so the gateway install and the policy/audit logic version independently. Config changes take effect via a **Reloader-triggered rolling restart against a `replicas >= 2` Deployment — LiteLLM has no in-process reload (ADR 0035; §6.1 line 214)**.

## 2. Scope

### 2.1 In scope

- LiteLLM install (Helm values / manifests in Git) pinned to a tested upstream version; non-secret config GitOps-managed, secret material via ESO (§6.1, lines 207, 222).
- The five logical gateway surfaces (§6.1, line 178): OpenAI-compatible completion/chat API; MCP brokering surface; A2A brokering surface; skill-gateway admin surface; virtual-key admin API (consumed only by B13).
- OpenAI-compatible → A2A translation at the gateway, so an agent written against the OpenAI API can call an internal Platform Agent or an approved external `A2APeer` transparently (§6.1, line 188).
- MCP brokering: proxying tool calls to approved `MCPServer`s, applying the pre-call OPA decision, recording the call, and forwarding using the credentials the platform holds per `MCPServer.authMode` (`system` or `system-mediated`; §6.1, line 186). A1 owns auth-mode enforcement: `system` mode uses credentials for system/platform processes (not tenant-specific); `system-mediated` mode uses credentials LiteLLM holds on behalf of a specific tenant's agents, with LiteLLM enforcing per-tenant isolation. The retired `user-cred` mode is not supported. Agents never hold or manage credentials directly (D-01).
- A2A brokering: internal and approved-external inter-agent calls, honoring `A2APeer.direction` (internal/external) and `A2APeer.auth` (§6.1, line 188).
- Model routing/failover via LiteLLM's native router with the source-stated four-case behaviour (§6.1, lines 224–231): one provider available → use it; one provider unavailable → error; multiple all unavailable → error; multiple with ≥1 available → silent failover to the next-priority working provider. A model alias (`Agent.modelRef`) maps to one or more provider deployments; provider credentials are platform-managed secrets never exposed to agents (§6.1, line 184). Admission rejects a resolved CapabilitySet declaring zero LLM providers (§6.1 line 224).
- The B13 kopf operator packaged as a **subchart** of the LiteLLM Helm chart (single ArgoCD release, lifecycle bound to LiteLLM, automatic version pinning; ADR 0006; §6.1 line 245).
- Config delivered via ConfigMap with **Reloader-triggered graceful rolling restart** (no in-process reload; ADR 0035; §6.1 line 214) against the `replicas >= 2` Deployment.
- PII scrubbing (Presidio) and Langfuse instrumentation as callback-pipeline stages — registration here, handler logic in B2 (§6.1 line 214).
- A **dry-run decision surface** on the OPA callback for the policy simulator (A20): same decision shape with `simulated: true` and no enforcement-path side effects (§6.6 simulator; ADR 0038).
- Brokering the ADR 0020 v1.0 MCP service set (GitHub, Google Drive, Context7, OpenSearch, Postgres, MongoDB, web-search, web-scrape) as registered `MCPServer`s (§6.1 lines 233–243).
- Virtual-key enforcement: the gateway enforces the key (bound to identity from the Platform JWT, a `CapabilitySet`, a `BudgetPolicy`, with `allowedModels`/`ttl` constraints); **B13 is the only writer** of key state via the admin API (§6.1, line 182; ADR 0006).
- The skill gateway managing `Skill` artifacts referenced by CapabilitySets (§6.1, line 178).
- Callback/hook **registration** at the three defined points — pre-call (authn/authz, budget check, OPA decision), post-call (audit emission, trace recording, cost accounting), on-failure (provider failover, error audit) — as the B2 integration surface (§6.1, line 180).
- Two-layered budget enforcement: LiteLLM per-key/per-model spend tracking integrated with `BudgetPolicy` CRDs; threshold crossings emit `platform.observability.*` and, per `thresholdActions`, may block or require approval (§6.1, line 190).
- Dynamic-registration enforcement point: when an `Agent` declaring `exposes` (A2A/MCP surfaces) is admitted, registering those surfaces with the gateway is OPA-gated and emits `platform.policy.*` (§6.1, line 192).
- Emission of `platform.gateway.*` events (routing decisions, provider failover, MCP health changes, A2A handoffs); audit via the audit adapter under `platform.audit.*`; budget/threshold under `platform.observability.*`; dynamic-registration accept/deny under `platform.policy.*` (§6.1, line 194).
- Three observability planes wired from the gateway: audit (compliance, via A18 adapter), OTel traces (Tempo), LLM-traces/generations (Langfuse) — correlated by `trace_id` (ADR 0015; §6.1, lines 205, 209).
- Backing Postgres (key/spend/config state) provisioned via the `Postgres` substrate XR (Crossplane v2 namespace-scoped composite resource); gateway code identical on kind vs AWS (§6.1, lines 196, 198).
- HA posture: stateless-per-request, horizontally scalable, PodDisruptionBudgets, readiness gated on backing store; fail-closed governance behaviour (§6.1, lines 196, 201).
- HolmesGPT gateway toolset (routing, redacted key state, provider health, budget posture) (§6.1, line 205).
- Standard Workstream A deliverables: per-product docs, operator runbook, alerts, `GrafanaDashboard` XR, Headlamp plugin, OPA/Rego integration surface, three-layer tests, tutorials & how-tos (§14.1).

### 2.2 Out of scope (and where it lives instead)

- Reconciling capability CRDs (`MCPServer`, `A2APeer`, `RAGStore`, `EgressTarget`, `Skill`, `CapabilitySet`, `VirtualKey`, `BudgetPolicy`) into gateway admin state — **B13** kopf operator (ADR 0006/0013). A1 ships the admin API and *packages* B13 as a subchart; B13 owns the reconciliation logic and is the only writer.
- The OpenAI-compat ↔ A2A translation adapter **if upstream LiteLLM OSS does not ship it** — fallback lands in **A8** (LibreChat), not forked into A1 (§14.1 A8 note). A1 assumes the upstream translation surface and surfaces the gap as a risk (R-A1-9).
- The callback **handler logic** (OPA decision execution, audit-event construction, trace recording) — **B2** (LiteLLM custom callbacks). A1 ships only the hook registration points.
- OPA policy **content** / Rego (including break-glass / fail-open exceptions) — **B3** (framework) / **B16** (initial content). A1 calls the decision point; it does not author policy.
- The audit endpoint + audit adapter library implementation — **A18** (ADR 0034). A1 links against the adapter; it never writes audit records directly to Postgres/S3/OpenSearch.
- The Knative broker, Triggers, and event delivery — **A4**; the per-event-type schemas — **B12**. A1 emits CloudEvents; it does not own the mesh or the schema registry.
- The `Postgres` substrate XRD + Compositions — **B4** (ADR 0044). A1 *creates/consumes* a `Postgres` XR in its own namespace; it does not author the Composition.
- Identity/JWT issuance (Keycloak) and the SSO proxy in front of UIs — **Keycloak** (ADR 0028/0029) / **B1**. A1 consumes the Platform JWT claims; it does not issue them.
- Egress proxying for non-gateway traffic — **A6** Envoy egress proxy. A1 governs model/tool/A2A egress only.
- Initial MCP services themselves (GitHub, Drive, Context7, etc.) — **A17** (ADR 0020). A1 brokers to them once B13 registers them.
- Letta memory / RAG backends — **A10** / **A11**. A1 brokers RAG capability references; it does not own the stores.
- ESO install — **A17/A15** context; A1 consumes secrets ESO syncs.

## 3. Context & Dependencies

**Upstream pieces consumed (HARD):** None in the wave sense — A1 is W0 foundation (`piece-index.csv`: empty upstream; waves.md W0). At install it creates a namespace-scoped `Postgres` XR for its backing store, but B4 (Crossplane Compositions) is W1; per the platform's phased posture A1 bootstraps against a directly-provisioned Postgres on kind and is rewired onto the `Postgres` XR as B4 lands (see §7 bootstrap tolerance; §6.1 line 196).

**Upstream pieces consumed (continuous / non-blocking):** B14 test framework, B22 threat-model standards (waves.md fan-out edges); A13 (Tempo + Mimir) and A2 (Langfuse) as trace/metric destinations; A18 audit adapter library (links against as available — durable-buffering bootstrap path, ADR 0034); A7 (OPA/Gatekeeper) for the pre-call decision point (fails closed if unreachable).

**Downstream consumers (`piece-index.csv`):** A16 (Interactive Access Agent calls models/peers through A1), A17 (initial MCP services brokered by A1), A19 (Mattermost adapter), A20 (policy simulator exercises gateway decisions), B1 (SSO/auth proxy fronting A1's surfaces), B2 (callbacks register into A1's hooks), B6 (Platform SDK targets the gateway API), B9 (`agent-platform` CLI drives gateway/key state via CRDs), B13 (kopf operator writes A1's admin API), B16 (OPA content targets gateway decisions), B17/B18 (agent profiles/compositions reference `modelRef` and capabilities resolved at A1).

**ADR decisions honored:**

- **ADR 0006** — LiteLLM has no native CRD interface; B13 (kopf) is the **only writer** of LiteLLM admin-API state; A1 ships the admin API + virtual-key surface and versions that contract with B13 via URL-path `/v1` (interface-contract §3.3). A1 is never hand-configured.
- **ADR 0001** — LiteLLM is one of the external enforcement perimeters (with OPA, Envoy) for the ARK-run agent model; the gateway is THE chokepoint while ARK does not enforce egress/policy itself.
- **ADR 0013** (capability model, via §6.8) — capabilities are CRDs reconciled into the gateway; unapproved capabilities are unreachable; the gateway/OPA consume `visibility`/`scopes`/`tags` on capability CRDs for per-call decisions.
- **ADR 0031 / §6.7** — the gateway emits only under the closed ten-namespace CloudEvent taxonomy (`platform.gateway.*`, `platform.audit.*`, `platform.observability.*`, `platform.policy.*`); per-event-type names are deferred to B12.
- **ADR 0034** — audit is durable-async via the adapter (A18), never direct-write; policy is synchronous-blocking — this asymmetry is deliberate (§6.1 line 203). A1 emits audit events via the audit-adapter interface; emission is gated on the audit-adapter freeze-gate (D-05) — A1 cannot emit audit events until the frozen audit-adapter interface and `audit_events` schema pass the freeze-gate.
- **ADR 0015** — gateway OTel traces (Tempo) and LLM-traces (Langfuse) correlate by `trace_id`; three distinct observability planes (§6.1 line 209).
- **ADR 0044** — backing Postgres via the namespace-scoped `Postgres` XR (Crossplane v2 substrate pattern); gateway code identical across kind/AWS substrate.
- **ADR 0030 / §6.13** — per-component versioning; A1 owns the version of its admin/OpenAI/MCP/A2A/skill HTTP surfaces (URL-path `/v1`, interface-contract §3.3); it owns no platform CRD reconciler (B13 owns the capability CRDs).

## 4. Interfaces & Contracts

Use ONLY Canon names. Anything not in Canon is tagged `[PROPOSED — not in source]`.

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)

A1 **owns no CRD reconciler**. The capability CRDs that govern gateway state are reconciled by **B13** (interface-contract §1.4; ADR 0006). A1 *consumes* their reconciled effect and *enforces* it. The CRDs whose effect lands in the gateway (source-stated fields only):

| CRD | Owner (reconciler) | Source-stated fields A1 enforces |
|---|---|---|
| `MCPServer` | B13 | `endpoint`, `authMode` (`system`/`system-mediated`), `credentialsRef`, `tags`, `scopes`, `visibility` |
| `A2APeer` | B13 | `endpoint`, `direction` (internal/external), `auth`, `tags` |
| `RAGStore` | B13 | `backend`, `indexes[]`, `contentSourceRefs[]`, `ingestionPipelineRef` |
| `EgressTarget` | B13 | `fqdn`, `port`, `scheme`, `allowedMethods` (egress-target enforcement is shared with A6 Envoy) |
| `Skill` | B13 | `gitRef`, `versionPin`, `schemaRef` (managed via the gateway's skill gateway) |
| `CapabilitySet` | B13 | `mcpServers[]`, `a2aPeers[]`, `ragStores[]`, `egressTargets[]`, `skills[]`, `llmProviders[]`, `opaPolicyRefs[]` |
| `VirtualKey` | B13 | `ownerIdentity`, `capabilitySetRef`, `budgetRef`, `environment`, `allowedModels[]`, `ttl` |
| `BudgetPolicy` | B13 | `scope` (key/agent/team/tenant), `period`, `limits`, `thresholdActions[]` |

- Substrate XR consumed: **`Postgres`** (namespace-scoped XR, Crossplane v2) — `version`, `size`, `storage`, `connectionSecretRef`, `substrateClass` (interface-contract §1.6; ADR 0044). A1 creates the `Postgres` XR in its own namespace and consumes the resulting connection secret (`host`, `port`, `user`, `password`, `dbname`).
- A1 produces **no** XRD/Composition.
- `[PROPOSED — not in source]` LiteLLM's *internal* admin-API object shapes (key records, model-deployment config) are upstream-LiteLLM data structures, not Canon CRD fields; A1 reuses upstream shapes and exposes them only to B13.

### 4.2 APIs / SDK surfaces

The five logical surfaces (§6.1 line 178), all URL-path versioned `/v1/...` (interface-contract §3.3):

1. **OpenAI-compatible completion/chat API** — what agents call; translated to A2A/MCP as needed.
2. **MCP brokering surface** — proxied, OPA-gated tool calls to `MCPServer`s.
3. **A2A brokering surface** — internal + approved-external inter-agent calls; OpenAI→A2A translation.
4. **Skill-gateway admin surface** — manages `Skill` artifacts.
5. **Virtual-key admin API** — the B13-only writer surface (issue/revoke/update keys, register MCP servers/peers, model deployments).

- **Callback/hook registration** at pre-call / post-call / on-failure — the B2 handler integration surface (§6.1 line 180). A1 defines the points and their invocation contract; B2 provides handlers. `[PROPOSED — not in source]` exact callback function signatures are not in Canon; they follow upstream LiteLLM's callback interface and are versioned with B2 (do not coin new names here).
- Platform SDK (B6) targets the OpenAI-compatible API; B6 ships a version-pinned compatibility matrix against the gateway version (interface-contract §3.1).
- Metrics surface: Prometheus metrics scraped into Mimir; OTel traces to Tempo; LLM-traces/generations to Langfuse (§6.1 lines 205, 220).

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)

- **Emitted** (each under exactly one of the closed ten namespaces; §6.1 line 194). Per QN-03, exactly one component owns each namespace; A1 **owns `platform.gateway`** and takes an explicit dependency on the owner of every other namespace it emits under:
  - `platform.gateway.*` — routing decisions, provider failover, MCP health changes, A2A handoffs. **A1 is the sole owner of `platform.gateway`**: A1 authors and registers the schema for this namespace in B12 (event catalogue). Dashboards and cost reporting take a dependency on A1 as the owner; they do not co-own it.
  - `platform.observability.*` — budget/threshold crossings (e.g. budget-exceeded → email user trigger flow, interface-contract §2). **Schema owned by A13** (Tempo + Mimir); A1 emits under it as a dependent, not an owner (QN-03).
  - `platform.policy.*` — dynamic-registration accept/deny decisions. **Schema owned by A7** (OPA engine); A1 emits under it as a dependent, not an owner (QN-03).
  - `platform.audit.*` — pure audit emissions, **via the A18 audit adapter** (not direct, not over the broker as system of record; ADR 0034). **Schema owned by A18** (audit endpoint); A1 emits as a dependent. Emission is gated on the audit-adapter freeze-gate (D-05).
  - `platform.security.*` — repeated-authn-failure / policy-bypass-attempt and other security-relevant signals at the gateway. **The `platform.security` namespace schema is owned by A7** (OPA engine); A1 is a dependent emitter, not an owner (QN-03). For any security-relevant event A1 detects, A1 MUST both perform its existing local handling AND additionally emit the event to the event bus under `platform.security` (schema owned by A7). `[PROPOSED — not in source]` concrete event types deferred to B12.
- **Consumed:** A1 does not subscribe to the broker on the synchronous request path; if the broker is down, A1 buffers/drops non-critical events rather than blocking the call (§6.7 line 584). Budget/observability notifications (e.g. budget-exceeded → email) are routed downstream by Triggers (A4) to subscribers, not consumed by A1.
- Per-event-type **names and schemas** under each namespace are **deferred to B12's schema registry**; each event carries `specversion` + `schemaVersion` (ADR 0030/0031). `[PROPOSED — not in source]` concrete gateway event-type names are not in Canon and must be registered in B12, not coined here.

### 4.4 Data schemas / connection-secret contracts

- A1's backing store (key/spend/config) is **Postgres**, provisioned via the namespace-scoped `Postgres` XR; A1 consumes the uniform connection secret shape `host`, `port`, `user`, `password`, `dbname` (interface-contract §4; ADR 0044). XR status is substrate-agnostic (`ready`, `endpoint`, `version`); A1 must not depend on substrate-specific fields.
- Provider credentials, MCP `credentialsRef` material, and `A2APeer.auth` material are platform-managed secrets abstracted by External Secrets Operator (ESO); never exposed to agents (§6.1 line 184; glossary ESO). The pull chain is: external store → ESO updates the k8s Secret → reloader restarts the consumer. **LiteLLM credential-entry / PushSecret flow:** LiteLLM reads credentials from a Secret in its own namespace; operators enter credentials (including OAuth — operator-entered secrets via the LiteLLM GUI, no OAuth-lifecycle machinery) through the LiteLLM GUI; LiteLLM writes them to a k8s Secret; ESO **PushSecret** propagates them to the external store. For each secret A1 handles, the spec declares whether the external store (pull) or LiteLLM/k8s (push) is authoritative — never both on one secret: provider/MCP/A2A credentials entered via the LiteLLM GUI are **push-authoritative** (LiteLLM/k8s → ESO PushSecret → external store); secrets sourced from an external store are **pull-authoritative**. agent-os startup gates on the reloader's presence; if the reloader is absent, startup fails.
- A1 produces **no** substrate connection secret of its own (it is a consumer).
- `[PROPOSED — not in source]` the exact column set of LiteLLM's spend/key tables is upstream-internal, not a Canon data schema; treated as opaque backing state behind the `Postgres` XR.

## 5. OSS-vs-Custom Decision

- **Upstream project:** **LiteLLM** (the gateway), the Canon-named product (glossary). **Mode: config + wrap**, not fork.
- Install unmodified at a pinned, tested version via Helm; non-secret config GitOps-managed; secrets via ESO (§6.1 lines 207, 222).
- **Wrap** with platform deliverables: the B2 callback handlers (registered into A1's hook points), B13 as the sole admin-API writer (ADR 0006), audit-adapter linkage (A18), OTel/Langfuse/Mimir wiring, the `Postgres` backing-store XR, a Headlamp plugin, and a HolmesGPT toolset.
- **Custom built net-new is deliberately minimal** in A1: the reconciliation control loop is B13, the callback logic is B2, the policy is B3/B16 — A1 is the *install + surface-exposure + operate* package. This keeps the gateway upgradeable and the custom logic versioned separately (ADR 0006 rationale: kopf chosen for Python ecosystem alignment).
- **Rationale:** LiteLLM is the chosen multi-provider gateway with native router, virtual keys, MCP/A2A brokering, and a callback model that exactly matches the pre/post/on-failure enforcement points the platform needs (§6.1).
- `[PROPOSED — not in source]` exact upstream LiteLLM version/chart coordinates are not stated in Canon; selected and pinned at install time and recorded in the runbook.

## 6. Functional Requirements

- REQ-A1-01: A1 installs LiteLLM via Helm values/manifests in Git at a single pinned, tested upstream version; the pin is recorded and ArgoCD-synced; no non-secret config is applied by hand.
- REQ-A1-02: All gateway secret material (provider credentials, MCP/A2A credentials) is delivered via ESO and is never exposed to any Platform Agent or returned on any agent-facing surface.
- REQ-A1-03: A1 exposes exactly the five logical surfaces — OpenAI-compatible API, MCP brokering, A2A brokering, skill-gateway admin, virtual-key admin — each URL-path versioned `/v1/...`.
- REQ-A1-04: The virtual-key admin API accepts writes **only** from the B13 kopf operator identity; no other caller (including operators by hand) can mutate key/capability state; the gateway is never hand-configured.
- REQ-A1-05: A1 enforces each `VirtualKey` by binding the Platform-JWT identity to its `capabilitySetRef`, `budgetRef`, `environment`, `allowedModels[]`, and `ttl`; a call outside `allowedModels[]` or past `ttl` is rejected.
- REQ-A1-06: An OpenAI-compatible request targeting an internal Platform Agent or an approved external `A2APeer` is translated to an A2A invocation; `A2APeer.direction` and `A2APeer.auth` govern reachability and authentication.
- REQ-A1-07: A tool call to an `MCPServer` is proxied through the gateway, gated by the pre-call OPA decision, recorded, and forwarded using system or per-user credentials per `MCPServer.authMode`; an unapproved (unregistered) MCP server is unreachable.
- REQ-A1-08: Model routing maps `Agent.modelRef` to one or more provider deployments via LiteLLM's native router with failover/load-balancing; provider credentials remain platform-managed secrets.
- REQ-A1-09: A1 registers callback/hook points at pre-call (authn/authz, budget check, OPA decision), post-call (audit emission, trace recording, cost accounting), and on-failure (provider failover, error audit); B2 handlers attach at these points.
- REQ-A1-10: The pre-call OPA decision is **synchronous-blocking and fails closed** (deny) when OPA is unreachable, except where a documented break-glass policy (owned by B3/B16) applies.
- REQ-A1-11: Audit emission is **durable-async via the A18 audit adapter library**; the audit path does not block the call, and A1 writes no audit record directly to Postgres/S3/OpenSearch.
- REQ-A1-12: A1 integrates `BudgetPolicy` two-layered (LiteLLM per-key/per-model tracking + policy `scope`/`period`/`limits`/`thresholdActions[]`); a threshold crossing emits `platform.observability.*` and, per `thresholdActions`, blocks further calls or requires approval.
- REQ-A1-13: When an `Agent` declaring `exposes` (A2A/MCP) is admitted, registration of those surfaces with the gateway is OPA-gated and emits a `platform.policy.*` accept/deny event.
- REQ-A1-14: A1 emits `platform.gateway.*` for routing/failover/MCP-health/A2A-handoff events; every emitted CloudEvent falls under exactly one of the closed ten namespaces and carries `specversion` + `schemaVersion`; A1 introduces no new top-level namespace.
- REQ-A1-15: A1 emits OTel traces to Tempo and LLM-traces/generations to Langfuse, both correlated by `trace_id` (three distinct observability planes), and exposes Prometheus metrics scraped into Mimir.
- REQ-A1-16: A1's backing key/spend/config store is Postgres consumed via a `Postgres` XR using the uniform connection-secret shape; no gateway code differs between kind and AWS substrate.
- REQ-A1-17: A1 is stateless-per-request and horizontally scalable with PodDisruptionBudgets and readiness gated on the backing store; loss of the gateway fails closed (all agent model/tool/A2A traffic stops) by design.
- REQ-A1-18: A1 ships a HolmesGPT gateway toolset exposing routing state, redacted key state, provider health, and budget posture.
- REQ-A1-19: A1 owns the versioning lifecycle of its `/v1` HTTP surfaces (URL-path versioning; deprecated versions reachable ≥1 platform release after replacement) per interface-contract §3.3 / ADR 0030.
- REQ-A1-20: A1 delivers per-product docs, an operator runbook, component-failure alert rules, a `GrafanaDashboard` Crossplane XR, a Headlamp plugin, the OPA decision-point integration, and three-layer tests.

## 7. Non-Functional Requirements

- **Security / trust boundary (§6.1 line 211; §6.9):** A1 is the trust boundary between Platform Agents and all external endpoints. Posture is fail-closed, externally policy-governed (OPA), secret-isolated (ESO; provider creds never reach agents), and audit-durable (A18). Multi-tenancy: virtual keys and budgets are per-identity/CapabilitySet; tenant attributes derive from the Platform JWT (`platform_tenants`, `platform_namespaces`, etc., ADR 0029); cross-tenant capability reach is impossible (unapproved capabilities unreachable, §6.8 line 612). Threats catalogued by B22.
- **Observability (§6.5; ADR 0015):** three planes — audit (compliance, A18), OTel traces (Tempo, technical), LLM-traces (Langfuse, prompt-level) — correlated by `trace_id`. Prometheus metrics → Mimir. Component-failure alerts required (gateway down, backing-store unreachable, OPA-unreachable deny rate, provider-failover rate, budget-threshold rate).
- **Availability / scale (§6.1 lines 196, 201):** tier-0 hot path for every agent call; stateless-per-request + HA replicas + PDBs; readiness gated on backing Postgres. Fail-closed is the explicit availability/security trade.
- **Versioning (ADR 0030 / §6.13; interface-contract §3.3):** per-component; A1 owns `/v1` URL-path versioning for all five surfaces and the B13 admin contract; deprecated versions served ≥1 platform release.
- **Substrate parity (ADR 0044):** identical gateway behaviour on kind vs AWS; only backing Postgres substrate differs via `Postgres`. No substrate-specific code paths.
- **Bootstrap tolerance:** A1 is W0 but several enforcement/observability dependencies land later (A7 OPA points, A18 audit endpoint, B13 reconciler, B4 `Postgres`). Degradation path: synchronous OPA fails closed when absent; audit buffers durably when A18 absent; gateway serves last-reconciled state if B13 is down (eventually consistent, ADR 0006); bootstrap against a direct Postgres until B4 `Postgres` lands. Each is rewired as the dependency arrives (phased posture mirroring ADR 0012).

## 8. Cross-Cutting Deliverable Checklist

| Deliverable (§14.1) | Status |
|---|---|
| Helm values / manifests in Git | Applicable — pinned LiteLLM chart, GitOps non-secret config |
| Per-product docs (10.5) | Applicable |
| Operator runbook (10.7) | Applicable — fail-closed behaviour, key-state recovery, provider-failover, backing-store restore |
| Backup / restore | Applicable — backing Postgres (key/spend/config) restore is a runbook item; the store itself is provisioned via `Postgres` (B4) |
| Alert rules | Applicable — gateway down, backing-store unreachable, OPA-unreachable deny spike, provider-failover spike, budget-threshold spike |
| Grafana dashboard (Crossplane `GrafanaDashboard` XR) | Applicable |
| Headlamp plugin | Applicable — gateway/key/provider/budget visibility (redacted) |
| OPA / Rego integration | Applicable — A1 owns the pre-call + dynamic-registration decision **points**; policy content is B3/B16 |
| Audit emission (ADR 0034) | Applicable — durable-async via A18 adapter; never direct-write |
| Knative trigger flow | Applicable — `platform.observability.*` budget-exceeded→email flow; `platform.gateway.*`/`platform.policy.*` emit |
| HolmesGPT toolset | Applicable — routing, redacted key state, provider health, budget posture |
| 3-layer tests (Chainsaw/Playwright/PyTest) | Applicable — Chainsaw for capability-CRD→gateway effect (with B13 fake), Playwright for Headlamp plugin, PyTest for callback/translation/budget logic |
| Tutorials & how-tos | Applicable |

## 9. Acceptance Criteria

- AC-A1-01 (REQ-A1-01): A clean ArgoCD sync installs LiteLLM at the pinned version; no drift exists between Git config and running config; no manual config mutation is required for steady state.
- AC-A1-02 (REQ-A1-02): No agent-facing API response or log line discloses provider/MCP/A2A credential material; secrets are sourced from ESO-synced secrets only.
- AC-A1-03 (REQ-A1-03): All five surfaces respond under `/v1/...`; a request to an unversioned path is not served as a stable contract.
- AC-A1-04 (REQ-A1-04): A virtual-key/capability mutation from the B13 identity succeeds; the identical mutation from any other identity (including a cluster-admin by hand against the admin API) is rejected.
- AC-A1-05 (REQ-A1-05): A call with a model outside the key's `allowedModels[]` is rejected; a call with an expired `ttl` key is rejected; a valid in-scope call succeeds.
- AC-A1-06 (REQ-A1-06): An OpenAI-compatible request addressed to an internal Platform Agent and to an approved external `A2APeer` is delivered via A2A; an `A2APeer` with `direction: internal` is unreachable from an external-only context.
- AC-A1-07 (REQ-A1-07): A tool call to a registered `MCPServer` is proxied, OPA-gated, recorded, and forwarded with the credential mode matching `authMode`; a call to an unregistered MCP server has no reachable path.
- AC-A1-08 (REQ-A1-08): A request for a `modelRef` with a failing primary provider fails over to a configured secondary; provider credentials never appear on the agent-facing response.
- AC-A1-09 (REQ-A1-09): With a registered B2 handler, pre-call, post-call, and on-failure hooks each fire exactly once per the corresponding event in a traced request.
- AC-A1-10 (REQ-A1-10): With OPA made unreachable, a pre-call decision denies (fail-closed) unless a configured break-glass policy applies; the deny is observable.
- AC-A1-11 (REQ-A1-11): With the A18 audit endpoint down, calls still complete and audit events are durably buffered by the adapter; no audit row is written by A1 directly to Postgres/S3/OpenSearch.
- AC-A1-12 (REQ-A1-12): Crossing a `BudgetPolicy` threshold emits a `platform.observability.*` event and applies the configured `thresholdAction` (block or require-approval) on the next call.
- AC-A1-13 (REQ-A1-13): Admitting an `Agent` with `exposes` causes an OPA-gated registration; a denied registration emits a `platform.policy.*` deny event and the surface is not registered.
- AC-A1-14 (REQ-A1-14): Every CloudEvent A1 emits has a `type` under exactly one of the ten namespaces with non-empty `specversion` + `schemaVersion`; no event uses a coined top-level namespace.
- AC-A1-15 (REQ-A1-15): A traced call produces correlated spans in Tempo and a generation in Langfuse sharing the same `trace_id`, and gateway metrics appear in Mimir.
- AC-A1-16 (REQ-A1-16): A1 runs identically against a `Postgres`-provided connection secret on kind and on AWS; switching substrate changes no gateway code or config beyond the XR.
- AC-A1-17 (REQ-A1-17): Killing the gateway stops all agent model/tool/A2A traffic (fail-closed verified); HA replicas + PDB keep the surface available across a single-replica disruption.
- AC-A1-18 (REQ-A1-18): The HolmesGPT gateway toolset answers a routing/provider-health/budget-posture query against live gateway state with key material redacted.
- AC-A1-19 (REQ-A1-19): A simulated `/v1`→`/v2` surface change keeps `/v1` served for ≥1 platform release with a deprecation notice.
- AC-A1-20 (REQ-A1-20): Docs, runbook, alert rules, a rendering `GrafanaDashboard` XR, a loading Headlamp plugin, the OPA decision-point wiring, and passing three-layer tests all exist.

## 10. Risks & Open Questions

- R-A1-1 (high): A1 is the tier-0 fail-closed chokepoint — gateway loss halts all agent traffic. Mitigation: HA replicas + PDBs + readiness on backing store + runbook; the fail-closed trade is intentional (§6.1 line 201). Blast radius: whole platform.
- R-A1-2 (high): OPA-unreachable fail-closed could mass-deny if OPA (A7) is flaky during bootstrap. Mitigation: documented break-glass policy owned by B3/B16; alert on deny-rate spike; phased bootstrap. `[PROPOSED]` break-glass policy content not in A1's scope.
- R-A1-3 (med): exact LiteLLM callback signatures are not in Canon — A1/B2 contract risk. Mitigation: pin to upstream callback interface; version A1↔B2 contract; do not coin signatures (`[PROPOSED — not in source]`).
- R-A1-4 (med): B13 (only admin-API writer) is W1 — gateway state cannot be reconciled until B13 lands. Mitigation: eventually-consistent serving of last state (ADR 0006); bootstrap with a minimal seeded config; no hand-config drift permitted.
- R-A1-5 (med): `Postgres` (B4) is W1 while A1 is W0 — backing store may not be XR-provisioned at first install. Mitigation: bootstrap on direct Postgres, rewire onto the XR; uniform connection-secret shape makes the swap config-only.
- R-A1-6 (med): EgressTarget enforcement is shared between A1 (gateway) and A6 (Envoy) — risk of a gap where one path is governed and the other is not. Mitigation: A1 governs model/tool/A2A egress; A6 governs general egress; B22 threat model must confirm no uncovered egress path. Open question: is any agent egress reachable that bypasses both A1 and A6? — resolve with A6/B22.
- R-A1-7 (low): per-event-type `platform.gateway.*` names are deferred to B12; A1 must not coin them. `[PROPOSED]` flag carried until B12 registers them.
- R-A1-8 (low): `platform.security.*` gateway signals (repeated authn failures, policy-bypass attempts) are mapped by namespace only; concrete event types deferred to B12. `[PROPOSED — not in source]`.
- Open question (low): does the skill-gateway admin surface need its own OPA decision point distinct from MCP/A2A registration? Not specified in source; resolve with B16 once `Skill` flows are exercised.

## 11. References

- architecture-overview.md §6.1 (The Gateway Layer (LiteLLM), lines 170–225), §6.7 (Eventing, lines 553–595, broker failure/emit behaviour), §6.8 (Capability Registries & Approved Primitives, lines 604–616), §6.9 (Multi-Tenancy & Namespacing, 672+), §6.5 (Observability), §6.13 (Versioning), §14.1 (Workstream A deliverables).
- ADR 0006 (Python kopf operator for LiteLLM reconciliation — B13 sole admin writer); ADR 0001 (ARK + external enforcement perimeters); ADR 0013 (capability CRD model, via §6.8); ADR 0031 (CloudEvent taxonomy); ADR 0034 (audit pipeline — durable adapter); ADR 0015 (Tempo + Langfuse correlation by trace_id); ADR 0044 (substrate abstraction — `Postgres`); ADR 0030 (CRD/API versioning); ADR 0029 (Keycloak JWT claim schema); ADR 0002 (OPA/Gatekeeper).
- interface-contract §1.4 (B13 capability CRDs), §1.6 (`Postgres`), §2 (CloudEvent taxonomy), §3.1 (Platform SDK), §3.3 (HTTP `/v1` versioning), §4 (connection-secret contract), §5 (audit adapter).
- glossary (LiteLLM, A2A, MCP, CapabilitySet, Platform JWT, ESO, capability CRDs). Related pieces: B13, B2, B3/B16, A18, A7, A6, B4, B6, A17, A2, A13.
