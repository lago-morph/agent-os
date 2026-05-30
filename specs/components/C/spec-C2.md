# SPEC C2 — Diataxis tutorials

> kind: COMPONENT · workstream: C · tier: T1
> upstream: [C1] · downstream: [C8] · adrs: [0019, 0008, 0024, 0010, 0033, 0011] · views: [6.4]
> canon-glossary: see _meta/glossary.md · canon-interface: see _meta/interface-contract.md

## 1. Purpose & Problem Statement

Tutorials are the **learning-oriented** Diataxis quadrant (§10.1): hand-holding, end-to-end,
working-result content for someone who has never used the platform. They are the on-ramp that
turns the architecture into something a newcomer can actually run, and — per the
docs-before-implementation commitment (§10) — each tutorial is authored **before** the component
it exercises is implemented, so it doubles as the source of acceptance test cases for that
component.

C2 owns the authored tutorial corpus that fills the tutorials slot defined by C1. Every tutorial
uses **Langchain Deep Agents** as the agent SDK throughout (ADR 0019) — the opinionated,
batteries-included default that reaches a working agent in the fewest steps; tutorials do not
introduce raw **LangGraph** (that surface lives in C4 reference and a dedicated C3 how-to). Without
a complete, structurally-consistent tutorial set, the docs-as-code on-ramp is incomplete and the
`platform-knowledge-base` `RAGStore` (§6.4) is missing the learning-oriented content newcomers and
Platform Agents query.

## 2. Scope

### 2.1 In scope
- The full set of §10.1 tutorials, each learning-oriented, end-to-end, with a working result:
  - Build your first Platform Agent (declarative `Agent` CRD, deploy via ArgoCD, invoke via
    LibreChat or A2A).
  - Build your first agent with the Platform SDK (BYO container).
  - Add an MCP server through the gateway (define an `MCPServer` CR, see it appear in the registry).
  - Compose a synthetic MCP server from an OpenAPI spec (`SyntheticMCPServer`).
  - Create a `Skill`, attach it to an agent, test it.
  - Run an evaluation suite against an agent (`Evaluation`).
  - Trigger an agent on a schedule.
  - Trigger an agent from an event (S3 drop, webhook).
  - Set up a long-running agent with checkpointing.
  - Use the Knowledge Base RAG from a custom agent (`platform-knowledge-base` `RAGStore`).
- Uniform tutorial structure: prerequisites, ordered steps, expected output at each step, and a
  verifiable end state — authored against C1's conventions and navigation slot.
- **Concrete test expectations** embedded per tutorial (§10) that are reviewed/signed off before
  the documented component is implemented, and become that component's acceptance test cases.
- **Mock-out instructions** for any tutorial whose dependency component has not yet landed (§10),
  so the tutorial is runnable against the mock and re-run against the real dependency later.
- Deep-Agents-only SDK usage throughout (ADR 0019).

### 2.2 Out of scope (and where it lives instead)
- Portal infrastructure, nav skeleton, search, per-version builds, contribution-workflow checks —
  **C1**.
- Problem-oriented how-to guides — **C3**. Information-oriented reference — **C4**.
  Understanding-oriented explanation — **C5**.
- Raw LangGraph lower-level surface — **C4** reference and the dedicated **C3** BYO-SDK how-to
  (tutorials stay Deep-Agents-only per ADR 0019).
- Per-product docs (§10.5) and per-product runbooks (§10.7) — **Workstream A** owners.
  Cross-cutting runbooks — **C6**. Maintainer/extender docs — **C7**.
- The RAG **indexing pipeline** and `platform-knowledge-base` wiring — **C8** (C2 only emits
  re-indexable Markdown).

## 3. Context & Dependencies

Upstream consumed: **C1** — the portal infrastructure, Diataxis navigation skeleton, MkDocs/
Markdown authoring conventions, and the contribution-workflow check that C2 content plugs into.

Continuous (non-blocking) inputs: **every Workstream A/B component each tutorial exercises** (A1
LiteLLM, A5 ARK, A6 agent-sandbox, A8 LibreChat, A12 OpenAPI→MCP, A17 MCP services, B6 Platform
SDK, B7 agent SDK, B9 CLI, B13 kopf operator, etc.) — tutorials are authored ahead of these per
the docs-before-implementation process; the relevant component is consumed continuously as it
lands and the tutorial's mock is swapped for the real dependency. Recorded as continuous inputs,
not blockers.

Downstream consumers: **C8** (indexes the tutorial Markdown into `platform-knowledge-base`).

ADR decisions honored:
- **ADR 0019** — tutorials use Langchain Deep Agents throughout (`Agent.sdk` value `deep-agents`);
  raw LangGraph is not introduced in tutorials.
- **ADR 0008** — Material for MkDocs docs-as-code; tutorial Markdown lives in-repo, reviewed in PR
  alongside code, and is re-indexable by C8.
- **ADR 0011** — three-layer testing with CLI orchestration; embedded test expectations map to the
  layered test framework (`agent-platform` CLI / B9 / B14).
- **ADR 0010 / 0033** — the docs build/lint/preview check runs as a GitHub Actions workflow against
  GitHub; any CI-touching tutorial step references the GitHub Actions reference pipeline (C3).
- **ADR 0024** — tutorials reference platform docs only; vendor-doc acquisition is the separate
  companion project.

## 4. Interfaces & Contracts

Use ONLY Canon names. Tutorials reference these Canon CRDs/primitives as authored subjects, not as
new interfaces C2 defines: `Agent`, `MCPServer`, `SyntheticMCPServer`, `Skill`, `Evaluation`,
`RAGStore`, `platform-knowledge-base`, ARK, ArgoCD, LibreChat, A2A, Langchain Deep Agents.

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
N/A — C2 defines no CRD/XRD. It documents the use of Canon CRDs (`Agent`, `MCPServer`,
`SyntheticMCPServer` (XR), `Skill`, `Evaluation`, `RAGStore`) owned by their respective components.

### 4.2 APIs / SDK surfaces
N/A — C2 exposes no API/SDK surface. It documents the **Langchain Deep Agents** authoring surface
(ADR 0019) and the Platform SDK surfaces (`memory.*`, `rag.*`, OTel emission, A2A registration
helpers — interface-contract §3.1) owned by B6/B7; specific method signatures beyond those named
surface groups are not specified in source and MUST NOT be invented — `[PROPOSED — not in source]`
for any concrete method a tutorial needs that is not yet published.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
N/A — C2 emits no CloudEvents. The event-trigger tutorial documents the `platform.lifecycle.*`
trigger path (AgentRun created from an event) at a conceptual level; concrete per-event-type names
are deferred to the B12 schema registry and MUST be referenced, not invented.

### 4.4 Data schemas / connection-secret contracts
N/A — C2 writes no data/connection-secret. The only contract is the **re-indexable Markdown corpus
shape** consumed by C8 (co-owned with C1/C8).

## 5. OSS-vs-Custom Decision
N/A — C2 is authored content, not software. It consumes the **Material for MkDocs** portal
(configured by C1, ADR 0008) and authors Markdown tutorials within it; no OSS project is forked,
wrapped, or built.

## 6. Functional Requirements
- REQ-C2-01: C2 SHALL author every tutorial enumerated in §10.1, each learning-oriented,
  end-to-end, and producing a working result for a first-time user.
- REQ-C2-02: Every tutorial SHALL use **Langchain Deep Agents** as the agent SDK and SHALL NOT
  introduce raw LangGraph (ADR 0019).
- REQ-C2-03: Every tutorial SHALL follow a uniform structure — prerequisites, ordered steps,
  expected output per step, and a verifiable end state — authored against C1's conventions and
  navigation slot.
- REQ-C2-04: Every tutorial SHALL embed **concrete test expectations** (§10) reviewed and signed
  off before the documented component is implemented, becoming that component's acceptance test
  cases.
- REQ-C2-05: Every tutorial whose dependency component is not yet implemented SHALL include
  **mock-out instructions** enabling the tutorial to run against the mock, with the mock replaced
  and the same tests re-run when the real dependency lands (§10).
- REQ-C2-06: Tutorials SHALL use ONLY Canon names for CRDs, products, and primitives; any element
  not present in Canon SHALL be tagged `[PROPOSED — not in source]`.
- REQ-C2-07: Tutorial Markdown SHALL be committed in-repo as docs-as-code, reviewed in the same PR
  path as related code, and pass the C1 contribution-workflow check (ADR 0008, ADR 0010 / 0033).
- REQ-C2-08: Tutorial Markdown SHALL conform to the clean-Markdown corpus contract so C8 can
  re-index it into the `platform-knowledge-base` `RAGStore` without portal-specific preprocessing.
- REQ-C2-09: Tutorial steps that touch CI SHALL reference the GitHub Actions reference pipeline
  (C3 / ADR 0010 / 0033) rather than an unsupported CI system.

## 7. Non-Functional Requirements
- Security: tutorials show secrets handled via ESO and capabilities declared via CRDs (never
  configured directly on the gateway); the docs check runs as a security-first GitHub Actions
  workflow (ADR 0010).
- Multi-tenancy (§6.9): tutorials run inside an approved namespace and illustrate namespace-scoped
  capability access; they carry no tenant data.
- Observability (§6.5): the checkpointing and evaluation tutorials surface trace/eval signals
  (Langfuse/Tempo) at the conceptual level; build success is visible via the GitHub Actions check.
- Scale: corpus grows with the tutorial set; per-version builds are additive (C1).
- Versioning (ADR 0030): tutorials are pinned to the documented component/API versions via C1's
  per-version builds; the `Agent.sdk` value `deep-agents` is the v1.0 tutorial SDK (ADR 0019).

## 8. Cross-Cutting Deliverable Checklist
- Helm/manifests: **N/A — C2 is authored content; build/deploy is owned by C1.**
- Per-product docs (10.5): **N/A — owned by Workstream A; C2 authors Diataxis tutorials only.**
- Runbook (10.7): **N/A — runbooks are C6 + Workstream A.**
- Alerts: **N/A — content piece, no runtime failure conditions.**
- Grafana dashboard (Crossplane XR): **N/A — no runtime metrics.**
- Headlamp plugin: **N/A — no UI surface (portal-vs-Headlamp linking deferred, backlog §1.15).**
- OPA/Rego integration: **N/A — content piece, no policy surface.**
- Audit emission (ADR 0034): **N/A — no audit-relevant runtime actions.**
- Knative trigger flow: **N/A — re-index triggering is owned by C8; tutorials only describe trigger flows.**
- HolmesGPT toolset: **N/A — KB access is via the RAGStore (C8), not a C2 toolset.**
- 3-layer tests (Chainsaw/Playwright/PyTest): **Applicable** — each tutorial's embedded test
  expectations map to the three layers (ADR 0011) and are executed as the documented component's
  acceptance tests; Playwright covers tutorial render/nav in-portal.
- Tutorials & how-tos: **Applicable — this IS the tutorials deliverable (§10.1); how-tos are C3.**

## 9. Acceptance Criteria
- AC-C2-01 (REQ-C2-01): Every §10.1 tutorial exists, is learning-oriented, and a first-time user
  following it end-to-end reaches the stated working result.
- AC-C2-02 (REQ-C2-02): No tutorial uses raw LangGraph; every tutorial's agent uses Deep Agents
  (`Agent.sdk` = `deep-agents`).
- AC-C2-03 (REQ-C2-03): Every tutorial contains prerequisites, ordered steps, per-step expected
  output, and a verifiable end state.
- AC-C2-04 (REQ-C2-04): Every tutorial's embedded test expectations are signed off before the
  documented component's implementation begins and are used as its acceptance tests.
- AC-C2-05 (REQ-C2-05): Every tutorial with an un-implemented dependency includes mock-out
  instructions, runs green against the mock, and runs green unchanged against the real dependency.
- AC-C2-06 (REQ-C2-06): A Canon-name lint over the tutorial corpus finds zero non-Canon CRD/product
  names except those explicitly tagged `[PROPOSED — not in source]`.
- AC-C2-07 (REQ-C2-07): A tutorial PR merges through the standard docs-as-code path and passes the
  C1 GitHub Actions docs check.
- AC-C2-08 (REQ-C2-08): C8 ingests the tutorial Markdown and indexes it into
  `platform-knowledge-base` with no portal-specific preprocessing (verified jointly with C8).
- AC-C2-09 (REQ-C2-09): No tutorial references an unsupported CI system; CI steps point at the
  GitHub Actions reference pipeline.

## 10. Risks & Open Questions
- R1 (med): Tutorials are authored before the components they exercise (§10). Open question: how to
  keep mock-out instructions in sync as upstream interfaces firm up — mitigated by the docs-before-
  implementation sign-off gate and re-running the same tests when the real dependency lands.
- R2 (med): Concrete Platform/agent SDK method signatures are **not specified in source**
  (interface-contract §3.1/§3.2). Any method a tutorial needs beyond the named surface groups is
  `[PROPOSED — not in source]` until B6/B7 publish it; blast radius bounded to affected tutorials.
- R3 (low): Per-event-type CloudEvent names for the event-trigger tutorial are deferred to B12
  (interface-contract §2/§6). Tutorial references the namespace path conceptually; no invented names.
- R4 (low): Tutorial folder naming depends on C1's `[PROPOSED]` directory layout (spec-C1 R1); C2
  adopts C1's published layout verbatim.

## 11. References
- architecture-overview.md §10 Documentation plan (~L1425), docs-timing (~L1431),
  docs-before-implementation + mock-out (~L1433).
- architecture-overview.md §10.1 Tutorials, Deep-Agents-only + tutorial list (~L1435–1451).
- architecture-overview.md §14.3 Workstream C table, C2 row (~L1725); C8 row (~L1731).
- architecture-overview.md §6.4 Knowledge Base as a separate primitive (`platform-knowledge-base`
  RAGStore).
- ADR 0019 (LangGraph + Deep Agents; Deep Agents default for tutorials), ADR 0008 (Material for
  MkDocs), ADR 0011 (three-layer testing / CLI orchestration), ADR 0010 / 0033 (GitHub Actions /
  AWS + GitHub), ADR 0024 (vendor docs separate).
- _meta/glossary.md (Langchain Deep Agents, LangGraph, `Agent`, `MCPServer`, `Skill`, `Evaluation`,
  `RAGStore`, `platform-knowledge-base`, ARK, LibreChat, A2A).
- _meta/interface-contract.md §1.2 (`Agent.sdk`), §3.1/§3.2 (SDK surfaces), §2 (CloudEvent taxonomy).
- spec-C1.md (portal infrastructure, navigation slots, conventions, contribution workflow).
