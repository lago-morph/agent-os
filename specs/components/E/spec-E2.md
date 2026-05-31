# SPEC E2 — Developer training track [PROPOSED]

> kind: COMPONENT · workstream: E · tier: T2
> upstream: [] · downstream: [] · adrs: [] · views: []
> canon-glossary: FROZEN · canon-interface: FROZEN

## 1. Purpose & Problem Statement

The Developer training track is the curriculum that takes an agent developer from zero to production-ready development on the Agentic Execution Platform. Per §12, documentation is necessary but not sufficient: developers need a curriculum that composes the docs into pedagogical order and pairs each topic with hands-on exercises in a sandboxed lab environment. This piece owns the developer-facing modules, their labs, working starter projects, exercises with checks, and knowledge checks.

The problem it solves: a developer who has read the SDK and capability docs (Workstream C, plus the per-product docs and how-tos of Workstreams A and B) still cannot reliably author, test, evaluate, debug, and roll out a Platform Agent without guided, graded practice. E2 builds that competence by sequencing the developer topics enumerated in §12.2 (platform overview → first agent declarative → first agent in code → skills → tools/MCP/A2A → memory/RAG/knowledge base → triggers → evaluation/testing → A/B testing and rollout → debugging with traces → cost optimization → multi-agent patterns → capability sets) into thirteen modules, each with a sandboxed lab namespace, a working starter project, exercises with checks, and a knowledge check that reuses the platform itself (§12.2).

E2 is a consumer-tier component (§14, §14.5): it continuously consumes the outputs of Workstreams A and B (the running platform and its SDKs) and Workstream C (docs), and is built after those outputs exist.

## 2. Scope

### 2.1 In scope

- Thirteen developer training modules mapped one-to-one to the §12.2 topic list.
- A sandboxed lab namespace per module that reuses the running platform (§12, §12.2: "Labs reuse the platform itself").
- A working starter project per module (the §12.2 per-module structure: "a sandboxed lab namespace, a working starter project, exercises with checks, and a knowledge check").
- Exercises with automated checks and a knowledge check per module.
- Defined learning outcomes per module, expressed as binary competencies a developer demonstrates.
- Module-to-docs linkage: each module references and links into the Workstream C docs (and the per-product how-tos of A/B) rather than restating them (§12).

### 2.2 Out of scope (and where it lives instead — name the owning piece)

- Authoring the SDK, agent SDK, or capability docs — owned by Workstream C (C1–C9) and the per-product docs/how-tos of the Platform SDK (B6), the agent SDK (B7), and the capability components (§14.1, §14.2, §14.3).
- The Platform SDK and agent SDK themselves — owned by B6 (Platform SDK Python + TypeScript) and B7 (LangGraph + Deep Agents).
- The `agent-platform` CLI and test framework — owned by B9 and B14; E2's evaluation/testing module exercises them, it does not build them.
- The Operator training track — owned by E1 (§14.5).
- Developer dashboards themselves — owned by D2 (§11.2, §14.4); E2's debugging and cost modules consume them.
- Building the platform components the labs exercise — owned by Workstreams A and B.

## 3. Context & Dependencies

Upstream pieces consumed: none are hard build-time CRD dependencies (the CSV lists no upstream/downstream for E2). As a consumer-tier piece (§14, §14.5), E2 consumes two continuous inputs:
- Workstream C docs (C1–C9) — modules link into tutorials, how-tos, reference, and explanation; E2 composes them into pedagogical order (§12).
- The running platform (Workstreams A and B) — labs reuse the platform itself (§12), so each module's lab namespace runs against live platform components and SDKs (LiteLLM gateway A1, Langfuse A2, ARK A5, agent-sandbox/Envoy A6, OPA A7, Letta A10, OpenSearch A11, the Platform SDK B6, the agent SDK B7, the `agent-platform` CLI B9, the test framework B14, the Knowledge Base RAGStore, etc.).

Downstream consumers: none (the CSV lists no downstream; E2 is terminal).

ADR decisions honored: no ADR is directly binding on a training curriculum. Labs that exercise platform behavior inherit the constraints of the components they run against (e.g. LangGraph + Deep Agents as the v1.0 agent SDK per ADR 0019, CapabilitySet layering per ADR 0013/0032, three-layer testing per ADR 0011, memory access modes per ADR 0025, synthetic MCP from OpenAPI per ADR 0020) but E2 imposes no new architectural decisions. `N/A — no ADR governs the training curriculum itself; labs inherit the constraints of the components they exercise.`

## 4. Interfaces & Contracts

E2 is curriculum content plus lab environments and starter projects; it defines no platform CRDs, APIs, or CloudEvents of its own. Lab namespaces are Approved namespaces (Canon) provisioned the same way any tenant namespace is.

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)

N/A — E2 defines no CRDs or XRDs. Labs apply existing Canon CRDs (`Agent`, `AgentRun`, `Team`, `Tool`, `Skill`, `MCPServer`, `A2APeer`, `RAGStore`, `Memory`, `CapabilitySet`, `Evaluation`, `Sandbox`, etc.) as authored by their owning components; E2 authors no new schema.

### 4.2 APIs / SDK surfaces

N/A — E2 exposes no API or SDK surface. Developer labs drive existing surfaces (the Platform SDK from B6, the agent SDK from B7, the `agent-platform` CLI from B9, the test framework from B14, LibreChat from A8) through guided exercises and starter projects; E2 adds no new surface.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)

N/A — E2 emits and consumes no CloudEvents of its own. Lab exercises may observe existing platform events (e.g. `platform.lifecycle.*`, `platform.evaluation.*`, `platform.audit.*`) as part of the triggers, evaluation, and debugging modules, but those events are emitted by the components under exercise, not by E2; `platform.evaluation.*` is owned by B14 (test & eval framework) and E2 is at most a consumer/dependent, never an owner.

### 4.4 Data schemas / connection-secret contracts

N/A — E2 defines no data schema or connection-secret contract. Lab namespaces and starter projects consume connection secrets in the uniform substrate shape (Canon: connection secret) produced by the components they exercise (e.g. memory backends, RAG stores).

## 5. OSS-vs-Custom Decision

Custom (build-new content): the curriculum, module structure, starter projects, exercises, checks, and lab definitions are authored content specific to this platform — no upstream project supplies them. Lab environments are not new software: they reuse the platform itself (§12) and standard Approved namespaces, so there is no fork/wrap of any OSS project. Knowledge-check and exercise-check tooling reuses the existing `agent-platform` CLI (B9) and the three-layer test runners (B14) where automated verification is needed, rather than introducing a new framework. Starter projects are scaffolded with the existing agent SDK (B7, LangGraph + Deep Agents per ADR 0019) and Platform SDK (B6).

## 6. Functional Requirements

- REQ-E2-01: The track SHALL provide exactly thirteen modules, one per topic in §12.2 (platform overview — what an agent is in our model; first agent — declarative; first agent in code — BYO with the SDK; skills — write, test, version; tools, MCP, A2A — including synthetic MCP from OpenAPI; memory, RAG, and the knowledge base; triggers — cron, event, long-running, interactive; evaluation and testing — Langfuse datasets, promptfoo, three-layer tests; A/B testing and rollout; debugging with traces; cost optimization; multi-agent patterns — A2A, Teams; capability sets — composing layered profiles).
- REQ-E2-02: Each module SHALL declare its module coverage — the named platform components, SDK surfaces, and docs (Workstream A/B/C pieces) it covers — so the union of modules covers the developer-facing surface enumerated in §12.2.
- REQ-E2-03: The "first agent — declarative" module (§12.2 item 2) and the "first agent in code — BYO with the SDK" module (item 3) SHALL each ship a working starter project: a declarative `Agent` CRD project and an SDK-based (B6/B7) project respectively.
- REQ-E2-04: Each module SHALL include a sandboxed lab namespace that reuses the running platform (§12, §12.2), provisioned as an Approved namespace isolated per the platform's namespace multi-tenancy model.
- REQ-E2-05: Each module SHALL include a working starter project from which the exercises begin (§12.2 per-module structure).
- REQ-E2-06: Each module SHALL include exercises with automated checks that verify the developer produced the required artifact or behavior (e.g. agent runs, skill versioned, MCP tool invoked, eval passes, A/B rollout promoted).
- REQ-E2-07: Each module SHALL include a knowledge check that gates module completion.
- REQ-E2-08: Each module SHALL declare explicit learning outcomes as binary, demonstrable developer competencies, and the exercises/checks SHALL map to those outcomes.
- REQ-E2-09: Modules SHALL link into the Workstream C docs (tutorials, how-tos, reference, explanation) and the per-product how-tos of B6/B7 rather than restating them, composing them into pedagogical order (§12).
- REQ-E2-10: The tools/MCP/A2A module (§12.2 item 5) SHALL exercise synthetic MCP from OpenAPI (the `SyntheticMCPServer` / A12 converter path per ADR 0020) and an A2A call between Platform Agents in the lab namespace.
- REQ-E2-11: The evaluation-and-testing module (§12.2 item 8) SHALL exercise Langfuse datasets, promptfoo, and the three-layer tests (Chainsaw/Playwright/PyTest) via the `agent-platform test` CLI (B9/B14), honoring promptfoo's evaluation-not-fourth-layer status (ADR 0011).
- REQ-E2-12: The capability-sets module (§12.2 item 13) SHALL exercise composing layered `CapabilitySet` profiles using the overlay add-if-not-there / replace-if-there semantics (Canon / ADR 0032).

## 7. Non-Functional Requirements

- Security / multi-tenancy (§6.9): lab namespaces SHALL be isolated Approved namespaces enforced by the platform's namespace + RBAC + OPA model (ADR 0016), so a trainee's lab cannot affect another trainee's lab or production tenants. Starter-project agents SHALL run in the sandbox (A6) and reach capabilities only through approved CRDs (Canon: Approved capability).
- Observability (§6.5): debugging and cost-optimization exercises SHALL use the real developer dashboards (D2), Langfuse traces (A2), and Tempo (A13) correlated by `trace_id` (ADR 0015) without a parallel training-only observability stack; lab activity is auditable like any platform activity (ADR 0034).
- Scale: the track SHALL support concurrent trainees by provisioning one isolated lab namespace per trainee per module; lab teardown SHALL reclaim resources.
- Versioning (ADR 0030): modules SHALL pin the platform/SDK/docs version they target and SHALL be revised when the components they cover change CRD/API/SDK versions, so a module never teaches a retired surface (e.g. the `Agent.sdk` value set, the CapabilitySet overlay semantics).

## 8. Cross-Cutting Deliverable Checklist

E2 is a consumer-tier (Workstream E) content component, not a Workstream A install/operate package; the §14.1 standard set is largely N/A. Each item is marked applicable / N/A below.

- Helm/manifests: N/A — E2 ships curriculum content + lab definitions + starter projects, not a deployed product. Lab namespaces are provisioned via existing tenant-onboarding mechanisms (A21).
- Per-product docs (10.5): N/A — product docs are owned by Workstream A/B/C; E2 consumes them.
- Runbook (10.7): N/A — E2 references docs/how-tos rather than authoring operator runbooks.
- Alerts: N/A — alert rules are owned by the Workstream A components; E2 contributes none.
- Grafana dashboard (Crossplane XR): N/A — developer dashboards owned by D2 and per-component dashboards by Workstream A; E2 consumes them in the debugging/cost modules.
- Headlamp plugin: N/A — E2 ships no plugin; labs use existing Headlamp UI (A9/A22).
- OPA/Rego integration: N/A — E2 contributes no policy; lab namespace isolation reuses existing OPA enforcement.
- Audit emission (ADR 0034): N/A — E2 emits no events of its own; lab actions are audited by the components under exercise.
- Knative trigger flow: N/A — E2 defines no event flows; the triggers module exercises existing flows (A4/B8/B12).
- HolmesGPT toolset: N/A — E2 contributes no toolset (HolmesGPT is an operator concern, exercised in E1 module 11).
- 3-layer tests (Chainsaw/Playwright/PyTest): applicable — exercise checks, starter-project validity, and lab-validity tests reuse the three-layer runners (Chainsaw for CRD/lab-state assertions, Playwright for LibreChat/dashboard UI exercises, PyTest for SDK and check logic) via the B9/B14 CLI; the evaluation module additionally teaches these layers.
- Tutorials & how-tos: N/A (as authored docs) — E2 links into C2/C3 tutorials and how-tos and the B6/B7 how-tos; it does not author the Diataxis content, it sequences it.

## 9. Acceptance Criteria

- AC-E2-01: The track contains thirteen modules whose titles map one-to-one to §12.2's thirteen topics. (→ REQ-E2-01)
- AC-E2-02: Each module lists the named components/SDK surfaces/docs it covers, and the union covers every §12.2 topic with no topic uncovered. (→ REQ-E2-02)
- AC-E2-03: Modules 2 and 3 each ship a runnable starter project — a declarative `Agent` project and an SDK (B6/B7) project respectively. (→ REQ-E2-03)
- AC-E2-04: Each module provisions an isolated lab namespace that runs against live platform components. (→ REQ-E2-04)
- AC-E2-05: Each module boots from a documented, runnable working starter project. (→ REQ-E2-05)
- AC-E2-06: Each module's exercises have automated checks that pass only when the required developer artifact/behavior was produced. (→ REQ-E2-06)
- AC-E2-07: Each module has a knowledge check that must pass to mark the module complete. (→ REQ-E2-07)
- AC-E2-08: Each module declares binary learning outcomes, and every exercise/check traces to a stated outcome. (→ REQ-E2-08)
- AC-E2-09: Every module links to at least one Workstream C doc (or B6/B7 how-to) and restates no doc content verbatim. (→ REQ-E2-09)
- AC-E2-10: The tools/MCP/A2A module's lab generates a synthetic MCP server from an OpenAPI spec and performs an A2A call between two Platform Agents, with checks passing on both. (→ REQ-E2-10)
- AC-E2-11: The evaluation module's lab runs a Langfuse dataset eval, a promptfoo eval, and a three-layer `agent-platform test` run, with checks passing on each. (→ REQ-E2-11)
- AC-E2-12: The capability-sets module's lab composes a layered `CapabilitySet` and the check verifies the resolved capability set reflects add-if-not-there / replace-if-there semantics. (→ REQ-E2-12)

## 10. Risks & Open Questions

- Risk (med): docs/SDK drift — if Workstream C docs or the B6/B7 SDKs change, module links, starter projects, and composed order rot. Mitigation: version-pin per REQ-E2-08/§7 and revalidate on doc/SDK release. Reconciliation: E2 links rather than restates (REQ-E2-09) to minimize duplicated drift surface; starter projects are CI-validated (§8 three-layer tests).
- Risk (med): lab provisioning load — concurrent trainees each needing an isolated namespace per module could strain the cluster. Mitigation: per-trainee teardown (§7 scale); cap concurrent labs.
- Risk (low): starter projects bit-rot as the agent SDK evolves. Mitigation: starter-project validity is a tested deliverable (§8) gated in CI on each SDK release.
- Open question (low): does the "first agent in code" module target both Platform SDK languages (Python + TypeScript, B6) or Python first? §12.2 item 3 says "BYO with the SDK" without specifying. `[PROPOSED]` resolution: Python-first starter project with a TypeScript variant as a stretch exercise; confirm with the SDK owner (B6).
- Open question (low): how are knowledge-check pass thresholds and re-attempts governed? Not specified in §12; `[PROPOSED]` to be set by the training owner (consistent with E1).
- Open question (low): does the A/B testing and rollout module (item 9) require a live multi-Stage Kargo trajectory, or can it run against the v1.0 single Stage? Per ADR 0040 the Stage list is evolving; `[PROPOSED]` resolution: exercise A/B at the prompt/agent-version level (D2 comparison) on the single Stage, deferring multi-Stage promotion drills to when additional Stages land.

## 11. References

- architecture-overview.md §12 Training (~line 1577) — curriculum rationale; "Labs reuse the platform itself".
- architecture-overview.md §12.2 Developer track (~line 1595) — the thirteen-topic source list and the per-module structure (lab namespace, working starter project, exercises with checks, knowledge check).
- architecture-overview.md §14 / §14.5 Workstream E (~line 1744) — consumer-tier positioning; E2 = "Developer training track — modules with labs".
- architecture-overview.md §11.2 / §14.4 — developer dashboards (D2) consumed by the debugging/cost modules.
- architecture-overview.md §13 (~line 1613) — three-layer testing + promptfoo status exercised by the evaluation module.
- ADR 0019 (agent SDK), ADR 0013/0032 (CapabilitySet + overlay), ADR 0011 (three-layer testing + promptfoo), ADR 0020 (synthetic MCP from OpenAPI), ADR 0015 (Tempo + Langfuse trace correlation), ADR 0025 (memory access modes).
- piece-index.csv — E2 row: tier T2, wave consumer, no upstream/downstream.
- _meta/glossary.md — Canon terms (Platform Agent, A2A, MCP, CapabilitySet, Knowledge Base, Approved capability, Approved namespace, connection secret, CRDs, LangGraph, Langchain Deep Agents, LiteLLM, Langfuse, Letta, OpenSearch).
