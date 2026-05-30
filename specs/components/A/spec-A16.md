# SPEC A16 — Interactive Access Agent

> kind: COMPONENT · workstream: A · tier: T1
> upstream: [A1, A14] · downstream: [] · adrs: [0007] · views: [6.2, 6.4, 6.10]
> canon-glossary: b0edae10 · canon-interface: 0ce201d5

## 1. Purpose & Problem Statement

A16 is the **Interactive Access Agent** — a general-purpose Platform Agent that LibreChat connects to as the **default chat endpoint** (glossary; ADR 0007 consequence; §6.4, §6.10). It is implemented **early** so the platform uses its own infrastructure to let LibreChat reach LLMs from day one (§6.4) — before the bulk of agent tooling exists, there is at least one working endpoint a user can talk to. It includes the **Knowledge Base** (`platform-knowledge-base` RAGStore) in its CapabilitySet and reaches it the same way any agent does — through the SDK's `rag.*` API, going through LiteLLM on the standard call path (§6.4, §6.10).

The problem A16 solves is "a working default conversational endpoint": LibreChat needs a Platform Agent to surface as its default endpoint so developers can reach LLMs and the Knowledge Base through the governed path (LiteLLM auth/OPA/audit/Langfuse) from the start. A16 is the same kind of Platform Agent as HolmesGPT — declared as an `Agent` CRD, run in a Sandbox, calling out only through LiteLLM and Envoy — but general-purpose and chat-facing rather than diagnostic. It and HolmesGPT both query the same Knowledge Base, which is a separate primitive, not part of either agent (§6.10).

## 2. Scope

### 2.1 In scope

- An `Agent` CRD definition for the Interactive Access Agent: general-purpose, chat-facing, with a CapabilitySet that includes the Knowledge Base RAGStore (§6.4, §6.10).
- Wiring it as the **default LibreChat endpoint** — the endpoint LibreChat connects to from day one (ADR 0007 consequence; §7.1). LibreChat surfaces it via the endpoint picker; visibility is driven by platform JWT claims (§6.9, §6.11).
- Knowledge Base access via the SDK `rag.*` path through LiteLLM (§6.4); design-time choice of SDK-API vs filesystem mount per access pattern (§6.4).
- Conversation memory: uploads from LibreChat are routed to the `Memory` CRD per ADR 0025 access modes and exposed to the connected agent (ADR 0007 decision) — A16 is the agent those uploads attach to.
- Standard Workstream A deliverables for an agent component (Helm/CRD manifests, docs, runbook, alerts, dashboard, OPA integration, audit, trigger-flow design where applicable, HolmesGPT toolset contribution, 3-layer tests, tutorials/how-tos) (§14.1, A16 row).

### 2.2 Out of scope (and where it lives instead)

- LibreChat install + `librechat.yaml` lockdown + upload→Memory wiring + Keycloak SSO + the OpenAI↔A2A adapter — **A8** (ADR 0007). A16 is the endpoint A8 connects to, not the chat frontend.
- The `Agent` CRD reconciler / ARK — **A5**. A16 is an `Agent` instance.
- LiteLLM gateway + model routing + OPA/audit/Langfuse callbacks — **A1** / **B2**.
- The Knowledge Base RAGStore itself + its indexing pipeline — **B4** (`RAGStore`/`MemoryStore` composition) + **C8** (indexing); A16 only includes it in its CapabilitySet.
- The CapabilitySet / capability CRDs reconciliation into LiteLLM — **B13** (ADR 0013).
- HolmesGPT — **A14** (upstream; shares the KB).
- Agent profile library / recommended compositions — **B17** / **B18** (A16 may be expressed using a profile, but the library is theirs).
- The agent SDK harness image — **B7**; A16 declares `sdk` (`langgraph`/`deep-agents`, ADR 0019) but does not build the harness.

## 3. Context & Dependencies

**Upstream consumed (HARD, per CSV):**
- **A1** (LiteLLM gateway) — A16's LLM, RAG (`rag.*`), and any MCP/A2A calls terminate at LiteLLM where auth/OPA/audit/Langfuse fire. Without A1 there is no governed call path.
- **A14** (HolmesGPT) — `[PROPOSED — not in source]` the CSV lists A14 as A16's upstream; Canon's stated relationship is that A16 and A14 **share the same Knowledge Base** primitive (§6.10) rather than A16 calling A14. Treated here as: A16 reuses the KB-access pattern and CapabilitySet wiring established alongside A14, and may hand off to HolmesGPT via A2A. Flagged because the dependency direction is not spelled out in §6.x.

**Upstream consumed (effective, not in CSV edge set):** A5 (ARK `Agent` CRD), A8 (LibreChat connects to A16 as default endpoint), B13 (CapabilitySet→LiteLLM), B4 (KB RAGStore composition), B6 (SDK `rag.*`). Consumed as they land. `[PROPOSED — not in source]` these are functional relationships beyond the CSV A1;A14 edges.

**Downstream consumers:** none in CSV.

**ADR decisions honored:**
- **ADR 0007** — A16 is the default endpoint LibreChat connects to from day one; endpoint visibility from platform JWT claims; LibreChat uploads route to the `Memory` CRD and attach to the connected agent (A16).
- **ADR 0019** — `Agent.sdk` is `langgraph` or `deep-agents`.
- **ADR 0022 / §6.4** — the Knowledge Base is a separate primitive included via CapabilitySet, reached through `rag.*`/LiteLLM.
- **ADR 0025** — uploaded artifacts land in a `Memory` whose `MemoryStore.accessMode` governs access.
- **ADR 0012 / §6.10** — A16 and HolmesGPT share the KB; A16 is a general-purpose Platform Agent under the same governance.
- **ADR 0002 / 0034 / 0031 / 0030** — OPA gating, audit adapter, CloudEvent taxonomy, versioning (as for all agents).

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)

A16 **owns no CRD**; it is an `Agent` instance (owner A5):

| Resource | Owner | Fields A16 sets (source-stated) |
|---|---|---|
| `Agent` | A5 (ARK) | `capabilitySetRefs[]` (includes the KB `RAGStore`), `sdk` (`langgraph`/`deep-agents`), `image`, `sandboxTemplateRef`, `modelRef`, `memoryRefs[]` (conversation/upload memory), `exposes` (if it offers an A2A/MCP interface) |
| `CapabilitySet` | B13 | `ragStores[]` (includes `platform-knowledge-base`), `llmProviders[]`, `opaPolicyRefs[]`, … |
| `Memory` | A5 | `memoryStoreRef` — destination for LibreChat uploads (ADR 0007/0025) |

`[PROPOSED — not in source]` no A16-specific `Agent` field is invented. The concrete CapabilitySet/profile contents beyond "includes the KB" are design-time (may draw on B17 profiles).

### 4.2 APIs / SDK surfaces

- A16 is reached **as a LibreChat endpoint** over the OpenAI-compatible HTTP surface, translated to A2A at the gateway (glossary: A2A; ADR 0007 streaming preserved across the OpenAI-compat↔A2A boundary). The endpoint is surfaced through LiteLLM, not a bespoke A16 API.
- A16 consumes the Platform SDK `rag.*` (KB queries) and `memory.*` (conversation/upload memory) through LiteLLM/Letta (B6).
- `[PROPOSED — not in source]` if A16 exposes its own A2A interface (e.g. for handoff to HolmesGPT), the interface identifier/version is design-time per interface-contract §3.3; not in Canon.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)

- **Emitted:** A16's runs (as an `AgentRun`) surface lifecycle under **`platform.lifecycle.*`** (via ARK/A5); its conversation/tool actions audit under **`platform.audit.*`** via the adapter (LiteLLM callbacks emit the request/response audit on its call path). 
- **Consumed:** none required beyond chat-request-driven `AgentRun` creation.
- Per-event-type names deferred to B12. `[PROPOSED — not in source]` concrete names not in Canon.

### 4.4 Data schemas / connection-secret contracts

- N/A — A16 owns no datastore. Conversation/upload state lives in the `Memory`/`MemoryStore` (Letta→Postgres) per ADR 0025; LibreChat-resident conversation state lives in the shared Postgres (A8/§7.2). A16 produces no substrate connection secret.

## 5. OSS-vs-Custom Decision

- **Build-new (configuration of platform primitives):** A16 is not a third-party product — it is a **declared Platform Agent** (an `Agent` CRD + CapabilitySet) assembled from platform building blocks (ARK, LiteLLM, the KB RAGStore, the agent SDK). The "custom" work is the agent definition + CapabilitySet + LibreChat default-endpoint wiring, not a new service.
- **Rationale:** §6.4 mandates the KB be reached via the standard `rag.*`/LiteLLM path and that a general-purpose agent be the default LibreChat endpoint from day one; ADR 0007 names A16 as that endpoint. No OSS product substitutes for "the platform's own default agent."
- `[PROPOSED — not in source]` whether A16 is authored directly or derived from a B17 profile is design-time.

## 6. Functional Requirements

- REQ-A16-01: A16 is defined as an `Agent` CRD (general-purpose, chat-facing) with `sdk` ∈ {`langgraph`, `deep-agents`} and a CapabilitySet that includes the `platform-knowledge-base` RAGStore.
- REQ-A16-02: A16 is wired as the **default endpoint LibreChat connects to**, surfaced in the LibreChat endpoint picker, with visibility driven by platform JWT claims (a user sees it only if their claims/roles permit).
- REQ-A16-03: A16 answers Knowledge-Base-grounded questions via the SDK `rag.*` path through LiteLLM (the standard governed call path).
- REQ-A16-04: All A16 LLM/RAG/MCP/A2A traffic terminates at LiteLLM where auth, OPA, audit, and Langfuse callbacks fire; any non-LiteLLM outbound HTTP goes only to `EgressTarget`-allowlisted destinations via Envoy.
- REQ-A16-05: LibreChat uploads attached to an A16 conversation route to a `Memory` whose `MemoryStore.accessMode` governs access (ADR 0007/0025); A16 is the agent those uploads are exposed to.
- REQ-A16-06: A16 runs in a Sandbox (via `sandboxTemplateRef`) like any Platform Agent; it has no privileged or out-of-band path.
- REQ-A16-07: A16's actions emit audit through the adapter and lifecycle under `platform.lifecycle.*`; it introduces no new top-level CloudEvent namespace.
- REQ-A16-08: A16 is admission- and runtime-gated by OPA like any agent (CapabilitySet scoping at admission, LiteLLM OPA at runtime, LibreChat agent-pick gating via claims).
- REQ-A16-09: A16 ships early enough that LibreChat has a working default endpoint from day one (ordering constraint, not just functional).
- REQ-A16-10: A16 delivers per-product docs, runbook, alerts, a `GrafanaDashboard` XR, a HolmesGPT toolset contribution (e.g. KB-query/conversation-health), and tutorials/how-tos.

## 7. Non-Functional Requirements

- **Security / multi-tenancy (§6.9):** A16 is a general-purpose endpoint reachable by many users — its CapabilitySet must be scoped so it cannot exceed approved capabilities; per-user endpoint visibility and agent-pick are claim-driven (§6.11) and OPA-gated. Prompt-injection from chat input is the salient threat (ADR 0027 / B22) — enforcement is external (LiteLLM/OPA/Envoy/sandbox), independent of harness internals.
- **Observability (§6.5):** LLM-grade traces to Langfuse; OTel spans correlated by `trace_id` (ADR 0015); alerts on endpoint unavailability / elevated error rate.
- **Streaming (ADR 0007 / §6.1):** streaming is preserved end-to-end LibreChat↔LiteLLM↔A16 across the OpenAI-compat↔A2A boundary.
- **Versioning (ADR 0030):** if A16 exposes an A2A/MCP interface, it declares a version; the `Agent` CRD follows ARK's CRD versioning.

## 8. Cross-Cutting Deliverable Checklist

| Deliverable (§14.1) | Status |
|---|---|
| Helm / manifests in Git | Applicable — `Agent` + CapabilitySet manifests (and LibreChat default-endpoint wiring config) |
| Per-product docs (10.5) | Applicable |
| Operator runbook (10.7) | Applicable |
| Backup / restore | N/A — no system-of-record state of its own (conversation memory in Letta/Postgres) |
| Alert rules | Applicable — endpoint down / error rate |
| Grafana dashboard (Crossplane XR) | Applicable |
| Headlamp plugin | N/A — A16 is an agent instance; visibility is via the ARK plugin (A5), no dedicated A16 plugin |
| OPA / Rego integration | Applicable — CapabilitySet admission scoping + LibreChat agent-pick + runtime gating targets to B16 |
| Audit emission (ADR 0034) | Applicable — conversation/tool actions via LiteLLM callbacks + adapter |
| Knative trigger flow | N/A — A16 is chat-request-driven, not event-triggered in v1.0 (no authored trigger flow) |
| HolmesGPT toolset | Applicable — KB-query / conversation-health toolset contribution |
| 3-layer tests | Applicable — Chainsaw (Agent admit/reconcile), Playwright (LibreChat default-endpoint chat + KB answer), PyTest (rag.* / upload-memory logic) |
| Tutorials & how-tos | Applicable |

## 9. Acceptance Criteria

- AC-A16-01 (REQ-A16-01): The A16 `Agent` reconciles with `sdk` ∈ {langgraph, deep-agents} and a CapabilitySet whose `ragStores[]` includes `platform-knowledge-base`.
- AC-A16-02 (REQ-A16-02): A LibreChat user whose claims permit it sees A16 as the default endpoint and can chat; a user lacking the claim does not see it.
- AC-A16-03 (REQ-A16-03): A KB-grounded question returns an answer sourced via `rag.*` against `platform-knowledge-base` through LiteLLM.
- AC-A16-04 (REQ-A16-04): A16 has no working network path to an LLM/MCP/A2A target except through LiteLLM, and no outbound HTTP except to `EgressTarget`-allowlisted FQDNs via Envoy; a disallowed egress attempt is blocked.
- AC-A16-05 (REQ-A16-05): A file uploaded in an A16 LibreChat conversation lands in a `Memory` whose `accessMode` is enforced and is readable by A16 in that conversation.
- AC-A16-06 (REQ-A16-06): A16 runs inside a Sandbox; no privileged escape path exists.
- AC-A16-07 (REQ-A16-07): A16 emits `platform.lifecycle.*` run events and `platform.audit.*` action audit; no event uses a new top-level namespace.
- AC-A16-08 (REQ-A16-08): An over-broad CapabilitySet for A16 is rejected at admission; a runtime call outside policy is denied at the LiteLLM OPA callback.
- AC-A16-09 (REQ-A16-09): In an environment with only the early wave landed (A1, KB, ARK), LibreChat reaches LLMs/KB through A16 as the default endpoint.
- AC-A16-10 (REQ-A16-10): Docs, runbook, alerts, `GrafanaDashboard` XR, HolmesGPT toolset, and tutorials exist.

## 10. Risks & Open Questions

- R-A16-1 (med): the CSV `A14` upstream edge is direction-ambiguous — Canon states A16 and A14 **share** the KB, not that A16 calls A14. `[PROPOSED]` reconciliation: treat A14 as a shared-KB / A2A-handoff peer, not a hard call dependency; confirm intended edge with the architecture owner. Blast radius med (affects wave sequencing assumptions).
- R-A16-2 (med): general-purpose chat endpoint is the platform's largest prompt-injection surface (untrusted user/text input). Mitigation: external enforcement (LiteLLM/OPA/Envoy/sandbox) + claim-driven visibility; principal item for B22.
- R-A16-3 (low): A16 must ship early but functionally needs A1, ARK, the KB RAGStore, B13, B6. Reconciliation: CSV wave is W2 (after A1@W0 and A14@W0); KB/ARK/B13 land W1, so the "from day one" property is satisfied within the early waves, not literally before A1.
- Open question (low): is A16 authored directly or derived from a B17 profile? `[PROPOSED — not in source]`; design-time.
- Open question (low): per-event-type names deferred to B12. `[PROPOSED]`.

## 11. References

- architecture-overview.md §6.2 (agent runtime / Platform-SDK surfaces, 247–292), §6.4 (KB as separate primitive / access patterns / Interactive Access Agent, 349–368), §6.9 (multi-tenancy/claims, 720+), §6.10 (HolmesGPT + shared KB + Interactive Access Agent, 774–836), §6.11 (identity federation), §7.1 (interactive chat), §14.1 (A16 row 1682).
- ADR 0007 (LibreChat — A16 as default endpoint, upload→Memory); ADR 0019 (SDK values); ADR 0022 (KB as separate primitive); ADR 0025 (memory access modes); ADR 0012 (HolmesGPT shared-KB); ADR 0002/0034/0031/0030/0015.
- interface-contract §1.2 (`Agent`), §1.4 (`CapabilitySet`, `RAGStore`), §3.1 (SDK `rag.*`/`memory.*`), §2 (CloudEvents). glossary (Interactive Access Agent, Knowledge Base, A2A, Platform Agent). Related: A8, A1, A14, A5, B13, B6, B17.
