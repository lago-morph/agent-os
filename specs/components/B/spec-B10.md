# SPEC B10 — Coach Component (self-retrospection)

> kind: COMPONENT · workstream: B · tier: T1
> upstream: [B6, A5, A2, A14] · downstream: [] · adrs: [0012, 0017, 0019, 0022, 0030, 0031, 0034, 0038] · views: [6.10, 6.2, 6.5, 6.4]
> canon-glossary: b0edae10a2e649ba06e2b184dc938235aab758e3 · canon-interface: 0ce201d5d5af5cffcf09b647ea4a902a47596d36

## 1. Purpose & Problem Statement

B10 is the **Coach Agent** (Canon glossary; §6.10 references it as "Coach Component") — the
platform's self-**improvement** Platform Agent, the sibling of HolmesGPT's self-**management**
role. Where HolmesGPT diagnoses and remediates the running platform, the Coach Agent runs on a
schedule, **observes traces, audit data, and evaluation results**, and **proposes prompt/skill
changes** to Platform Agents. Its proposals land as **auto-PRs or Headlamp suggestion cards** —
it never mutates an agent's prompt or skill set directly; every change goes through the same
Git → CI → ArgoCD path (§7.4) and, where the action requires it, the generalized approval
system (§7.5, B19, whose initial use case is explicitly "Coach skill changes").

The problem it solves: a Platform Agent's prompts and `Skill` references drift from optimal as
models, capabilities, and workloads change, but nobody is systematically reviewing the
trace/eval corpus to spot regressions or improvement opportunities. The Coach Agent closes that
loop with a low-trust, proposal-only mechanism: all of its signal comes from already-collected
observability (Langfuse traces, audit trail, `Evaluation` results), and all of its output is a
reviewable change suggestion gated by human approval.

The Coach Agent is itself a Platform Agent declared as an `Agent` CRD, reconciled by ARK, run
in a `Sandbox`, calling out only through LiteLLM — administered like any other agent.

## 2. Scope

### 2.1 In scope
- The Coach Agent as a Platform Agent: its `Agent` CRD, its `CapabilitySet` (Langfuse/observability
  read access, Knowledge Base read access, the relevant LLM provider), and its scheduled trigger.
- **Retrospection logic**: consuming Langfuse traces (A2), the audit trail (via the audit
  endpoint / OpenSearch advisory index), and `Evaluation` results to identify candidate
  prompt/skill improvements.
- **Proposal emission**: generating auto-PRs (against the Git repo holding `Agent`/prompt/`Skill`
  definitions) and/or Headlamp **suggestion cards**, mirroring the HolmesGPT propose-via-PR pattern.
- Routing skill-change proposals through the **generalized approval system** (B19) where the
  action is gated `upon-approval`.
- Audit emission for every retrospection run and every proposal (ADR 0034).
- The Coach Agent's own observability (OTel spans, Langfuse traces of its own LLM calls).

### 2.2 Out of scope (and where it lives instead)
- **The approval CRD / queue / Argo Workflow integration** → **B19** (generalized approval system).
- **The Headlamp suggestion-card / approval-queue UI plugin** → **B5** (cross-cutting Headlamp
  plugins) and **B19-ui**; B10 emits the proposal payload, it does not own the panel.
- **The Platform SDK surfaces** the Coach uses (`memory.*`, `rag.*`, OTel emission, A2A helpers)
  → **B6**.
- **The agent harness / SDK** (LangGraph / Deep Agents) the Coach runs on → **B7** (ADR 0019).
- **HolmesGPT** self-management (remediation, diagnostics) → **A14** (§6.10). Coach is
  self-improvement, not self-management; the two are distinct Platform Agents.
- **The Knowledge Base RAGStore** content/indexing → A-side + C8; B10 only *includes* it in a
  CapabilitySet.
- **Langfuse / Tempo / Mimir install** → A2 / A13.

## 3. Context & Dependencies

**Upstream consumed (HARD):**
- **A2 (Langfuse)** — the trace/eval/cost corpus the Coach reads to find improvement candidates.
- **A5 (ARK)** — reconciles the Coach's `Agent` CRD and the scheduled `AgentRun`s.
- **A14 (HolmesGPT)** — named as upstream in `piece-index.csv`; B10 reuses HolmesGPT's
  **propose-via-PR / suggestion-card pattern** (§6.10) and may hand off analysis questions to
  HolmesGPT over A2A. The shared "propose, don't mutate" trust model originates with A14/ADR 0012.
- **B6 (Platform SDK)** — `memory.*`, `rag.*`, OTel emission, and A2A registration helpers used
  by the Coach's harness.

**Downstream consumers:** none in `piece-index.csv`. (The Coach's *proposals* feed Git/CI and,
where gated, B19 — but those are runtime interactions, not build-time downstream pieces.)

**Implementation ordering (§14.2 / B10 row):** "Implemented **after** the components it depends
on land: Langfuse, ARK, LiteLLM, observability stack. Doesn't necessarily need OPA; design-time
decision." → W3.

**ADRs honored:**
- **ADR 0012** — first-class Platform Agent pattern; **propose-via-PR / suggestion card**, read-mostly
  default, write actions gated by the three-state OPA model (allowed / upon-approval / denied).
- **ADR 0017** — generalized approval system; Coach skill changes are a named initial use case.
- **ADR 0019** — the Coach runs on a supported v1.0 SDK (`langgraph` or `deep-agents`).
- **ADR 0022** — Knowledge Base is a separate primitive the Coach *uses* via CapabilitySet.
- **ADR 0030** — the Coach's `Agent` CRD (and any exposed A2A interface) is versioned per policy.
- **ADR 0031** — proposal / evaluation events fall under the closed CloudEvent namespace set.
- **ADR 0034** — audit emission via the adapter / endpoint; retrospection runs are auditable.
- **ADR 0038** — Coach may use the policy simulator when a proposal touches policy-adjacent config.

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
B10 **defines no new CRD**. It is realized as instances of existing Canon CRDs:
- **`Agent`** (ARK, namespaced) — the Coach declared with `capabilitySetRefs[]`, `sdk`
  (`langgraph` | `deep-agents`), `modelRef`, `triggers` (its schedule), and optionally `exposes`
  (an A2A interface for handing analysis to/from HolmesGPT).
- **`AgentRun`** (ARK) — one scheduled retrospection execution, carrying `traceId`, `triggeredBy`.
- **`CapabilitySet`** (kopf/B13) — bundles the Coach's Langfuse/observability read access,
  `platform-knowledge-base` RAGStore, and `llmProviders[]`.
- **`Evaluation`** (ARK) — the Coach **reads** `Evaluation` results as an input signal. `[PROPOSED
  — not in source]` whether the Coach also *creates* `Evaluation` resources to validate a proposal
  before opening the PR (an A/B-style check) — source does not state this.
- **`Approval`** (B19) — a skill/prompt change gated `upon-approval` is represented as an
  `Approval` with `requestingAgent` = the Coach, `actionType`, `evidenceRefs[]`. Owned by B19;
  B10 produces the request payload only.

The **schedule trigger mechanism**: §6.10/§14.2 state the Coach "runs on schedule" but the
concrete trigger field is not specified. `[PROPOSED — not in source]` that the schedule is
expressed in `Agent.triggers` (a scheduled trigger) or a scheduled CI job invoking the Coach;
this is a design-time decision.

### 4.2 APIs / SDK surfaces
- **Consumes the Platform SDK (B6):** `memory.*`, `rag.*` (to query the Knowledge Base), OTel
  emission, and A2A registration helpers. Method signatures beyond these named groups are **not
  specified in source** (interface-contract §3.1).
- **Consumes Langfuse read APIs (A2)** to pull traces/evals/costs. Specific Langfuse API surface
  is the vendor's; not Canon.
- **A2A interface (optional, `exposes`):** if the Coach exposes A2A, it declares a version
  (e.g. `coachAgent.v1` — `[PROPOSED — not in source]`, the example shape per interface-contract §3.3).
- **Proposal output:** a **PR against Git** and/or a **Headlamp suggestion card**. The
  suggestion-card payload schema is owned by B5/B19; **not specified in source** here.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
Per-event-type names are deferred to B12's registry; B10 binds only to the **closed namespace set**:
- **Emits** under `platform.evaluation.*` — retrospection findings / proposed-improvement events
  ("A/B test results, red-team findings" sit in this namespace per interface-contract §2). Exact
  type names `[PROPOSED — not in source]`, defined in B12.
- **Emits** under `platform.audit.*` — every retrospection run and proposal (ADR 0034).
- **Emits/participates** under `platform.approval.*` when a proposal is gated — the *approval*
  events are emitted by B19; the Coach is the requester.
- **Emits** under `platform.lifecycle.*` — its own `AgentRun` lifecycle (created/started/completed/
  failed), as for any Platform Agent.
- **Consumes**: it reads Langfuse/audit data through APIs, not by subscribing to CloudEvents; no
  source-stated event subscription. `[PROPOSED — not in source]` if a `platform.observability.*`
  threshold-crossing were used to trigger an out-of-cycle retrospection.

### 4.4 Data schemas / connection-secret contracts
N/A — B10 provisions no datastore of its own and owns no connection secret. It reads via Langfuse
APIs and the Platform SDK; any memory it keeps is via a `Memory`/`MemoryStore` (ADR 0025), whose
connection secret follows the uniform `host`/`port`/`user`/`password`/`dbname` shape (ADR 0044).

## 5. OSS-vs-Custom Decision
**Build-new agent on OSS foundations.** The Coach is **not** a forked product; it is a custom
Platform Agent authored on the supported agent SDK (LangGraph / Deep Agents, B7/ADR 0019),
wrapping no new infrastructure. It **wraps/consumes** Langfuse (A2), ARK (A5), the Platform SDK
(B6), and reuses the HolmesGPT propose-via-PR pattern (A14/ADR 0012). No upstream "coach"
product is adopted; the logic (what to read, what to propose) is custom. No version is pinned by
source for a Coach upstream because there is none — it is platform-authored agent code.

## 6. Functional Requirements
- **REQ-B10-01:** The Coach MUST be declared as an `Agent` CRD, reconciled by ARK, run in a
  `Sandbox`, and call out only through LiteLLM — i.e. it is a first-class Platform Agent (§6.10).
- **REQ-B10-02:** The Coach MUST run **on a schedule** (not continuously / not per-request),
  producing one `AgentRun` per scheduled retrospection.
- **REQ-B10-03:** The Coach MUST consume **Langfuse traces** (A2) as a retrospection input.
- **REQ-B10-04:** The Coach MUST consume **audit data** and **`Evaluation` results** as additional
  retrospection inputs.
- **REQ-B10-05:** The Coach MUST produce improvement proposals as **auto-PRs and/or Headlamp
  suggestion cards** — it MUST NOT directly mutate any agent's prompt or `Skill` set.
- **REQ-B10-06:** Skill/prompt-change proposals that require human review MUST be routed through
  the **generalized approval system** (B19) as an `Approval`, with the Coach as `requestingAgent`
  and supporting `evidenceRefs[]` (ADR 0017).
- **REQ-B10-07:** Every retrospection run and every emitted proposal MUST emit audit events
  through the **platform audit adapter** to the audit endpoint (ADR 0034).
- **REQ-B10-08:** The Coach MUST include the `platform-knowledge-base` RAGStore in its
  `CapabilitySet` to ground proposals against platform docs/runbooks (§6.4, ADR 0022) — and MUST
  access it via CapabilitySet, never embed it.
- **REQ-B10-09:** The Coach MUST emit findings under `platform.evaluation.*` and lifecycle under
  `platform.lifecycle.*` (taxonomy per ADR 0031), with concrete schemas registered in B12.
- **REQ-B10-10:** The Coach's `Agent` CRD (and any A2A interface it `exposes`) MUST be versioned
  per ADR 0030 (per-component ownership; the Coach owns its own versioning lifecycle).
- **REQ-B10-11:** The Coach MUST run on a supported v1.0 agent SDK (`Agent.sdk` ∈ {`langgraph`,
  `deep-agents`}) per ADR 0019.
- **REQ-B10-12:** Read-mostly default — the Coach's only write effect is proposal emission; any
  autonomous write beyond proposals MUST be gated by OPA's three-state model (ADR 0012). (Whether
  OPA gating is wired at all is a design-time decision per §14.2; if wired, it follows ADR 0012.)

## 7. Non-Functional Requirements
- **Security / least privilege:** read-mostly CapabilitySet; no direct write to agent definitions;
  all writes are reviewable proposals (ADR 0012 propose-don't-mutate). The Coach's egress is
  allowlisted via its CapabilitySet/`EgressTarget`s; LLM calls go through LiteLLM only.
- **Multi-tenancy (§6.9):** the Coach runs in an approved namespace and only retrospects over
  agents/traces it is authorized to read; it MUST NOT cross tenant boundaries without an
  RBAC/OPA-controlled grant. `[PROPOSED — not in source]` whether a Coach instance is per-tenant
  or platform-wide — design-time decision (§14.2 phrasing leaves OPA optional).
- **Observability (§6.5):** the Coach emits OTel spans (correlated to Langfuse by `trace_id`,
  ADR 0015) and audit events; its own runs are first-class observable activity.
- **Scale:** scheduled, low-frequency batch workload — not on any hot path. Retrospection cost is
  bounded by the trace window it reads; no latency SLO applies.
- **Versioning (ADR 0030):** the Coach owns its `Agent`/A2A version lifecycle independently.

## 8. Cross-Cutting Deliverable Checklist
- Helm/manifests — **applicable** (the Coach `Agent` CRD + `CapabilitySet` manifests, scheduled trigger).
- Per-product docs (10.5) — **applicable** (what the Coach reads, what it proposes, how to act on a card).
- Runbook (10.7) — **applicable** (muting the Coach, handling a bad proposal, schedule changes).
- Alerts — **applicable** (Coach run failure, proposal-emission failure). `[PROPOSED — not in source]`.
- Grafana dashboard (Crossplane XR) — **applicable** (proposals opened/accepted/rejected over time,
  via `GrafanaDashboard` XR, ADR 0021). `[PROPOSED — not in source]` for exact panels.
- Headlamp plugin — **N/A (consumes B5/B19-ui)** — the suggestion-card UI is owned by B5/B19-ui;
  B10 emits the payload.
- OPA/Rego integration — **conditional / N/A by design-time decision** — §14.2 states the Coach
  "doesn't necessarily need OPA"; if write actions are added they are gated by ADR 0012's model.
- Audit emission (ADR 0034) — **applicable** (every run + proposal).
- Knative trigger flow — **applicable** (scheduled retrospection trigger; mechanism `[PROPOSED]`,
  may be `Agent.triggers` or a scheduled CI job).
- HolmesGPT toolset — **N/A** — the Coach is a *peer* Platform Agent, not a HolmesGPT toolset; it
  may *hand off* to HolmesGPT over A2A but does not contribute a toolset.
- 3-layer tests (Chainsaw/Playwright/PyTest) — **applicable** (Chainsaw: apply Coach `Agent`,
  expect scheduled `AgentRun` + audit/eval events; PyTest: retrospection logic; Playwright: a
  suggestion-card flow end-to-end, exercising B5/B19-ui).
- Tutorials & how-tos — **applicable** ("how to enable the Coach for your agents", "acting on a
  Coach suggestion card").

## 9. Acceptance Criteria
- **AC-B10-01:** Applying the Coach `Agent` CRD yields an ARK-reconciled agent that runs in a
  Sandbox and routes LLM calls through LiteLLM. (→ REQ-B10-01)
- **AC-B10-02:** On its schedule, exactly one `AgentRun` per cycle is created. (→ REQ-B10-02)
- **AC-B10-03:** A retrospection run reads Langfuse traces (verified by a Langfuse query in the
  run's trace). (→ REQ-B10-03)
- **AC-B10-04:** A retrospection run incorporates audit data and `Evaluation` results into its
  candidate set (verified by a fixture seeding both). (→ REQ-B10-04)
- **AC-B10-05:** A run produces a PR and/or suggestion card and makes **no** direct edit to any
  `Agent`/`Skill` resource (verified by absence of a direct CRD mutation). (→ REQ-B10-05)
- **AC-B10-06:** A skill-change proposal creates an `Approval` with the Coach as `requestingAgent`
  and populated `evidenceRefs[]`. (→ REQ-B10-06)
- **AC-B10-07:** Each run and proposal produces audit events at the audit endpoint. (→ REQ-B10-07)
- **AC-B10-08:** The Coach's `CapabilitySet` references `platform-knowledge-base`; removing it
  causes a documented validation/lint failure. (→ REQ-B10-08)
- **AC-B10-09:** Findings appear under `platform.evaluation.*` and lifecycle under
  `platform.lifecycle.*` on the broker. (→ REQ-B10-09)
- **AC-B10-10:** The Coach `Agent`/A2A interface declares a version; a breaking change mints a new
  version per ADR 0030. (→ REQ-B10-10)
- **AC-B10-11:** `Agent.sdk` is one of `langgraph` / `deep-agents`; any other value is rejected at
  admission. (→ REQ-B10-11)
- **AC-B10-12:** No code path mutates an agent definition without a proposal; if OPA gating is
  wired, an `upon-approval` action is held pending approval. (→ REQ-B10-12)

## 10. Risks & Open Questions
- **R1 (med):** The **schedule trigger mechanism** is `[PROPOSED — not in source]` — `Agent.triggers`
  scheduled trigger vs. scheduled CI job. Picking the CI path couples the Coach to B15; the CRD
  path keeps it self-contained. Blast radius: med (affects B15/B8 wiring).
- **R2 (med):** Whether the Coach **creates `Evaluation` resources** to A/B-validate a proposal
  before opening a PR is `[PROPOSED — not in source]`. If yes, it adds an ARK `Evaluation`
  dependency and an `platform.evaluation.*` round-trip.
- **R3 (low/med):** OPA gating is explicitly **optional** for the Coach (§14.2). If a later release
  grants the Coach autonomous writes, the three-state OPA model (ADR 0012) must be wired then —
  the design must not foreclose it. Blast radius: low now, med later.
- **R4 (med):** **Multi-tenant scoping** of the Coach (per-tenant vs platform-wide) is
  `[PROPOSED — not in source]`; a platform-wide Coach reading all tenants' traces is a
  cross-tenant read that needs RBAC/OPA justification (§6.9, ADR 0016/0018).
- **OQ1:** Does the Coach hand off to HolmesGPT over A2A, or only consume the same observability
  independently? `piece-index.csv` lists A14 upstream; §6.10 shows HolmesGPT exposing A2A. `[PROPOSED]`
  optional A2A handoff, not required for v1.0.
- **OQ2:** Suggestion-card payload schema — owned by B5/B19; B10 must bind to it once frozen.

## 11. References
- architecture-overview.md §6.10 (line 774; HolmesGPT/Coach self-management vs self-improvement,
  propose-via-PR / suggestion cards, three-state OPA model), §6.2 (line 247; ARK runtime, agent
  SDK), §6.5 (line 370; Langfuse/Tempo correlation, audit adapter), §6.4 (line 349; Knowledge Base
  primitive), §7.4 (line 1219; Git→CI→ArgoCD), §7.5 (approvals), §14.2 (line 1704; B10 row).
- Glossary: **Coach Agent** (line 25). interface-contract §2 (`platform.evaluation.*`), §3.1 (SDK).
- ADR 0012 (HolmesGPT first-class agent / propose-via-PR / three-state OPA), ADR 0017 (approval
  system — Coach skill changes), ADR 0019 (agent SDK), ADR 0022 (Knowledge Base), ADR 0030
  (versioning), ADR 0031 (CloudEvent taxonomy), ADR 0034 (audit), ADR 0038 (policy simulator).
- Related pieces: A2, A5, A14, B6, B7, B19 (approval), B5/B19-ui (suggestion-card UI), B12 (schemas).
