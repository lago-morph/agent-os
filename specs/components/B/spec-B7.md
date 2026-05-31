# SPEC B7 — Initial agent SDK support (LangGraph + Langchain Deep Agents)

> kind: COMPONENT · workstream: B · tier: T0
> upstream: [B6, A6, A5] · downstream: [B18, B21] · adrs: [0019, 0001, 0013, 0032, 0003, 0030] · views: [6.2, 6.6, 6.13]
> canon-glossary: b0edae10a2e649ba06e2b184dc938235aab758e3 · canon-interface: 0ce201d5d5af5cffcf09b647ea4a902a47596d36

## 1. Purpose & Problem Statement

B7 is the **agent SDK** — the agent-authoring harness for Platform Agents and the **agent base
container image** that wraps the platform SDK (B6) and binds it into the two v1.0-supported agent
SDKs: **LangGraph** (the supported lower-level graph runtime, `Agent.sdk` value `langgraph`) and
**Langchain Deep Agents** (the opinionated batteries-included default built on LangGraph, `Agent.sdk`
value `deep-agents`). This is the decision frozen by ADR 0019 and stated in §6.2.

B7 exists so agent authors get one supported, tested harness that runs inside Sandboxes (A6) under
ARK (A5), reaches the platform only through the documented interaction surfaces — Kubernetes CRDs,
the LiteLLM call path via B6, and the Envoy egress proxy — and gets the platform's enforcement
perimeter (LiteLLM, OPA, Envoy) for free regardless of harness internals. It is **distinct from the
platform SDK (B6)**: B6 is *what agents call into the platform with*; B7 is *the harness that hosts
agent code, binds B6 into the two SDK idioms, and ships as the base image ARK runs*. B7 is T0
foundation because B18 (recommended compositions) and B21 (agent dev environment) build on it and the
whole tutorial/onboarding chain (B6→B7→CI→deploy) targets Deep Agents.

## 2. Scope

### 2.1 In scope
- The **agent base container image** (multi-SDK harness) wrapping B6 and bundling **LangGraph** and
  **Langchain Deep Agents** for v1.0.
- The **`Agent.sdk` field contract**: accepting `langgraph` and `deep-agents` as the two v1.0 values
  (Canon), with the harness selecting the runtime accordingly.
- **Binding B6's surfaces** (`memory.*`, `rag.*`, OTel emission, A2A registration) into both
  LangGraph primitives and Deep Agents idioms behind B6's single API contract (ADR 0019).
- Wiring the harness to the **documented interaction surfaces** (§6.2): Kubernetes CRDs (`Agent`,
  `CapabilitySet`, `Memory`, `MCPServer`, `EgressTarget`, `Skill`), the LiteLLM call path for
  LLM/MCP/A2A, and the Envoy egress proxy (A6, ADR 0003) for non-LiteLLM outbound HTTP.
- **Capability-change handling** (ADR 0013) surfaced through B6's callback + `refresh_capabilities()`
  in both SDKs.
- The **multi-SDK harness shape** preserved so OpenAI Agents SDK / Strands / Anthropic Agent SDK /
  Mastra / ARK ADK enroll later additively, by adding allowed `sdk` values, **without a schema break**.
- The **agent base image build pipeline matrix dimension** (LangGraph + Deep Agents variant), built
  and scanned, structured so a second variant family is a config change (§8, §9.5).
- **Documentation of the harness contract** for adventurous BYO-image users (the surfaces above) —
  while **third-party harness images remain NOT officially supported** (ADR 0019).

### 2.2 Out of scope (and where it lives instead)
- **Platform SDK surfaces / signatures** (`memory.*` etc.) → **B6** (B7 binds, does not define).
- **ARK operator + `Agent`/`AgentRun` reconciliation** → **A5**; B7 does not own the CRD or reconciler.
- **Sandbox runtime (gVisor/Kata), warm pools, skill init container/volume** → **A6** / agent-sandbox.
- **Envoy egress proxy itself** → **A6 / ADR 0003**; B7 only routes outbound HTTP through it.
- **Enrolling additional SDKs** (OpenAI Agents SDK, Strands, etc.) → future per ADR 0019 / backlog
  §3.15; B7 only preserves the harness shape.
- **Agent profile library / recommended compositions** → **B17 / B18** (consume B7).
- **Officially supporting third-party harness images** → explicitly out for v1.0 (ADR 0019).
- **Agent local dev environment** → **B21**.

## 3. Context & Dependencies

**Upstream consumed:**
- **B6 (Platform SDK)** — the harness wraps B6; the two SDK idioms bind B6's `memory.*`, `rag.*`,
  OTel, A2A surfaces and the capability-change client.
- **A6 (agent-sandbox + Envoy egress proxy)** — the harness runs as pods inside `Sandbox`es and
  routes non-LiteLLM outbound HTTP through the Envoy egress proxy (allowlisted `EgressTarget`s).
- **A5 (ARK)** — ARK reconciles `Agent` CRDs (incl. the `sdk` field) into pods running the base image;
  SDK choice is harness-level and does not constrain ARK (ADR 0001).

**Downstream consumers:**
- **B18** (recommended agent compositions) — target Deep Agents end-to-end.
- **B21** (agent dev environment) — ships the harness for local development.
- (per CSV downstream) feeds B17 usage via B18; tutorials/how-tos target Deep Agents.

**ADRs honored:**
- **ADR 0019** — LangGraph supported + Deep Agents default; `Agent.sdk` accepts `langgraph` and
  `deep-agents` in v1.0, growing by enrollment without schema migration; third-party harness images
  unsupported but the contract is documented; bindings stay behind one B6 contract.
- **ADR 0001** — ARK reconciles `Agent` regardless of in-harness SDK; B7 imposes no operator changes.
- **ADR 0013 / 0032** — capability-change notification + CapabilitySet overlays compose identically
  across both SDKs, surfaced through B6's callback.
- **ADR 0003** — outbound HTTP not via LiteLLM goes through the Envoy egress proxy (allowlist).
- **ADR 0030 / §6.13** — the per-SDK compatibility matrix starts with two columns and grows by column;
  the base image build matrix has the variant dimension for additive enrollment.

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
B7 **defines no CRD**; it **consumes** the `Agent` CRD (owner: A5) and is bound by its **`sdk`** field
(Canon). The `sdk` field accepting **`langgraph`** and **`deep-agents`** in v1.0 is Canon
(interface-contract §1.2, §3.2; ADR 0019). B7 also consumes `CapabilitySet`, `Memory`, `MCPServer`,
`EgressTarget`, `Skill`, `Sandbox`/`SandboxTemplate` as read context. No new CRD fields are
introduced; enrolling future SDK values is an **additive enum extension** owned by A5's `Agent`
versioning, not a B7 schema. `[PROPOSED — not in source]` that the `Agent.sdk` enum is validated by
admission against the allowed-value set — flagged because source states the field grows "without a
schema break" but does not name the validation mechanism.

### 4.2 APIs / SDK surfaces
- **Harness ↔ platform contract** (§6.2, Canon surfaces): (a) Kubernetes CRDs above; (b) the LiteLLM
  call path for LLM/MCP/A2A via B6; (c) the Envoy egress proxy for other outbound HTTP. B7 binds, it
  does not redefine these.
- **B6 binding surface:** B7 re-exposes B6's `memory.*`, `rag.*`, OTel, A2A helpers through **LangGraph
  primitives** and **Deep Agents idioms** behind B6's one API contract (ADR 0019). Specific binding
  symbol names are `[PROPOSED — not in source]` (source: "exposed through both LangGraph primitives and
  Deep Agents idioms" — idiom names not enumerated).
- **`Agent.sdk` selection contract:** value `langgraph` → LangGraph runtime; `deep-agents` → Deep
  Agents runtime (both Canon).
- **Exposed A2A/MCP interface versioning:** agents authored on B7 declare interface versions of form
  `myAgent.v1` (§6.13); the *shipping* component (B17/B18) owns the version, not B7.
- **BYO-harness contract doc:** the documented interface (CRD shapes + LiteLLM path + Envoy + B6 API)
  that unsupported third-party images must comply with (ADR 0019).

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
- **Consumed (via B6):** `platform.capability.changed` (Canon, ADR 0013) — surfaced to agent code in
  both SDKs.
- **Lifecycle:** `AgentRun` lifecycle events fall under `platform.lifecycle.*` (Canon namespace) but
  are emitted by **ARK / the runtime**, not authored by B7. B7 mints **no** new event types; concrete
  schemas live in **B12**. `[PROPOSED — not in source]` that B7 emits any CloudEvent itself.

### 4.4 Data schemas / connection-secret contracts
N/A — B7 is a harness/base image; it holds no datastore and writes no connection secret. Memory/RAG
access is via B6; checkpoints/state persist in platform backends (Postgres via ARK/Letta), not owned
here.

## 5. OSS-vs-Custom Decision
**Wrap + config** of OSS: **LangGraph** and **Langchain Deep Agents** (built on LangGraph) are the
upstream SDKs; B7 **builds-new** the harness/base-image around them and **wraps** B6. No fork of
LangGraph/Deep Agents — the harness preserves the multi-SDK shape so further SDKs enroll additively
(ADR 0019). Exact LangGraph / Deep Agents pinned versions `[PROPOSED — not in source]` (tracked in the
per-SDK compatibility matrix, §9.5). Rationale (ADR 0019): supporting both costs little because Deep
Agents is already a LangGraph application — harness, B6 bindings, and integration surfaces are shared.

## 6. Functional Requirements
- **REQ-B7-01:** B7 MUST ship an **agent base container image** (multi-SDK harness) wrapping the
  platform SDK (B6) and bundling **LangGraph** and **Langchain Deep Agents** for v1.0.
- **REQ-B7-02:** The harness MUST select the runtime from the **`Agent.sdk`** field, supporting values
  **`langgraph`** and **`deep-agents`** (Canon).
- **REQ-B7-03:** B7 MUST expose B6's `memory.*`, `rag.*`, OTel, and A2A surfaces through **both
  LangGraph primitives and Deep Agents idioms** behind B6's single API contract (ADR 0019).
- **REQ-B7-04:** The harness MUST reach the platform only via the documented surfaces — Kubernetes CRDs,
  the LiteLLM call path (LLM/MCP/A2A) via B6, and the **Envoy egress proxy** for other outbound HTTP
  (ADR 0003) — and MUST NOT bypass them.
- **REQ-B7-05:** The harness MUST surface **capability-change notifications** (ADR 0013) via B6's
  callback + `refresh_capabilities()` in both SDKs.
- **REQ-B7-06:** The harness MUST run as pods inside `Sandbox`es reconciled via ARK (A5/A6) without
  requiring operator changes for the SDK choice (ADR 0001).
- **REQ-B7-07:** The harness shape MUST be **preserved for additive enrollment** so a new `Agent.sdk`
  value can be added without an `Agent` schema break (ADR 0019, backlog §3.15).
- **REQ-B7-08:** The **agent base image build pipeline** MUST build and scan the LangGraph + Deep Agents
  variant and carry the **variant-matrix dimension** so a second variant family is a config change
  (§8, §9.5).
- **REQ-B7-09:** B7 MUST **document the harness contract** (CRD shapes + LiteLLM path + Envoy + B6 API)
  for BYO-image users while marking **third-party harness images NOT officially supported** (ADR 0019).
- **REQ-B7-10:** The **per-SDK compatibility matrix** MUST start with two columns (LangGraph, Deep
  Agents) and grow by column on enrollment (§9.5, ADR 0030).

## 7. Non-Functional Requirements
- **Security:** all enforcement is external (LiteLLM/OPA/Envoy); the harness MUST NOT carry an escape
  hatch around the egress proxy or the LiteLLM path; base image is scanned in the build pipeline (§8).
- **Multi-tenancy (§6.9):** the harness inherits the agent's namespace/tenant + CapabilitySet scope;
  it MUST NOT broaden reach beyond the resolved capability set.
- **Observability (§6.5):** agent spans emit via B6's OTel surface with `trace_id` correlation
  (ADR 0015); both SDK variants emit comparably.
- **Scale:** the harness must run in agent-sandbox warm pools without per-start heavy initialization
  beyond skill-volume mount; capability-change subscription degrades to poll without crashing.
- **Versioning (ADR 0030):** the base image and bundled SDK versions track the compatibility matrix
  (§9.5); enrolling an SDK is additive (new column + allowed `sdk` value), never a break.

## 8. Cross-Cutting Deliverable Checklist
- Helm/manifests — **applicable (partial)** — base image + any harness manifests; ARK/sandbox install
  is A5/A6. `[PROPOSED — not in source]` for harness-specific manifests beyond the image.
- Per-product docs (10.5) — **applicable** (LangGraph + Deep Agents reference, BYO-harness contract,
  compatibility matrix).
- Runbook (10.7) — **applicable** (image/SDK version-mismatch, egress-route failure diagnosis).
- Alerts — **N/A** — runtime alerts are agent/gateway/sandbox-level (A1/A6), not harness-level.
- Grafana dashboard (Crossplane XR) — **N/A** — agent/runtime dashboards cover harness-emitted traces.
- Headlamp plugin — **N/A** — no operator UI owned here.
- OPA/Rego integration — **applicable (consumer)** — enforcement is external; harness surfaces denials,
  authors no policy.
- Audit emission (ADR 0034) — **applicable (via B6)** — harness actions audit through B6/the adapter.
- Knative trigger flow — **applicable (consumer)** — consumes `platform.capability.changed` via B6;
  `AgentRun` lifecycle events are emitted by ARK, not B7.
- HolmesGPT toolset — **N/A** — B7 is agent-facing.
- 3-layer tests — **applicable** (PyTest harness/binding tests; Chainsaw for `Agent`-with-`sdk` →
  pod-from-base-image via ARK/sandbox; Playwright N/A — no UI).
- Tutorials & how-tos — **applicable** — the "build your first agent" chain targets Deep Agents
  end-to-end (ADR 0019, §10).

## 9. Acceptance Criteria
- **AC-B7-01:** A built agent base image contains both LangGraph and Deep Agents and the platform SDK
  (B6). (→ REQ-B7-01)
- **AC-B7-02:** An `Agent` with `sdk: langgraph` runs the LangGraph runtime; one with `sdk: deep-agents`
  runs Deep Agents. (→ REQ-B7-02)
- **AC-B7-03:** The same `memory.*`/`rag.*`/OTel/A2A program runs under both SDK bindings via B6's one
  contract. (→ REQ-B7-03)
- **AC-B7-04:** Outbound HTTP not via LiteLLM to a non-allowlisted FQDN is blocked by the Envoy egress
  proxy; an allowlisted `EgressTarget` succeeds. (→ REQ-B7-04)
- **AC-B7-05:** On `platform.capability.changed`, the agent receives the notification under both SDKs;
  `refresh_capabilities()` returns updated capabilities. (→ REQ-B7-05)
- **AC-B7-06:** ARK reconciles an `Agent` referencing the base image into a sandboxed pod with no
  operator change for either `sdk` value. (→ REQ-B7-06)
- **AC-B7-07:** A test confirms adding a new (mock) `sdk` value requires no `Agent` CRD schema change —
  only an allowed-value/config addition. (→ REQ-B7-07)
- **AC-B7-08:** The build pipeline builds + scans the LangGraph+Deep Agents variant and exposes a
  variant-matrix dimension. (→ REQ-B7-08)
- **AC-B7-09:** The BYO-harness contract doc enumerates the CRD/LiteLLM/Envoy/B6 surfaces and states
  third-party images are unsupported. (→ REQ-B7-09)
- **AC-B7-10:** The compatibility matrix ships with LangGraph and Deep Agents columns. (→ REQ-B7-10)

## 10. Risks & Open Questions
- **R1 (high):** B7 binds B6's `[PROPOSED]` signatures; if B6's contract shifts, both SDK bindings
  break. Mitigation: pin B6 via the compatibility matrix; B6 freezes its contract first. Blast radius:
  high (both SDK paths + tutorials).
- **R2 (med):** The LangGraph/Deep Agents idiom binding symbol names are `[PROPOSED — not in source]`;
  divergent idioms could leak SDK-specific surface and break portability (REQ-B7-03). Mitigation: keep
  bindings strictly behind B6's contract; test the same program on both.
- **R3 (med):** `Agent.sdk` enum-validation mechanism is `[PROPOSED]`; if A5 doesn't admission-validate
  the value, an unenrolled `sdk` reaches the harness. Mitigation: harness rejects unknown `sdk`;
  coordinate validation with A5.
- **R4 (med):** Pinned LangGraph/Deep Agents versions `[PROPOSED]`; upstream churn (LangChain ecosystem
  moves fast) risks matrix drift. Mitigation: matrix-driven pinning + build-pipeline scan.
- **R5 (low):** BYO third-party images, though unsupported, still run because enforcement is external;
  residual confusion risk mitigated by explicit "unsupported" doc framing (ADR 0019).
- **OQ1:** Does B7 ship harness-specific Helm manifests, or only the base image (ARK/sandbox own
  deployment)? `[PROPOSED]` image-only + a sample `Agent` manifest; deployment via A5/A6.
- **OQ2:** Where the BYO-harness contract doc lives (B7 per-product docs vs C-workstream reference).
  `[PROPOSED]` authored in B7 §10.5, surfaced in C4 reference.

## 11. References
- architecture-overview.md §6.2 (lines 282–292; primary SDK, `sdk` field accepts `langgraph`/
  `deep-agents`, harness interaction surfaces, BYO-harness unsupported), §6.13 (line 991; exposed A2A/
  MCP interface versioning; line 989 SDK matrix), §8 (agent base image build pipeline), §9.5 (per-SDK
  compatibility matrix).
- interface-contract.md §3.2 (Agent SDK; `langgraph`/`deep-agents` values; signatures not specified),
  §1.2 (`Agent.sdk`), §2 (`platform.capability.changed`).
- ADR 0019 (LangGraph supported + Deep Agents default), ADR 0001 (ARK reconciles `Agent`), ADR 0013 /
  0032 (capability change + overlays SDK-independent), ADR 0003 (Envoy egress), ADR 0030 (versioning).
- Related pieces: B6 (platform SDK), A5 (ARK), A6 (sandbox + Envoy), B17 (profiles), B18
  (compositions), B21 (dev env), B12 (event schemas).
