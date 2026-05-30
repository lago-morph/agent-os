# SPEC ADR-0022 — Knowledge Base as a separate primitive [PROPOSED]

> kind: ADR · workstream: — · tier: T1
> upstream: [A11] · downstream: [A14, A16, B10, C8, B6] · adrs: [0022] · views: [6.4]
> canon-glossary: cf2d1a754a58 · canon-interface: 45ee7b798c47

## 1. Purpose & Problem Statement

ADR 0022 is a settled decision: the Knowledge Base is a first-class `RAGStore` named `platform-knowledge-base`, independent of HolmesGPT and any other agent, reached only through the SDK `rag.*` API on the standard LiteLLM-mediated call path. This SPEC states what honoring that decision obliges of the store definition, the consuming agents, and the access path — plus acceptance criteria proving it is honored. It does not re-argue folding-into-an-agent versus separate-primitive; that choice is settled.

The problem the decision solves: multiple consumers (HolmesGPT, Interactive Access Agent, Coach Agent, any opt-in Platform Agent) need the same corpus of platform docs / runbooks / pinned vendor docs, with an ingestion cadence independent of any agent's lifecycle, while keeping retrieval observable, policy-gated, and traced like the rest of agent traffic.

## 2. Scope

### 2.1 In scope
- The obligation that the Knowledge Base exist as exactly one `RAGStore` named `platform-knowledge-base`, not embedded in any agent.
- The obligation that consumers reach it only via `CapabilitySet` inclusion and the SDK `rag.*` API on the LiteLLM-mediated path (observable, policy-gated, traced).
- The obligation that the Interactive Access Agent (A16) is the default LibreChat endpoint by virtue of including `platform-knowledge-base` in its CapabilitySet — LibreChat does not bind the store directly.
- Reuse of OpenSearch (A11, ADR 0009) as the retrieval substrate — no new substrate.

### 2.2 Out of scope (and where it lives instead)
- `RAGStore` ingestion pipeline / indexing conventions — component **C8** (Knowledge Base RAG indexing pipeline) SPEC.
- Vendor-doc acquisition / re-indexing — separate companion project (**ADR 0024**, Workstream F item F3).
- Agent-pod access pattern (SDK call vs read-only filesystem mount) — deferred design-time sub-decision (architecture-backlog § 1.7).
- A richer search-and-browse UI beyond LibreChat — deferred revisit trigger (architecture-backlog § 3.10).
- `rag.*` method signatures beyond the named surface group — Platform SDK (**B6**); not specified in source.

## 3. Context & Dependencies

Upstream consumed: **A11** OpenSearch (retrieval/index substrate for the store, ADR 0009). **B6** Platform SDK (`rag.*` surface) as the sole access API.
Downstream consumers: **A14** HolmesGPT, **A16** Interactive Access Agent, **B10** Coach Agent, and any Platform Agent that opts in via CapabilitySet; **C8** ingestion pipeline populates the store.

ADR decisions honored:
- **ADR 0022** (this) — single named `RAGStore`, separate primitive, `rag.*`-only access.
- **ADR 0009** — OpenSearch is the search/vector store; no separate retrieval substrate.
- **ADR 0013** — the store is a `RAGStore` capability CRD, included via `CapabilitySet`; never configured directly on the gateway.
- **ADR 0024** — vendor-doc acquisition is an upstream producer (companion project), not part of this primitive.
- **ADR 0007** — LibreChat is locked down; it reaches the Knowledge Base only through the A16 agent, not by binding the store.

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs
- `RAGStore` (namespaced, kopf-operator/B13-reconciled) — the single instance named `platform-knowledge-base`. Source-stated fields: `backend`, `indexes[]`, `contentSourceRefs[]`, `ingestionPipelineRef`. Versioned per ADR 0030.
- `CapabilitySet` — consumers reference the store via `ragStores[]`. No new CRD introduced.

### 4.2 APIs / SDK surfaces
- SDK `rag.*` (owner **B6**) is the sole consumption API on the LiteLLM-mediated call path. Method signatures beyond the `rag.*` surface group are not specified in source — `[PROPOSED — not in source]` if detailed.

### 4.3 CloudEvents emitted / consumed
- Capability-registry changes to the `RAGStore` flow under `platform.capability.*` (e.g. `platform.capability.changed` per ADR 0013). No new top-level namespace (closed set, ADR 0031).
- Retrieval calls are traced on the LiteLLM-mediated path; any per-event-type retrieval CloudEvent name is `[PROPOSED — not in source]` (deferred to B12 registry).

### 4.4 Data schemas / connection-secret contracts
- Index/content lives in OpenSearch (A11); index schema and ingestion conventions are a **C8** design-time deliverable — `[PROPOSED — not in source]` if detailed here. No new connection secret introduced by this decision.

## 5. OSS-vs-Custom Decision
N/A — ADR. (Enforcement note: reuses **OpenSearch** (ADR 0009) as substrate and the **LiteLLM** gateway path; the `RAGStore` CRD is reconciled by the custom kopf operator B13. No fork; no new retrieval substrate.)

## 6. Functional Requirements
- REQ-ADR-0022-01: The platform MUST define the Knowledge Base as exactly one `RAGStore` named `platform-knowledge-base`, not embedded in HolmesGPT, the Interactive Access Agent, Coach, or any other agent.
- REQ-ADR-0022-02: A Platform Agent MUST gain access to the Knowledge Base only by including `platform-knowledge-base` in its `CapabilitySet`; an agent without it MUST NOT reach the store.
- REQ-ADR-0022-03: All consumer access MUST go through the SDK `rag.*` API on the LiteLLM-mediated call path, so retrieval is observable, policy-gated, and traced uniformly with other agent traffic.
- REQ-ADR-0022-04: The Interactive Access Agent (A16) MUST be the default LibreChat endpoint by including `platform-knowledge-base` in its CapabilitySet; LibreChat MUST NOT bind the store directly.
- REQ-ADR-0022-05: The Knowledge Base MUST reuse OpenSearch (ADR 0009) as its retrieval/index substrate; no separate retrieval substrate is introduced.

## 7. Non-Functional Requirements
- Observability (§6.5): retrieval traffic MUST appear in the same trace/observability surfaces as other LiteLLM-mediated calls.
- Security/multi-tenancy: access is gated by CapabilitySet membership and OPA on the gateway path (ADR 0018); the store is an Approved capability and unreachable otherwise.
- Versioning (ADR 0030): the `RAGStore` CRD versions per the CRD policy; the `rag.*` SDK surface versions with the Platform SDK.
- Lifecycle independence: the store ships and evolves on its own ingestion cadence, decoupled from any consuming agent's lifecycle.

## 8. Cross-Cutting Deliverable Checklist
N/A — ADR (verification map lives in the PLAN). The §14.1 standard set applies to the enforcing components (C8 ingestion, A16/A14/B10 consumers, B13 reconciler), not to this decision record.

## 9. Acceptance Criteria
- AC-ADR-0022-01: Honored when exactly one `RAGStore` named `platform-knowledge-base` exists and no agent image embeds the corpus. (REQ-01)
- AC-ADR-0022-02: Honored when an agent lacking the store in its CapabilitySet gets no retrieval results and an agent with it succeeds. (REQ-02)
- AC-ADR-0022-03: Honored when a retrieval call produces a trace on the LiteLLM-mediated path and is subject to OPA gating. (REQ-03)
- AC-ADR-0022-04: Honored when the A16 CapabilitySet lists `platform-knowledge-base` and LibreChat config contains no direct store binding. (REQ-04)
- AC-ADR-0022-05: Honored when the store's `backend` resolves to OpenSearch and no other retrieval substrate is provisioned for it. (REQ-05)

## 10. Risks & Open Questions
- OQ-1 (low): Agent-pod access pattern (SDK call vs read-only mount) is a deferred design-time sub-decision (backlog § 1.7); ACs assume SDK-path access. `[PROPOSED]`
- R-1 (low): Until C8 ingestion and the ADR 0024 companion project land, corpus coverage is partial; blast radius is answer quality, not the access contract.

## 11. References
- ADR 0022 (`adr/0022-knowledge-base-as-separate-primitive.md`) — the decision enforced here.
- architecture-overview.md §6.4 (Knowledge Base as a separate primitive), §15 glossary ("Knowledge Base").
- Enforcing/related components: C8 (ingestion), A16 (Interactive Access Agent), A14 (HolmesGPT), B10 (Coach), A11 (OpenSearch), B6 (`rag.*`), B13 (RAGStore reconciler).
- ADR 0007, 0009, 0013, 0024 (cited constraints).
