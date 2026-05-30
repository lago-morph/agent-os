# SPEC ADR-0015 — Tempo + Langfuse correlated by trace_id [PROPOSED]

> kind: ADR · workstream: — · tier: T1
> upstream: [A13, A2] · downstream: [B6, B7, A1, A6, A5, A14, D1, D2] · adrs: [0015] · views: [6.5]
> canon-glossary: 6aadcc2a4f38 · canon-interface: 54f5ede58e5f

## 1. Purpose & Problem Statement
ADR 0015 is a settled decision: the platform runs **Tempo** for general OTel traces and **Langfuse** for LLM-grade traces, correlated by a shared `trace_id`. This SPEC states what honoring that decision obliges: the agent SDK and LiteLLM gateway MUST emit the same `trace_id` into both backends; a `trace_id` MUST exist at request entry and propagate across A2A/MCP/gateway hops; Grafana dashboards MUST deep-link Tempo↔Langfuse; HolmesGPT MUST consume both; and trace granularity MUST be gated by the ADR 0035 dynamic toggle (no always-on sample-and-drop). It does not re-argue Tempo+Langfuse vs Langfuse-only.

The problem the decision solves: a single backend cannot serve both general distributed tracing and LLM-grade prompt/cost/eval semantics; developers and HolmesGPT must move fluidly between "what the system did" and "what the model saw."

## 2. Scope
### 2.1 In scope
- The SDK + LiteLLM obligation to emit the same `trace_id` to Tempo and Langfuse — a load-bearing invariant, test-covered.
- The request-entry `trace_id` generation + propagation across A2A, MCP, and gateway hops.
- Grafana deep-link obligation (Tempo span → Langfuse, and reverse where applicable).
- HolmesGPT observability toolset consuming Tempo + Mimir + Loki + Langfuse.
- Trace granularity gated by the ADR 0035 toggle; off-mode has no span-creation cost.
- Non-unit test runs publishing via OTel along the Tempo/Mimir/Loki path with the same correlation (ADR 0011).

### 2.2 Out of scope (and where it lives instead)
- Tempo + Mimir install/operation — component **A13**.
- Langfuse install/operation — component **A2**.
- The dynamic toggle mechanism — ADR 0035 / its enforcing component.
- Dashboard provisioning (as `GrafanaDashboard` XRs) — components **D1/D2** / ADR 0021.
- promptfoo red-team / eval data definitions — evaluation components.

## 3. Context & Dependencies
Upstream consumed: **A13** Tempo+Mimir (general trace + metrics backend), **A2** Langfuse (LLM trace backend). Downstream consumers / conformers: **B6** Platform SDK + **B7** agent SDK (emit correlated spans), **A1** LiteLLM (emits to both), **A6** Envoy / **A5** ARK (general OTel spans), **A14** HolmesGPT (consumes both), **D1/D2** dashboards (deep links).

ADR decisions honored:
- **ADR 0015** (this) — two backends correlated by `trace_id`; SDK/LiteLLM emit invariant.
- **ADR 0011** — non-unit test runs publish via OTel on the Tempo/Mimir/Loki path; their traces share the correlation.
- **ADR 0012** — HolmesGPT consumes both backends via its observability toolset.
- **ADR 0035** — trace granularity is gated by the dynamic toggle; no always-on tracing.
- **ADR 0021** — dashboards delivered as namespaced `GrafanaDashboard` XRs.
- **ADR 0030** — SDK semantic versioning carries the correlation contract.

## 4. Interfaces & Contracts
### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
- `GrafanaDashboard` (XR, namespaced, ADR 0021) — `dashboardJson`, `folder`, `visibility`; carries the Tempo↔Langfuse deep-link panels.
- `LogLevel` (namespaced, ADR 0035) — the toggle surface gating trace granularity; owned by ADR 0035.

### 4.2 APIs / SDK surfaces
- Platform SDK (B6) OTel-emission surface generates a `trace_id` at request entry and propagates it; this is a load-bearing invariant, not a convention. Method signatures beyond the named surface group: not specified in source — `[PROPOSED — not in source]` if detailed.
- LiteLLM gateway emits the same `trace_id` into spans sent to both backends.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
- Threshold/alert routing under `platform.observability.*`. Trace data itself rides OTel, not the CloudEvent bus. No new top-level namespace introduced.

### 4.4 Data schemas / connection-secret contracts
- N/A — no substrate primitive introduced; Tempo/Mimir/Langfuse backends own their own storage. `trace_id` is the only cross-backend correlation key.

## 5. OSS-vs-Custom Decision
N/A — ADR. (Enforcement note: upstream **Tempo** + **Mimir** (A13) and **Langfuse** (A2) installed/configured, correlated via SDK-emitted `trace_id`; config/wrap, no fork. Rejected alternative — Langfuse-only as the system-wide tracing store — is not built.)

## 6. Functional Requirements
- REQ-ADR-0015-01: The agent SDK and LiteLLM gateway MUST emit the same `trace_id` into spans sent to both Tempo and Langfuse for any LLM-bearing request.
- REQ-ADR-0015-02: Any component originating a request MUST ensure a `trace_id` exists at request entry; the SDK MUST generate one by default and propagate it across A2A, MCP, and gateway hops.
- REQ-ADR-0015-03: General OTel spans (Envoy, ARK, kopf operators, eventing, UIs) MUST go to Tempo; LLM-grade traces (prompts, completions, costs, evaluator scores, dataset runs) MUST go to Langfuse; non-LLM spans MUST NOT enter Langfuse.
- REQ-ADR-0015-04: Grafana dashboards MUST provide deep links from Tempo spans into Langfuse (and the reverse where applicable) without manual ID copy-paste.
- REQ-ADR-0015-05: HolmesGPT MUST consume both backends (Tempo + Mimir + Loki + Langfuse) via its observability toolset.
- REQ-ADR-0015-06: Trace granularity in both Tempo and Langfuse MUST be gated by the ADR 0035 dynamic toggle; v1.0 MUST NOT run always-on tracing with sample-and-drop — span creation MUST be conditional on live config so off-mode has no performance cost.
- REQ-ADR-0015-07: Non-unit test runs MUST publish run metrics/logs via OTel along the Tempo/Mimir/Loki path, and any emitted traces MUST share the same `trace_id` correlation (ADR 0011).

## 7. Non-Functional Requirements
- Observability (§6.5): Tempo/Langfuse cost MUST scale with verbosity (via the toggle), not with traffic.
- Security/tenancy: trace data respects the same namespace/RBAC visibility as other observability surfaces; dashboard `visibility` is RBAC + OPA controlled (ADR 0021).
- Versioning (ADR 0030): the SDK correlation contract is semantically versioned; a migration to a single backend later is a config swap because the SDK centralizes span emission.
- Resilience: a request entering without a `trace_id` breaks correlation — the SDK default generation MUST prevent this.

## 8. Cross-Cutting Deliverable Checklist
N/A — ADR (verification map in the PLAN). The §14.1 set is owned by enforcing components (A13, A2, B6, B7, A1, D1/D2).

## 9. Acceptance Criteria
- AC-ADR-0015-01: Honored when one LLM request produces a Tempo span and a Langfuse trace sharing an identical `trace_id`. (REQ-01)
- AC-ADR-0015-02: Honored when a request crossing A2A→MCP→gateway hops carries one `trace_id` end to end, generated at entry when absent. (REQ-02)
- AC-ADR-0015-03: Honored when a non-LLM Envoy/ARK span appears in Tempo and is absent from Langfuse, and an LLM trace appears in Langfuse. (REQ-03)
- AC-ADR-0015-04: Honored when a Tempo span in Grafana deep-links to the correlated Langfuse trace with no manual ID copy. (REQ-04)
- AC-ADR-0015-05: Honored when HolmesGPT retrieves correlated spans from both backends within one diagnostic session. (REQ-05)
- AC-ADR-0015-06: Honored when, with the toggle off, no spans are created (zero cost), and raising the toggle on one component produces spans only for it. (REQ-06)
- AC-ADR-0015-07: Honored when a non-unit test run emits OTel metrics/logs on the Tempo/Mimir/Loki path and its traces share the correlation. (REQ-07)

## 10. Risks & Open Questions
- R-1 (high): The SDK/LiteLLM `trace_id` emission is a single point whose failure silently breaks correlation; mitigated by mandatory test coverage (REQ-01) and default entry generation (REQ-02).
- R-2 (med): Two trace backends plus storage/retention/dashboards is a real ops cost accepted by the ADR; blast radius is operational, bounded by config swap reversibility.
- OQ-1 (low): Exact deep-link panel definitions live in D1/D2 dashboard specs; `[PROPOSED]` until those land.

## 11. References
- ADR 0015 (`adr/0015-tempo-langfuse-correlated-tracing.md`) — the decision enforced here.
- architecture-overview.md §6.5 (observability); architecture-backlog.md §2.4.
- Enforcing components: A13 (Tempo+Mimir), A2 (Langfuse), B6/B7 (SDKs), A1 (LiteLLM), A6/A5 (general spans), A14 (HolmesGPT), D1/D2 (dashboards).
- Related ADRs: 0011, 0012, 0021, 0030, 0035.
