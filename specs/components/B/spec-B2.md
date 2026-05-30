# SPEC B2 — LiteLLM custom callbacks

> kind: COMPONENT · workstream: B · tier: T1
> upstream: [A1, A7] · downstream: [] · adrs: [0006, 0034, 0031, 0035, 0018] · views: [6.1, 6.6, 6.5]
> canon-glossary: b0edae10a2e649ba06e2b184dc938235aab758e3 · canon-interface: 0ce201d5d5af5cffcf09b647ea4a902a47596d36

## 1. Purpose & Problem Statement

LiteLLM is the single chokepoint for every LLM, MCP, and A2A call (§6.1). Its fine-grained audit, RBAC, org-level guardrails, and advanced budget features are enterprise-only (§9). B2 fills those gaps with **custom Python callbacks** registered into LiteLLM's hook chain (the callbacks pipeline that fires at `pre-call`, `post-call`, `success`, `failure`, §6.1). The callbacks add: **PII scrubbing** (Presidio), **audit emission** through the platform audit adapter to the audit endpoint (ADR 0034 — never direct writes to OpenSearch), an **OPA runtime-decision bridge** (tool/model authorization, rate limits, budget enforcement, §6.6), and **content guardrails**.

Of these, **audit emission and the OPA bridge are mandatory** — they are the load-bearing enforcement and accountability hooks for the gateway; PII and guardrails are required-but-tunable by policy. The callbacks are configured in the LiteLLM ConfigMap and, because LiteLLM has no in-process reload, reload-on-change is a **rolling restart per ADR 0035** against the `replicas >= 2` Deployment (§6.1).

## 2. Scope

### 2.1 In scope
- Custom callback classes registered at `pre-call` / `post-call` / `success` / `failure` (§6.1) covering: PII scrub (Presidio), audit emission, OPA runtime-decision bridge, content guardrails.
- The OPA bridge: per-request consultation for tool/model authorization, rate limiting, content checks, and **budget enforcement** (§6.6 — "on each request, LiteLLM's OPA callback consults OPA").
- Audit emission of every request/response, including MCP calls and A2A handoffs (§6.6 — "A17 MCP services ride this path; no separate emission point").
- **Budget-exceeded CloudEvent emission** for the v1.0 budget-exceeded → email trigger flow (§6.7 — "a LiteLLM callback emits a CloudEvent when a virtual key exceeds its budget").
- Structured **dry-run mode** support so the policy simulator (A20) can drive the LiteLLM OPA callback layer (§6.6 — "components that implement runtime authorization … honor the dry-run flag").
- Callback ConfigMap wiring + Reloader-driven rolling restart (ADR 0035).
- **Placement-fallback rule:** B2 is designed to run as part of the LiteLLM gateway. If A1 (LiteLLM) has landed and its callback pipeline is available, B2 callbacks register into it directly. If A1 is not yet available at integration time, B2 provides a fallback harness (documented mock LiteLLM hook host) so the callback logic can be developed and tested independently and re-wired when A1 lands (per the §10 documentation-before-implementation mock-out commitment).

### 2.2 Out of scope (and where it lives instead)
- LiteLLM **install / Helm chart** — Component A1. B2's callbacks ship configured into A1's ConfigMap; the kopf operator is an A1 subchart (ADR 0006).
- The **OPA engine / Gatekeeper** install and the Rego content it evaluates — A7 (engine), B3/B16 (policy library + content). B2 only **calls** OPA and shapes the input/decision.
- **CRD reconciliation** into LiteLLM (`MCPServer`, `VirtualKey`, `BudgetPolicy`, etc.) — Component **B13** kopf operator (ADR 0006). B2 is a request-path callback, not a reconciler.
- The **audit endpoint + audit adapter library** itself — Component A18 (ADR 0034). B2 links the adapter; it does not implement it.
- The **email notification adapter** for the budget flow — the Knative notification adapter (§6.7); B2 only emits the CloudEvent.
- **CloudEvent schema authoring** — Component B12 registry. B2 emits under the committed namespaces only.
- SSO/auth proxy for the LiteLLM admin UI — Component **B1**.

## 3. Context & Dependencies

**Upstream consumed:**
- **A1 (LiteLLM)** — the callback pipeline host (`pre/post/success/failure` hooks, §6.1), the ConfigMap, the `replicas >= 2` Deployment, and the built-in per-virtual-key spend tracking the budget callback reads.
- **A7 (OPA / Gatekeeper)** — the runtime OPA decision endpoint the bridge consults; returns allow/deny with the §6.6 decision shape (`simulated: true` on dry-run).

**ADRs honored:**
- **ADR 0006** — the LiteLLM-facing reconciliation is kopf's (B13); B2 stays a callback, not a controller. Callbacks ship with LiteLLM (subchart lifecycle).
- **ADR 0034** — audit is mediated by the single platform audit adapter library; **no component writes audit directly** to Postgres/S3/OpenSearch. B2 emits via the adapter under `platform.audit.*`.
- **ADR 0031** — CloudEvents fall under exactly one committed namespace; budget events under `platform.observability.*`, gateway events under `platform.gateway.*`, policy decisions under `platform.policy.*`.
- **ADR 0035** — LiteLLM has no in-process reload; callback config changes roll via Reloader against `replicas >= 2`. Callbacks honor the `LogLevel` toggle for verbosity.
- **ADR 0018** — RBAC-as-floor / OPA-as-restrictor: the OPA bridge may only restrict, never grant.

**Downstream consumers:** none in piece-index.csv. Operationally: A17 MCP services ride the callback audit path; the budget-exceeded trigger flow (§6.7) consumes B2's emitted event.

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs
N/A — B2 owns no CRD. It **consumes** `BudgetPolicy` (`scope`, `period`, `limits`, `thresholdActions[]`) and `VirtualKey` (`ownerIdentity`, `budgetRef`, `allowedModels[]`) data as reconciled into LiteLLM/OPA by B13 (interface-contract §1.4); it does not reconcile them.

### 4.2 APIs / SDK surfaces
- **Registers** callback classes at the four LiteLLM hook points (`pre-call`, `post-call`, `success`, `failure`) — these are LiteLLM's hook names (§6.1), reproduced verbatim; the per-class Python signatures are LiteLLM's, **not platform Canon** — `[PROPOSED — not in source]` for any method names beyond the four hook points.
- **Consumes** the OPA runtime decision API: input = subject + action + target; output = the §6.6 decision shape incl. `simulated: true` under dry-run. `[PROPOSED — not in source]` for the concrete OPA input/decision field names (Canon states the shape exists and is uniform, not the field list).
- **Consumes** the platform **audit adapter library** (A18, ADR 0034) — a Python library link, not an HTTP call B2 owns.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
- **Emitted:**
  - Budget-exceeded notification → `platform.observability.*` (§6.7 budget-exceeded flow; threshold-crossing namespace). `[PROPOSED — not in source]` concrete event-type name (B12 registry).
  - Gateway routing/failover/MCP-health/A2A-handoff events → `platform.gateway.*` (§6.7).
  - OPA decision / violation surfaced from the callback → `platform.policy.*`.
  - Audit emissions → `platform.audit.*` (via adapter, ADR 0034).
- **Consumed:** none directly (B2 is request-path; the email adapter consumes the budget event).

### 4.4 Data schemas / connection-secret contracts
- Audit record shape is owned by the audit adapter (A18); B2 supplies structured fields, not a schema. Retention/redaction deferred to F1 (interface-contract §5).
- No connection-secret XRD owned. Presidio model/config is a vendored dependency, not a connection secret.

## 5. OSS-vs-Custom Decision

**Build-new (callbacks) on top of configured OSS.** Upstream: **LiteLLM** (A1) provides the hook chain; **Presidio** provides PII detection/scrub; **OPA** (A7) provides decisions; **audit adapter** (A18) provides emission. B2 writes the Python callback glue that wires them into the request path. Rationale (§9): fine-grained audit/RBAC, org-level guardrails, and advanced budget routing are enterprise-only in LiteLLM — "anything missing from OSS gets added via callback." No fork of LiteLLM. ADR linkage: ADR 0006 (callback, not controller), ADR 0034 (audit via adapter), ADR 0018 (OPA restricts).

## 6. Functional Requirements

- **REQ-B2-01:** B2 MUST register callback classes at the `pre-call`, `post-call`, `success`, and `failure` hook points of the LiteLLM callbacks pipeline (§6.1).
- **REQ-B2-02 (mandatory):** Every gateway request and response — including MCP tool calls and A2A handoffs — MUST emit a structured audit event via the platform audit adapter to the audit endpoint (ADR 0034); B2 MUST NOT write audit directly to Postgres/S3/OpenSearch.
- **REQ-B2-03 (mandatory):** On each request, the OPA bridge MUST consult OPA for tool/model authorization, rate limiting, content checks, and budget enforcement, and MUST enforce a deny by failing the request (§6.6).
- **REQ-B2-04:** The OPA bridge MUST only restrict access; it MUST NOT grant access beyond what RBAC/virtual-key claims already permit (ADR 0018).
- **REQ-B2-05:** A PII-scrub callback MUST run on request/response content per the configured Presidio policy; behavior is tunable by policy, not RBAC-gated off.
- **REQ-B2-06:** A content-guardrail callback MUST apply org-level content-policy checks per configured policy.
- **REQ-B2-07:** When a virtual key exceeds its budget, a callback MUST emit a CloudEvent under `platform.observability.*` carrying the metadata the email notification adapter needs (§6.7).
- **REQ-B2-08:** Each OPA-bridged callback MUST honor a structured **dry-run** input, returning the same decision shape with `simulated: true` and producing no enforcement-path side effect, so the policy simulator (A20) can compose it (§6.6).
- **REQ-B2-09:** Dry-run decisions MUST themselves be audited under `platform.policy.*` (§6.6 — "simulator runs ARE audited").
- **REQ-B2-10:** Callback config MUST live in the LiteLLM ConfigMap and roll via Reloader as a rolling restart against `replicas >= 2` (ADR 0035, §6.1).
- **REQ-B2-11:** Callbacks MUST honor the dynamic `LogLevel` toggle (ADR 0035) for verbosity, scoped per component/tenant/eventClass.
- **REQ-B2-12 (placement-fallback):** If A1's callback pipeline is available, B2 MUST register into it; if A1 is not yet available, B2 MUST be developable/testable against a documented mock LiteLLM hook host and re-wire to A1 with the same tests when A1 lands (§10 mock-out commitment).
- **REQ-B2-13:** Audit and OPA callbacks are mandatory and MUST be non-bypassable on the request path; PII and guardrail callbacks are required but their action is policy-tunable.

## 7. Non-Functional Requirements

- **Security:** audit and OPA hooks are non-bypassable (§6.6 defense-in-depth); PII scrub reduces secret-extraction-via-prompt-injection blast radius (§6.6 threat model). No direct audit writes (ADR 0034).
- **Multi-tenancy (§6.9):** OPA input carries the §6.9 claims (`platform_tenants`/`platform_namespaces`/`capability_set_refs`) so cross-tenant A2A authorization ("may caller in tenant A invoke callee in tenant B?", §6.9) is decided per request.
- **Observability (§6.5):** callbacks emit traces/metrics on the OTel path; Langfuse instrumentation runs in the same pipeline (§6.1). `trace_id` correlation per ADR 0015.
- **Scale:** callbacks run in-line on every request — latency budget must be bounded; OPA consultation cached/batched where the decision shape allows. `replicas >= 2`.
- **Versioning (ADR 0030):** callback config schema versioned per-component; OPA input/decision contract pinned to A7's.

## 8. Cross-Cutting Deliverable Checklist

- Helm/manifests — **applicable** (callback ConfigMap fragments shipped into A1's chart; Reloader annotations).
- Per-product docs (10.5) — **applicable** (callback catalog + config reference).
- Runbook (10.7) — **applicable** (callback failure / OPA-unreachable / audit-endpoint-down behavior).
- Alerts — **applicable** (callback error rate, OPA-deny spike, audit-emission failure, budget-event lag).
- Grafana dashboard (Crossplane XR) — **applicable** (callback latency, OPA decision mix, PII-hit rate) — `GrafanaDashboard` XR (ADR 0021).
- Headlamp plugin — N/A — request-path callbacks have no admin CRD; gateway admin UI is fronted by B1 and edited via B5/A22.
- OPA/Rego integration — **applicable** (B2 is the LiteLLM OPA decision point; consumes B16 content).
- Audit emission (ADR 0034) — **applicable** (core deliverable, mandatory).
- Knative trigger flow — **applicable** (budget-exceeded → email, a v1.0 flow; gateway events under `platform.gateway.*`).
- HolmesGPT toolset — **applicable** (query gateway deny/error patterns). `[PROPOSED — not in source]`.
- 3-layer tests — **applicable** (PyTest for callback logic incl. mock host; Playwright for "POST to LiteLLM, expect OPA decision, expect audit event" per §13; Chainsaw for ConfigMap/Reloader rollout).
- Tutorials & how-tos — **applicable** ("add a guardrail callback" how-to).

## 9. Acceptance Criteria

- **AC-B2-01** (REQ-B2-01): Callbacks are registered and observed firing at all four hook points. (PyTest)
- **AC-B2-02** (REQ-B2-02): A model call, an MCP call, and an A2A handoff each produce an audit event via the adapter; no direct OpenSearch/Postgres write occurs from B2. (Playwright/PyTest)
- **AC-B2-03** (REQ-B2-03): An OPA deny on tool/model/budget fails the request; an allow passes it. (Playwright)
- **AC-B2-04** (REQ-B2-04): A request RBAC/virtual-key would deny is not made allowable by any OPA outcome. (PyTest)
- **AC-B2-05** (REQ-B2-05): Configured PII entities are scrubbed from request/response content. (PyTest)
- **AC-B2-06** (REQ-B2-06): Content violating the configured guardrail policy is blocked/redacted. (PyTest)
- **AC-B2-07** (REQ-B2-07): Exceeding a virtual key's budget emits a `platform.observability.*` CloudEvent with the affected-user metadata. (Chainsaw — expect event on broker)
- **AC-B2-08** (REQ-B2-08): A dry-run request returns the decision shape with `simulated: true` and changes no state / denies no live request. (PyTest/Playwright)
- **AC-B2-09** (REQ-B2-09): A dry-run decision appears as a `platform.policy.*` audit record. (Chainsaw)
- **AC-B2-10** (REQ-B2-10): A callback ConfigMap change triggers a Reloader rolling restart against `replicas >= 2`. (Chainsaw)
- **AC-B2-11** (REQ-B2-11): A `LogLevel` toggle raises callback verbosity for the targeted scope and restores it on expiry. (Chainsaw/PyTest)
- **AC-B2-12** (REQ-B2-12): The callback suite passes against the mock hook host and, unchanged, against A1 when present. (PyTest)
- **AC-B2-13** (REQ-B2-13): Disabling/erroring a PII or guardrail callback degrades per policy, but audit and OPA callbacks cannot be bypassed on the request path. (PyTest)

## 10. Risks & Open Questions

- **OQ-B2-1** (high): Concrete OPA input/decision field names and the per-event-type budget/gateway event schemas are deferred (A20 dry-run shape, B12 registry). `[PROPOSED — not in source]`; reconciliation: bind to A7's decision shape and B12 names as they land.
- **R-B2-1** (high): In-line callbacks add per-request latency; an OPA outage could fail-closed on every call. Reconciliation note: define fail-open vs fail-closed per callback (audit/OPA mandatory → fail-closed for OPA deny semantics; document the OPA-unreachable posture in the runbook). Blast radius: every gateway call.
- **R-B2-2** (med): Placement-fallback drift — mock hook host diverging from real LiteLLM hook semantics. Mitigated by re-running the same suite against A1 (AC-B2-12).
- **OQ-B2-2** (low): Whether PII scrub applies to streamed chunks incrementally (streaming preserved end-to-end, §6.1) — design-time. `[PROPOSED — not in source]`.
- **R-B2-3** (low): LiteLLM hook signature changes across versions; pinned LiteLLM version (A1) bounds this.

## 11. References

- architecture-overview.md §6.1 Gateway architecture — callbacks pipeline / hook points / rolling restart (~184, ~214), failover (~224–231).
- architecture-overview.md §6.6 audit & OPA hook points (~lines under 436: LiteLLM callbacks emit every request/response ~512; OPA decision points ~533; dry-run mode ~549–551; budgets through OPA ~498–509).
- architecture-overview.md §6.7 budget-exceeded → email trigger flow (~626).
- architecture-overview.md §9 LiteLLM fill-in rows (audit/RBAC, guardrails, budgets; ~1404–1406).
- ADR 0006 (kopf vs callback split), ADR 0034 (audit adapter — no direct writes), ADR 0031 (CloudEvent taxonomy), ADR 0035 (rolling restart + LogLevel), ADR 0018 (RBAC floor / OPA restrictor).
- Interface-contract §1.4 (`BudgetPolicy`, `VirtualKey` consumed), §2 (`platform.observability.*`, `platform.gateway.*`, `platform.policy.*`, `platform.audit.*`), §5 (audit adapter).
- Related pieces: A1, A7, A18, B13, B16, A20, B12.
