# SPEC A20 — Policy simulator service

> kind: COMPONENT · workstream: A · tier: T1
> upstream: [A7, A1, A6] · downstream: [A22] · adrs: [0002, 0003, 0012, 0017, 0018, 0027, 0030, 0031, 0034, 0038, 0039] · views: [6.6, 6.10]
> canon-glossary: 6aadcc2a4f38 · canon-interface: 54f5ede58e5f

## 1. Purpose & Problem Statement

A20 is the platform's **policy simulator service**: given a synthetic request — subject (Keycloak claims), action, resource, context — it returns the decision at **every applicable enforcement layer** with the matching rule cited, **without performing the action**. The platform stacks many enforcement layers on every privileged action (Gatekeeper admission, LiteLLM OPA callbacks, Envoy egress OPA, Kubernetes RBAC floor, approval-system OPA elevation, `agent-platform` CLI promotion gates), and the predictable failure mode is cross-layer interaction: a rule authored in one layer unexpectedly interacts with another, and audit logs record the eventual decision but not the counterfactual. A20 answers **"what would happen for this exact request?"** before an incident, not after (ADR 0038, ADR 0027).

A20 is a thin **aggregator service** that fans the synthetic request out to each layer's **structured dry-run** path and composes the per-layer verdicts into one trace. It is exposed two ways with identical semantics: a **Headlamp simulator panel** (interactive, the editing companion for OPA bundles, alongside A39/A22's editors) and a **HolmesGPT skill** (callable inside diagnostic flows so HolmesGPT can "simulate the request that just got denied and tell me why"). **RBAC is reported as a separately-attributed layer** alongside the OPA-driven layers (preserving the RBAC-floor / OPA-restrictor contract). Simulator runs **ARE audited** under `platform.policy.*` but **never enter the enforcement path** — no real admission, runtime, or egress decision, no state change.

## 2. Scope

### 2.1 In scope
- **Aggregator service** — accepts a synthetic request (subject/action/resource/context), fans out to each layer's dry-run, composes a single per-layer decision trace with cited rules and per-layer `allow` + `simulated: true`.
- **Per-layer dry-run invocation** for: **Kubernetes RBAC** (reported as its own layer), **Gatekeeper admission** (audit-mode evaluation), **LiteLLM OPA callback** (runtime tool/model/MCP/A2A/budget), **Envoy egress OPA** (ext-authz dry-run), **approval-system elevation** (resolved required level without creating an `Approval`), and the **`agent-platform` CLI promotion gates**.
- **Headlamp simulator panel** — the interactive surface; preview-before-commit companion to the A22/A39 editors.
- **HolmesGPT skill** — the same aggregator API exposed as a HolmesGPT-callable skill (one API, two consumers; a HolmesGPT explanation is reproducible in Headlamp).
- **Audit of simulator runs** under `platform.policy.*` via the A18 adapter.
- Standard §14.1 deliverables: Helm/manifests, per-product docs, runbook, alerts, Grafana dashboard (XR), Headlamp plugin (the panel), OPA/Rego integration (consumes dry-runs; admission of A20 itself), audit emission, Knative trigger-flow design, HolmesGPT toolset (the skill), 3-layer tests, tutorials/how-tos.

### 2.2 Out of scope (and where it lives instead)
- **Implementing each layer's dry-run mode** → owned by the **layer component itself** (a per-component deliverable per ADR 0038): B2/A1 (LiteLLM callbacks), A6/Envoy egress, B19 (approval elevation), A7/Gatekeeper (audit-mode), Headlamp action gates (A9/A22), `agent-platform` CLI (B9). A20 **consumes** these dry-run paths; it does not build them.
- **The OPA decision contract / `simulated` flag shape** → **B3** (framework). A20 composes B3's per-layer decision documents.
- **Headlamp framework + editor framework** → **A9** (framework) / **A22** (editor framework). A20 ships the *simulator panel* that integrates with A22's editors (A22 is the downstream consumer per CSV).
- **Formal verification / policy-space enumeration / cross-layer conflict analysis / drift detection** → explicitly **out of v1.0 scope** (ADR 0038); point queries only.
- **The security review** ADR 0027 requires → F4 / B22; the simulator does not replace it.
- **OPA engine / content** → A7 / B16; **audit subsystem** → A18.

## 3. Context & Dependencies

**Upstream consumed (HARD):**
- **A7 (OPA / Gatekeeper)** — the OPA engine and Gatekeeper audit-mode evaluation the simulator queries.
- **A1 (LiteLLM)** — the LiteLLM OPA-callback dry-run path (runtime tool/model/MCP/A2A/budget).
- **A6 (agent-sandbox + Envoy egress proxy)** — the Envoy egress ext-authz dry-run path.

**Effective dependencies (named, not all in CSV):**
- **B3** — the canonical OPA decision contract + `simulated` flag A20 composes. Hard contract dependency; `[PROPOSED — not in source]` build-order note (CSV lists A7/A1/A6).
- **B19-core (approval system)** — the elevation dry-run (resolve required level without creating an `Approval`).
- **B9 (`agent-platform` CLI)** — the promotion-gate dry-run.
- **A22** — the editor framework the panel plugs into (CSV downstream).
- **A18** — audit adapter for `platform.policy.*` run records. **A14 (HolmesGPT)** — hosts the skill.

**Downstream consumers:** **A22** (Headlamp graphical-editor framework — every editor integrates with the simulator for preview-before-commit). De-facto: A17's web-search/scrape bundle and A19's routing bundle are simulated before rollout; HolmesGPT uses the skill in diagnosis.

**ADRs honored:**
- **ADR 0038** — the simulator surface; aggregator that fans out to per-layer dry-runs; two consumers (Headlamp panel + HolmesGPT skill); RBAC reported as a separate layer; approval elevation reported without creating an `Approval`; runs audited under `platform.policy.*` but never enforcement; point-queries only.
- **ADR 0018** — RBAC-as-floor / OPA-as-restrictor preserved (RBAC as a distinct attributed layer, not collapsed into OPA).
- **ADR 0017** — approval-system elevation composition (resolved required level, no `Approval` created).
- **ADR 0002 / 0003** — OPA engine; Envoy egress ext-authz. **ADR 0027** — does not replace the security review; targets named attack patterns (OPA bypass, capability escape).
- **ADR 0039** — Headlamp policy editors; the panel sits alongside them. **ADR 0012** — HolmesGPT as a first-class agent hosting the skill.
- **ADR 0034 / 0031 / 0030** — audit via adapter; `platform.policy.*` taxonomy; versioning.

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
N/A — A20 defines no CRD/XRD. It is a stateless aggregator service plus a Headlamp panel and a HolmesGPT skill. `[PROPOSED — not in source]` if simulator presets were ever modeled as a CRD; none is required by source.

### 4.2 APIs / SDK surfaces
- **Aggregator API** — the single API both Headlamp and HolmesGPT consume (ADR 0038: "one API, two consumers"). Input: synthetic request — `subject` (Keycloak/Platform JWT claims), `action`, `resource`, `context`. Output: a **per-layer decision trace** — for each layer: `layer` (which enforcement point), `allow`, cited `reasons`/rule, and `simulated: true`. The `simulated` field is the only source-named output field; the rest of the field names are **`[PROPOSED — not in source]`** (A20 should bind to B3's decision-document shape).
- **Per-layer dry-run calls A20 makes** (source-fixed behavior): OPA-backed components evaluate `data.allow` against the supplied input bundle; Gatekeeper exposes its existing audit-mode evaluation; Envoy exposes an ext-authz dry-run path; the approval system evaluates elevation **without creating an `Approval`**; RBAC is evaluated and attributed as its own layer; the `agent-platform` CLI promotion gates honor the dry-run flag.
- **HolmesGPT skill** — wraps the aggregator API as a skill (managed via LiteLLM's skill gateway; the `Skill` CRD is owned by B13, referenced by HolmesGPT's CapabilitySet). A20 provides the skill artifact; method surface beyond the aggregator API is `[PROPOSED — not in source]`.
- HTTP API URL-path-versioned `/v1/...` (interface-contract §3.3).

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
- **Emits `platform.policy.*`** — every simulator run/dry-run decision is recorded under this namespace via the A18 adapter (§6.6: "Simulator runs ARE audited under `platform.policy.*`"), so the policy-edit history is itself observable — **but never enters the enforcement path**. The `platform.policy` schema is **owned by A7** (QN-03); A20 is a dependent emitter, not an owner, and audit emission is gated on the audit-adapter freeze-gate (D-05).
- **OPA decision-document format (D-03):** A20 is a **consumer** of the OPA decision-document format, which is **owned by B3** — A20 composes/renders B3's per-layer decision documents and does not own or define the format. (A7 runs the engine; A1/A6 are likewise consumers.)
- Consumes none directly (it is request/response-driven). Per-event-type schemas → **B12**; `[PROPOSED — not in source]` for concrete event names; A20 commits only to the namespace.

### 4.4 Data schemas / connection-secret contracts
N/A — A20 is **stateless**: no datastore, no connection secret. Its "data" is the composed per-layer decision trace (transient). It reuses B3's OPA input/decision documents. Any caching of dry-run results is `[PROPOSED — not in source]` and must remain side-effect-free.

## 5. OSS-vs-Custom Decision
- **Aggregator service** — **build-new** (custom Python; "all custom code is Python", backlog §6). No OSS tool composes the platform's specific layer set; it is a thin fan-out + compose. Rationale: ADR 0038 specifies a thin aggregator over existing per-layer dry-runs.
- **Per-layer dry-run paths** — **not built here**; A20 *consumes* OSS/native capabilities exposed by each layer (OPA `data.allow`, Gatekeeper audit-mode, Envoy ext-authz dry-run).
- **Headlamp panel** — **build-new** plugin on the A9/A22 framework.
- **HolmesGPT skill** — **build-new** skill artifact wrapping the aggregator API.
- No pinned versions in source → `[PROPOSED — not in source]` at implementation.

## 6. Functional Requirements
- **REQ-A20-01:** A20 MUST accept a synthetic request — `subject` (Keycloak claims), `action`, `resource`, `context` — and return the decision at **every applicable enforcement layer** with the matching rule cited.
- **REQ-A20-02:** A20 MUST evaluate and attribute, as **separate layers**: Kubernetes **RBAC**, **Gatekeeper admission** (audit-mode), **LiteLLM OPA callback**, **Envoy egress OPA**, **approval-system elevation**, and the **`agent-platform` CLI promotion gates**.
- **REQ-A20-03:** A20 MUST report **RBAC as a separately-attributed layer** (not collapsed into OPA), preserving the RBAC-floor / OPA-restrictor contract (ADR 0018).
- **REQ-A20-04:** For approval-bearing actions, A20 MUST report the **resolved required approval level** (post-OPA-elevation) **without creating an `Approval`** (ADR 0017).
- **REQ-A20-05:** Every per-layer verdict A20 composes MUST carry `simulated: true` and MUST produce **no enforcement-path side effect** (no state change, no real denial of a live request).
- **REQ-A20-06:** A20 MUST expose **one aggregator API** consumed by **both** the Headlamp panel and the HolmesGPT skill (a HolmesGPT explanation MUST be reproducible in Headlamp).
- **REQ-A20-07:** A20 MUST ship a **Headlamp simulator panel** that composes/pastes a request (subject + action + target) and shows the layer-by-layer decision and why.
- **REQ-A20-08:** A20 MUST ship a **HolmesGPT-callable skill** with the same semantics for use inside diagnostic flows.
- **REQ-A20-09:** Every simulator run MUST be **audited under `platform.policy.*`** via the A18 adapter; the run record MUST NOT trigger any enforcement-path effect.
- **REQ-A20-10:** A20 MUST consume each layer's **structured dry-run** path (it MUST NOT reimplement layer logic); where a layer's dry-run is absent, A20 MUST surface that layer as **unavailable** rather than fabricate a verdict.
- **REQ-A20-11:** A20 MUST be **stateless** and side-effect-free; repeated identical simulations MUST yield identical traces given unchanged policy.
- **REQ-A20-12:** A20 MUST limit scope to **point queries** ("what would happen for this case"); it MUST NOT claim formal verification, policy-space enumeration, conflict analysis, or drift detection (ADR 0038).

## 7. Non-Functional Requirements
- **Security:** The simulator must be **incapable of enforcement-path effects** (REQ-A20-05) — its dry-run calls must use each layer's non-enforcing path. It is a defense against named attack patterns (OPA bypass, capability escape; ADR 0027) but does **not** replace the security review (F4/B22). Access to the simulator is itself OPA-gated (who may simulate which subject's claims). Audit records are tamper-evident via A18.
- **Multi-tenancy (§6.9):** The `subject` carries Keycloak claims (`platform_tenants`/`platform_namespaces`/`platform_roles`/`tenant_roles`/`capability_set_refs`); simulating another tenant's request must itself be OPA-authorized. Traces respect tenant boundaries.
- **Observability (§6.5):** Grafana dashboard (XR) for simulation volume, per-layer availability, deny-reason distribution. Alerts on a layer's dry-run path being unavailable. Each run is a `platform.policy.*` audit record.
- **Scale:** stateless aggregator, horizontally scalable; latency bounded by the slowest per-layer dry-run; fan-out is parallelizable.
- **Versioning (ADR 0030):** aggregator HTTP API URL-path-versioned; the decision-trace shape binds to B3's contract version; the HolmesGPT skill is versioned via the `Skill` CRD (B13).

## 8. Cross-Cutting Deliverable Checklist
- Helm/manifests — **applicable** (aggregator Deployment, Headlamp panel plugin, skill artifact + `Skill` reference).
- Per-product docs (10.5) — **applicable** (how to read a per-layer trace; how to simulate a request).
- Runbook (10.7) — **applicable** (layer-dry-run-unavailable triage, panel/skill divergence check).
- Backup/restore — **N/A — stateless** (no datastore).
- Alerts — **applicable** (per-layer dry-run unavailability, aggregator error rate). `[PROPOSED]` thresholds.
- Grafana dashboard (Crossplane XR) — **applicable**.
- Headlamp plugin — **applicable (core deliverable — the simulator panel)**; integrates with A22/A39 editors for preview-before-commit.
- OPA/Rego integration — **applicable** (consumes every layer's dry-run; admission Rego for the A20 service itself).
- Audit emission (ADR 0034) — **applicable** (runs audited under `platform.policy.*` via A18; never enforcement).
- Knative trigger flow — **N/A (request/response service)** — A20 is invoked synchronously by Headlamp/HolmesGPT, not event-driven. `[PROPOSED — not in source]` no trigger flow required.
- HolmesGPT toolset — **applicable (core deliverable — the skill)**.
- 3-layer tests — **applicable** (PyTest: fan-out/compose + no-side-effect + statelessness; Playwright: Headlamp panel trace; Chainsaw: admission of the A20 service / dry-run wiring across layers using fakes for not-yet-landed layers).
- Tutorials & how-tos — **applicable** ("simulate a request before editing a policy"; "ask HolmesGPT why a request was denied").

## 9. Acceptance Criteria
- **AC-A20-01:** A synthetic request returns a trace with a verdict + cited rule for each available layer. (→ REQ-A20-01)
- **AC-A20-02:** The trace includes distinct entries for RBAC, Gatekeeper, LiteLLM OPA, Envoy egress, approval elevation, and CLI promotion gates. (→ REQ-A20-02)
- **AC-A20-03:** A request denied at the RBAC floor is attributed to the **RBAC** layer, not to OPA. (→ REQ-A20-03)
- **AC-A20-04:** Simulating an approval-bearing action reports the resolved required level and creates **no** `Approval` object. (→ REQ-A20-04)
- **AC-A20-05:** Every layer verdict in a trace carries `simulated: true`; a state-diff before/after a simulation shows no enforcement-path change (no real admission/runtime/egress decision). (→ REQ-A20-05)
- **AC-A20-06:** A request simulated via the HolmesGPT skill and via the Headlamp panel returns identical traces. (→ REQ-A20-06, -08)
- **AC-A20-07:** The Headlamp panel renders a layer-by-layer decision with reasons for a composed request. (→ REQ-A20-07)
- **AC-A20-08:** A simulator run produces a `platform.policy.*` audit record via the A18 adapter with no enforcement effect. (→ REQ-A20-09)
- **AC-A20-09:** With one layer's dry-run path stopped, the trace marks that layer **unavailable** and does not fabricate a verdict. (→ REQ-A20-10)
- **AC-A20-10:** Two identical simulations against unchanged policy return identical traces; A20 holds no persistent state. (→ REQ-A20-11)
- **AC-A20-11:** The service surfaces only point-query results and documents that verification/conflict-analysis/drift are out of scope. (→ REQ-A20-12)
- **AC-A20-12:** An unauthorized caller cannot simulate another tenant's subject (OPA-gated). (→ REQ-A20-03, NFR multi-tenancy)

## 10. Risks & Open Questions
- **R1 (high):** A20 depends on **every layer exposing a structured dry-run** (a per-component ADR 0038 deliverable). If a layer ships without one, A20's trace is incomplete. Mitigation: REQ-A20-10 surfaces unavailable layers honestly; A20 uses fakes for not-yet-landed layers in tests. Blast radius high (cross-component).
- **R2 (high):** The **decision-trace field shape** is `[PROPOSED — not in source]` beyond `simulated`; it MUST bind to B3's contract or the panel/skill/per-layer outputs diverge. Mitigation: bind to B3; freeze before A22 integrates.
- **R3 (med):** **No-enforcement-side-effect** guarantee (REQ-A20-05) is only as strong as each layer's dry-run honoring it; a buggy layer dry-run could mutate state. Mitigation: contract tests per layer; A20 cannot guarantee what it does not own.
- **R4 (med):** Simulating arbitrary subjects is a **privilege-escalation surface** if not gated (one could probe another tenant's policy). Mitigation: OPA-gate the simulator itself (AC-A20-12).
- **R5 (low):** Panel/skill **divergence** if the two consumers don't share the exact aggregator API. Mitigation: one API, reproducibility test (AC-A20-06).
- **OQ1:** Does A20 evaluate RBAC by querying the live Kubernetes `SubjectAccessReview`-style path in dry-run, or a mirrored claim model? `[PROPOSED]` mirror via §6.9 claim schema (ADR 0018 mirrors RBAC into claims).
- **OQ2:** Is the HolmesGPT skill delivered as a `Skill` CRD via B13 or a built-in HolmesGPT tool? `[PROPOSED]` `Skill` CRD referenced by HolmesGPT's CapabilitySet.
- **OQ3:** Does the CLI promotion-gate dry-run run in-cluster or require the CLI binary? `[PROPOSED]` an in-cluster dry-run endpoint exposed by B9.

## 11. References
- architecture-overview.md §6.6 (line 544; policy simulator — two surfaces, the reported layers incl. RBAC as separate, structured dry-run with `simulated: true`, runs audited under `platform.policy.*` but never enforcement; line 532 OPA decision points; line 511 audit emission points), §6.10 (HolmesGPT), §14.1 (line 1686; A20 deliverable incl. RBAC + Gatekeeper/CLI gates).
- ADR 0038 (policy simulators — aggregator, two consumers, RBAC separate, approval elevation without `Approval`, audited not enforcing, point-queries only). ADR 0018 (RBAC-floor/OPA-restrictor). ADR 0017 (approval system). ADR 0002/0003 (OPA / Envoy egress). ADR 0027 (threat-model attack patterns; not a replacement for security review). ADR 0039 (Headlamp policy editors). ADR 0012 (HolmesGPT). ADR 0034/0031/0030 (audit / taxonomy / versioning).
- Related pieces: A7 (OPA/Gatekeeper), A1 (LiteLLM callbacks), A6 (Envoy egress), B3 (decision contract), B19-core (approval elevation), B9 (CLI gates), A22 (editor framework — downstream), A18 (audit), A14 (HolmesGPT).
