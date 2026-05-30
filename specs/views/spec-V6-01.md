# SPEC V6-01 — Gateway architecture `[PROPOSED]`

> kind: VIEW · workstream: — · tier: T0
> upstream: [] · downstream: [] · adrs: [0006, 0013, 0020, 0034, 0035] · views: [6.1]
> canon-glossary: 555021949426 · canon-interface: 00b73bca4128

## 1. Purpose & Problem Statement

This view defines the **integration contract** for the gateway slice of the Agentic Execution Platform: the architectural slice in which **LiteLLM** is the single chokepoint through which every Platform Agent reaches LLM providers, MCP servers, and A2A peers. It is not a buildable component; it is realized by the LiteLLM install (A1), its custom callbacks (B2), the kopf operator that drives its registry (B13), and the SSO/auth proxy layer (B1).

The problem it solves: the platform needs exactly one enforcement perimeter for model/tool/agent traffic so that authentication (Keycloak claims + virtual keys), policy (OPA), audit (the adapter/endpoint), and observability (Langfuse) all fire in one place rather than being re-implemented per agent. This view fixes the invariants that every participating component must honor so the chokepoint property holds end-to-end and cannot be bypassed.

## 2. Scope
### 2.1 In scope
- The chokepoint invariant: all LLM, MCP, and A2A traffic transits LiteLLM (A1).
- The callbacks pipeline contract (B2): `pre-call` / `post-call` / `success` / `failure` hooks firing PII scrub (Presidio), OPA decisions, audit emission via the adapter, and Langfuse instrumentation.
- OpenAI-compatible ↔ A2A translation so frontends see Platform Agents as models.
- MCP brokering, A2A peering (internal + external, inbound disallowed for external self-onboarding), and dynamic agent registration gated by OPA.
- Virtual key issuance and registry reconciliation surface owned by B13.
- LLM provider failover semantics; MCP server health surfacing; end-to-end streaming.
- Operator packaging invariant (ADR 0006): kopf operator ships as a LiteLLM Helm subchart.

### 2.2 Out of scope (and where it lives instead)
- Capability-registry CRD model and CapabilitySet layering semantics — view V6-08 / ADR 0013 / ADR 0032.
- Audit pipeline topology (Postgres + S3 + OpenSearch) — view V6-05 / ADR 0034 / component A18.
- Security/policy enforcement model in full (defense-in-depth, budgets, simulator) — view V6-06.
- Memory and RAG call targets behind the gateway — views V6-03 / V6-04.
- Eventing fabric the gateway emits onto — view V6-07.
- Identity federation chain / JWT claim schema — views V6-11 (ADR 0028/0029).

## 3. Context & Dependencies

Realizing components and what each contributes to the slice:
- **A1 LiteLLM (gateway)** — the proxy itself: virtual-key auth, router + fallbacks, MCP gateway, A2A endpoint, registry, OpenAI-compat ↔ A2A translation.
- **B2 LiteLLM custom callbacks** — the Python hook chain that adds PII scrub, OPA runtime decisions, audit emission, Langfuse.
- **B13 Custom Python kopf operator** — reconciles `MCPServer`, `A2APeer`, `RAGStore`, `EgressTarget`, `Skill`, `CapabilitySet`, `VirtualKey`, `BudgetPolicy` into the LiteLLM registry/admin API (ADR 0013); ships as a LiteLLM subchart (ADR 0006).
- **B1 SSO/auth proxy layer** — fronts the gateway with oauth2-proxy and supplies Keycloak claims.

ADR decisions honored:
- **ADR 0006** — kopf operator packaged as a subchart of the LiteLLM Helm chart; single ArgoCD release; no gateway/operator version skew.
- **ADR 0013** — capability CRDs are the only way capabilities enter the gateway; nothing is configured directly on LiteLLM.
- **ADR 0020** — the fixed v1.0 initial MCP services set is brokered through LiteLLM.
- **ADR 0034** — gateway audit is emitted through the platform audit adapter to the audit endpoint; never direct writes.
- **ADR 0035** — LiteLLM has no in-process reload; config/policy changes are a rolling restart against `replicas >= 2`.

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
The gateway slice consumes (does not own) these B13-reconciled CRDs:
- `MCPServer` — `endpoint`, `authMode` (system/user-cred), `credentialsRef`, `tags`, `scopes`, `visibility`.
- `A2APeer` — `endpoint`, `direction` (internal/external), `auth`, `tags`.
- `RAGStore` — `backend`, `indexes[]`, `contentSourceRefs[]`, `ingestionPipelineRef`.
- `EgressTarget` — `fqdn`, `port`, `scheme`, `allowedMethods`.
- `Skill` — `gitRef`, `versionPin`, `schemaRef` (served via LiteLLM's skill gateway).
- `CapabilitySet` — `mcpServers[]`, `a2aPeers[]`, `ragStores[]`, `egressTargets[]`, `skills[]`, `llmProviders[]`, `opaPolicyRefs[]`.
- `VirtualKey` — `ownerIdentity`, `capabilitySetRef`, `budgetRef`, `environment`, `allowedModels[]`, `ttl`.
- `BudgetPolicy` — `scope`, `period`, `limits`, `thresholdActions[]`.

All namespaced; versioning per ADR 0030; owner of versioning lifecycle = B13.

### 4.2 APIs / SDK surfaces
- OpenAI-compatible HTTP ingress (frontends and agents) translated to A2A at the gateway.
- A2A endpoint (internal Platform-Agent-to-Platform-Agent and outbound to approved external `A2APeer`s).
- MCP gateway with tool namespacing and OAuth flows.
- LiteLLM admin API (the reconciliation target of B13) — HTTP, URL-path versioned (`/v1/...`) per interface-contract §3.3.
- Platform SDK model-invocation / `rag.*` / A2A-registration helpers terminate here (owned by B6; consumed, not defined, by this view).

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
Emitted by the gateway slice (per-event-type names deferred to B12 registry):
- `platform.gateway.*` — routing decisions, provider failover, MCP health changes, A2A handoffs.
- `platform.audit.*` — request/response, MCP calls, A2A handoffs (via the audit adapter).
- `platform.policy.*` — OPA decisions and dynamic-registration accept/deny from the callback path.
- `platform.capability.*` — capability-registry changes reconciled by B13 (`platform.capability.changed`, ADR 0013).
- `platform.observability.*` — budget-exceeded notification trigger (the v1.0 budget-exceeded → email flow originates as a LiteLLM callback CloudEvent).

### 4.4 Data schemas / connection-secret contracts
- MCP/A2A/RAG registry is internal LiteLLM state, populated only by B13 reconciliation — `[PROPOSED — not in source]` for any field beyond the CRD-stated set.
- Secret material for MCP services is injected via ESO (`credentialsRef` on `MCPServer`); the gateway never holds long-lived provider secrets in CRD spec.

## 5. OSS-vs-Custom Decision
N/A — VIEW (the OSS-vs-custom decision for LiteLLM, callbacks, and the kopf operator lives in the realizing component SPECs A1, B2, B13).

## 6. Functional Requirements
Each requirement is an **invariant/constraint this view imposes** on every participating component.

- **REQ-V6-01-01:** All LLM, MCP, and A2A traffic from a Platform Agent MUST transit LiteLLM (A1); no component may offer an alternate path to a model/tool/peer that bypasses the gateway. (§6.1, line ~172)
- **REQ-V6-01-02:** The callbacks pipeline (B2) MUST fire PII scrub (Presidio), OPA runtime decision, audit emission via the platform audit adapter, and Langfuse instrumentation at the LiteLLM hook points; audit MUST go through the adapter/endpoint, never direct writes (ADR 0034). (§6.1, line ~214)
- **REQ-V6-01-03:** Capabilities MUST enter the gateway only via B13 reconciliation of capability CRDs; nothing is configured directly on LiteLLM (ADR 0013). (§6.1, line ~216; §6.8)
- **REQ-V6-01-04:** The gateway MUST translate OpenAI-compatible HTTP ↔ A2A so frontends see Platform Agents as models, preserving stream chunks end-to-end. (§6.1, lines ~172, ~220)
- **REQ-V6-01-05:** External organizations MUST NOT self-onboard A2A peers; outbound external A2A is reachable only via an admin-defined `A2APeer` CRD reconciled by B13. Same registry, auth, and audit for all directions. (§6.1, line ~216)
- **REQ-V6-01-06:** Dynamic agent registration of an exposed A2A/MCP interface MUST be gated by OPA against the agent's namespace; deregistration occurs on agent termination. (§6.1, line ~218)
- **REQ-V6-01-07:** LLM provider failover MUST follow the declared-order semantics: single available provider used directly; single unavailable returns error; multi all-unavailable returns error; multi with ≥1 available fails over silently. Zero providers is rejected at admission. (§6.1, lines ~224–231)
- **REQ-V6-01-08:** MCP server health MUST surface to the agent as a connection failure surfaces, and emit observability signals; no synthetic health-check layer beyond MCP-native signals. (§6.1, line ~222)
- **REQ-V6-01-09:** Config/policy changes to LiteLLM MUST be applied by rolling restart against `replicas >= 2` (Reloader-triggered), because LiteLLM has no in-process reload (ADR 0035). (§6.1, line ~214; §6.5, line ~428)
- **REQ-V6-01-10:** The kopf operator MUST ship as a subchart of the LiteLLM Helm chart, as a single ArgoCD release, eliminating gateway/operator version skew (ADR 0006). (§6.1, line ~245)
- **REQ-V6-01-11:** The fixed v1.0 initial MCP services set (GitHub, Google Drive, Context7, OpenSearch, Postgres, MongoDB, web search + web scrape) MUST be brokered through the gateway with secret-plumbing, OPA admission, CapabilitySet wiring, audit tagging, and Headlamp inspection wired end-to-end. Firecrawl is NOT in the set. (§6.1, lines ~233–243; ADR 0020)

## 7. Non-Functional Requirements
- **Security/multi-tenancy:** virtual-key auth carries Keycloak claims; per-agent capability scope is enforced at the gateway via virtual-key claims + OPA; tenant isolation via namespace-scoped capability resolution.
- **Observability (§6.5):** every request/response emits Langfuse traces correlated by `trace_id` and audit via the adapter; gateway emits `platform.gateway.*` events.
- **Scale:** `replicas >= 2` with readiness probe gating Service traffic; `preStop: sleep 10-15s`; `terminationGracePeriodSeconds` 60–300s so streaming completes across rollouts (ADR 0035).
- **Versioning (ADR 0030):** gateway HTTP/admin APIs URL-path versioned; capability CRD versioning owned per-component by B13.

## 8. Cross-Cutting Deliverable Checklist
N/A — VIEW (cross-cutting deliverables are owned by realizing components A1, B1, B2, B13).

## 9. Acceptance Criteria
The view holds when:
- **AC-V6-01-01:** A traffic audit shows zero Platform-Agent LLM/MCP/A2A paths that do not transit LiteLLM. (→ REQ-01)
- **AC-V6-01-02:** For a sampled request, Langfuse trace, an audit record at the endpoint, and an OPA decision all exist for the same `trace_id`, produced via the callback chain (not direct writes). (→ REQ-02)
- **AC-V6-01-03:** Adding a capability without a corresponding CRD reconcile leaves it unreachable from any agent. (→ REQ-03)
- **AC-V6-01-04:** A LibreChat client invokes a Platform Agent as a "model" and receives streamed chunks end-to-end. (→ REQ-04, REQ-04 streaming)
- **AC-V6-01-05:** An external party cannot register an A2A peer; an admin-created `A2APeer` CRD makes the outbound peer reachable with audit. (→ REQ-05)
- **AC-V6-01-06:** An agent exposing an A2A/MCP interface registers only after an OPA allow; on termination its registry entry is removed. (→ REQ-06)
- **AC-V6-01-07:** Each failover row in REQ-07 is reproduced by test; zero-provider CapabilitySet is rejected at admission. (→ REQ-07)
- **AC-V6-01-08:** Killing an MCP backend surfaces failure to the agent and emits a `platform.gateway.*` health event with no synthetic prober present. (→ REQ-08)
- **AC-V6-01-09:** A ConfigMap change triggers a rolling restart with no dropped in-flight stream. (→ REQ-09)
- **AC-V6-01-10:** `helm template` shows the kopf operator as a subchart; ArgoCD shows one release. (→ REQ-10)
- **AC-V6-01-11:** All eight initial MCP service classes are reachable through the gateway with audit tags; Firecrawl is absent. (→ REQ-11)

## 10. Risks & Open Questions
- **OQ-1 (med):** Internal registry field set beyond CRD-stated fields is `[PROPOSED — not in source]`; resolved in A1/B13 component SPECs.
- **OQ-2 (low):** Some LiteLLM budget features may be enterprise-only (§6.6); if so, extended via callbacks (B2). Does not change this view's invariants.
- **R-1 (med):** Rolling-restart on every policy change (no in-process reload) can churn the gateway under rapid OPA-bundle edits; mitigated by ADR 0035 deployment shape and by editing budgets via OPA-data (not LiteLLM config) where possible.
- **R-2 (low):** Dynamic registration gated only by OPA — a permissive policy could over-expose interfaces; mitigated by namespace-scoped OPA checks (REQ-06).

## 11. References
- architecture-overview.md §6.1 Gateway architecture (lines ~170–245); §6.5 (line ~428, rolling-restart shape); §6.6 (budgets, lines ~497–509); §6.8 (capability registries, lines ~632+).
- ADRs: 0006 (kopf subchart), 0013 (capability CRD model), 0020 (initial MCP set), 0034 (audit pipeline), 0035 (log-level/rolling-restart).
- Realizing components: A1 (LiteLLM), B2 (callbacks), B13 (kopf operator), B1 (SSO/auth proxy).
- Related views: V6-05, V6-06, V6-07, V6-08, V6-11.
