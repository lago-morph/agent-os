# SPEC C5 — Diataxis explanation

> kind: COMPONENT · workstream: C · tier: T1
> upstream: [C1] · downstream: [C8] · adrs: [0001, 0002, 0003, 0004, 0006, 0008, 0012, 0013, 0016, 0018, 0024, 0025, 0032] · views: [6.1, 6.2, 6.3, 6.4, 6.6, 6.9, 6.10]
> canon-glossary: see _meta/glossary.md · canon-interface: see _meta/interface-contract.md

## 1. Purpose & Problem Statement

Explanation is the **understanding-oriented** Diataxis quadrant (§10.4): discursive, design-rationale
content that helps a reader understand *why* the platform is built the way it is. It does not give
steps (how-to) or enumerate fields (reference) — it provides the conceptual backbone, drawing
directly on the architecture views (§6) and the ADRs that record each decision. It is the natural
home for the "why X over Y" narratives so the rationale lives next to the platform rather than only
in ADRs.

C5 owns the authored explanation corpus that fills the explanation slot defined by C1. Each
explanation article is anchored to the relevant architecture view and ADR, so it stays faithful to
Canon rather than re-deriving rationale. Without it, the platform has steps and field lists but no
authored conceptual layer, and the `platform-knowledge-base` `RAGStore` (§6.4) is missing the
understanding-oriented corpus that newcomers and Platform Agents draw on to reason about the system.

## 2. Scope

### 2.1 In scope
- The full set of §10.4 explanation articles, each understanding-oriented and anchored to its view +
  ADR:
  - Why this architecture — design principles (§6.1).
  - How sandboxing works — defense in depth (agent-sandbox + Envoy egress, ADR 0003; §6.2).
  - How the gateway controls cost (LiteLLM + `BudgetPolicy`; §6.1).
  - How memory works across agents — namespaces, sharing, lifecycle (`Memory`/`MemoryStore` access
    modes, ADR 0025; §6.3).
  - Trade-offs between memory backends (Letta, ADR 0005; §6.3).
  - Why ARK as the agent operator (ADR 0001; §6.2).
  - Why NATS JetStream as the broker (ADR 0004; §6.7).
  - Why Envoy egress rather than CNI L7 policy (ADR 0003; §6.6).
  - Why a Python kopf operator rather than a Crossplane provider for LiteLLM (ADR 0006; §6.8).
  - Multi-tenancy story (namespaces + RBAC-floor / OPA-restrictor, ADR 0016 / 0018; §6.9).
  - CapabilitySet layering semantics (`CapabilitySet` overlay, ADR 0013 / 0032; §6.8).
  - The platform self-management model with HolmesGPT (ADR 0012; §6.10).
- Uniform explanation structure: the problem/tension, the decision, the trade-offs considered, and
  the consequences — each cross-citing the anchoring §6 view and ADR(s).
- Faithful restatement of Canon rationale: explanations SHALL track the cited ADRs and views and
  SHALL NOT introduce decisions or rationale absent from source; anything beyond source is tagged
  `[PROPOSED — not in source]`.

### 2.2 Out of scope (and where it lives instead)
- Portal infrastructure, nav skeleton, search, contribution-workflow check — **C1**.
- Learning-oriented tutorials — **C2**. Problem-oriented how-tos — **C3**. Information-oriented
  reference — **C4**.
- The architecture views and ADRs themselves — authored upstream (the `architecture-overview.md` §6
  views and the `adr/` set); C5 explains them for a reader, it does not amend them.
- Per-product docs (§10.5) / per-product runbooks (§10.7) — **Workstream A** owners. Cross-cutting
  runbooks — **C6**. Maintainer/extender docs — **C7**.
- The RAG **indexing pipeline** — **C8** (C5 only emits re-indexable Markdown).

## 3. Context & Dependencies

Upstream consumed: **C1** — portal infrastructure, explanation navigation slot, authoring
conventions, contribution-workflow check.

Continuous (non-blocking) inputs: the **architecture views (§6.1–§6.10)** and the **ADR set** are
the source material each article restates; as views/ADRs are revised, the affected articles are
re-aligned. The components whose design each article explains (A1 LiteLLM, A5 ARK, A6 agent-sandbox +
Envoy, A4 Knative + NATS, A10 Letta, A14 HolmesGPT, B13 kopf operator, OPA/Gatekeeper) are consumed
as conceptual subjects, not as code dependencies. Recorded as continuous inputs, not blockers.

Downstream consumers: **C8** (indexes the explanation Markdown into `platform-knowledge-base`).

ADR decisions honored (each article anchors to its ADR(s)):
- **ADR 0001** (ARK), **0002 / 0018** (OPA + Gatekeeper; RBAC-floor / OPA-restrictor), **0003**
  (Envoy egress vs CNI L7), **0004** (NATS JetStream broker), **0006** (Python kopf operator vs
  Crossplane provider), **0012** (HolmesGPT self-management), **0013 / 0032** (CapabilitySet
  layering / overlay), **0016** (multi-tenancy via namespaces), **0025** (memory access modes),
  **0005** (Letta memory backend).
- **ADR 0008** — docs-as-code Markdown-in-repo, PR review, re-indexable by C8.
- **ADR 0024** — explanations cover the platform's own rationale; vendor docs are the companion
  project.

## 4. Interfaces & Contracts

Use ONLY Canon names. C5 references (does not define): ARK, LiteLLM, agent-sandbox, Envoy egress
proxy, NATS JetStream, Letta, HolmesGPT, OPA / Gatekeeper, `CapabilitySet`, `Memory`, `MemoryStore`,
`BudgetPolicy`, RBAC-as-floor / OPA-as-restrictor.

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
N/A — C5 defines no CRD/XRD. It explains the design intent behind Canon CRDs/XRs (`CapabilitySet`,
`Memory`, `MemoryStore`, `BudgetPolicy`) at a conceptual level; field-level detail is C4 reference.

### 4.2 APIs / SDK surfaces
N/A — C5 exposes and documents no API/SDK surface; it explains architectural rationale, not
interfaces.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
N/A — C5 emits no CloudEvents. The broker article (NATS JetStream / Knative) explains the eventing
*model* (§6.7); the ten-namespace taxonomy detail is C4 reference, owned by B12.

### 4.4 Data schemas / connection-secret contracts
N/A — C5 writes no data/connection-secret. The only contract is the re-indexable Markdown corpus
shape consumed by C8 (co-owned with C1/C8).

## 5. OSS-vs-Custom Decision
N/A — C5 is authored content, not software. It consumes the **Material for MkDocs** portal
(configured by C1, ADR 0008) and authors explanation Markdown restating Canon rationale. No OSS
project is forked, wrapped, or built.

## 6. Functional Requirements
- REQ-C5-01: C5 SHALL author every explanation article enumerated in §10.4, each
  understanding-oriented (rationale, not steps or field lists).
- REQ-C5-02: Every article SHALL anchor to and cite its relevant architecture view (§6.x) and ADR(s),
  restating their rationale faithfully without introducing decisions absent from source.
- REQ-C5-03: Every article SHALL follow a uniform structure — the tension/problem, the decision, the
  trade-offs considered, and the consequences — authored against C1's conventions and navigation slot.
- REQ-C5-04: Articles SHALL use ONLY Canon names; any concept or rationale beyond source SHALL be
  tagged `[PROPOSED — not in source]`.
- REQ-C5-05: The multi-tenancy article SHALL present the namespaces + RBAC-as-floor /
  OPA-as-restrictor model (ADR 0016 / 0018) and the CapabilitySet-layering article SHALL present the
  overlay semantics (ADR 0013 / 0032) consistent with Canon.
- REQ-C5-06: Explanation Markdown SHALL be docs-as-code (in-repo, PR-reviewed, passing the C1 check)
  and conform to the clean-Markdown corpus contract so C8 can re-index it without preprocessing.
- REQ-C5-07: Explanation articles SHALL cross-link to the corresponding reference (C4) and how-to
  (C3) where a reader moving from "why" to "what"/"how" is the expected path.

## 7. Non-Functional Requirements
- Security: the sandboxing, egress, and multi-tenancy articles explain the defense-in-depth and
  RBAC-floor / OPA-restrictor posture (ADR 0003 / 0016 / 0018) faithfully; no secrets in content.
- Multi-tenancy (§6.9): explained as a first-class subject (the multi-tenancy article); the corpus
  itself is platform-wide reference content carrying no tenant data.
- Observability (§6.5): the self-management article explains HolmesGPT's observe-and-remediate model
  (ADR 0012) at a conceptual level; build status visible via the C1 GitHub Actions docs check.
- Scale: corpus grows with the article set; per-version builds are additive (C1).
- Versioning (ADR 0030): explanation articles are revised when their anchoring ADR/view changes;
  per-version builds align them to the platform version they describe.

## 8. Cross-Cutting Deliverable Checklist
- Helm/manifests: **N/A — authored content; build/deploy owned by C1.**
- Per-product docs (10.5): **N/A — Workstream A; C5 authors Diataxis explanation only.**
- Runbook (10.7): **N/A — runbooks are C6 + Workstream A.**
- Alerts: **N/A — content piece, no runtime failure conditions.**
- Grafana dashboard (Crossplane XR): **N/A — no runtime metrics.**
- Headlamp plugin: **N/A — no UI surface (portal-vs-Headlamp linking deferred, backlog §1.15).**
- OPA/Rego integration: **N/A — C5 explains the policy model; policies are B3/B16.**
- Audit emission (ADR 0034): **N/A — no audit-relevant runtime actions.**
- Knative trigger flow: **N/A — re-index triggering is C8; the broker article only explains the model.**
- HolmesGPT toolset: **N/A — C5 explains the self-management model; toolsets ship with their components.**
- 3-layer tests (Chainsaw/Playwright/PyTest): **Partial** — Playwright validates in-portal
  render/nav/cross-links; PyTest validates Canon-name lint + view/ADR-citation presence; Chainsaw
  **N/A — no CRD**.
- Tutorials & how-tos: **N/A — C5 is the explanation quadrant (§10.4); tutorials=C2, how-tos=C3.**

## 9. Acceptance Criteria
- AC-C5-01 (REQ-C5-01): Every §10.4 explanation article exists and is understanding-oriented (no
  step-by-step instructions, no field enumerations).
- AC-C5-02 (REQ-C5-02): Every article cites its anchoring §6 view and ADR(s); a review confirms it
  introduces no decision or rationale absent from those sources.
- AC-C5-03 (REQ-C5-03): Every article contains the tension, the decision, the trade-offs, and the
  consequences.
- AC-C5-04 (REQ-C5-04): A Canon-name lint finds zero non-Canon names except those tagged
  `[PROPOSED — not in source]`.
- AC-C5-05 (REQ-C5-05): The multi-tenancy article presents the namespaces + RBAC-floor /
  OPA-restrictor model and the layering article presents the CapabilitySet overlay semantics,
  consistent with ADR 0016 / 0018 / 0013 / 0032.
- AC-C5-06 (REQ-C5-06): An explanation PR passes the C1 docs check and C8 indexes the output into
  `platform-knowledge-base` with no preprocessing (verified with C8).
- AC-C5-07 (REQ-C5-07): Each article cross-links to the corresponding C4 reference and/or C3 how-to
  where the "why → what/how" path is expected.

## 10. Risks & Open Questions
- R1 (med): Explanation rationale must stay faithful as ADRs/views are revised. Open question: how
  articles are re-aligned on ADR change — mitigated by per-article ADR/view anchoring (AC-C5-02) and
  per-version builds (C1).
- R2 (low): The CapabilitySet-layering article depends on overlay semantics that are partly deferred
  to ADR 0032 + design specs (interface-contract §6). Article presents the source-stated model and
  marks deferred detail as such; no invented semantics.
- R3 (low): Cross-links to C3/C4 (AC-C5-07) depend on those quadrants' page paths existing —
  sequencing risk only; resolved as C3/C4 land within the same wave.
- R4 (low): Explanation folder naming depends on C1's `[PROPOSED]` directory layout (spec-C1 R1); C5
  adopts C1's published layout verbatim.

## 11. References
- architecture-overview.md §10 Documentation plan (~L1425).
- architecture-overview.md §10.4 Explanation, full list (~L1489–1502).
- architecture-overview.md §14.3 Workstream C table, C5 row (~L1728).
- architecture-overview.md §6 views (§6.1 principles, §6.2 agent runtime, §6.3 memory/data, §6.4
  Knowledge Base, §6.6 security/policy, §6.7 eventing, §6.8 capability registries, §6.9
  multi-tenancy, §6.10 HolmesGPT self-management).
- ADR 0001 (ARK), 0002/0018 (OPA / RBAC-floor), 0003 (Envoy egress), 0004 (NATS JetStream), 0005
  (Letta), 0006 (kopf operator), 0012 (HolmesGPT), 0013/0032 (CapabilitySet layering/overlay), 0016
  (multi-tenancy), 0025 (memory access modes), 0008 (Material for MkDocs), 0024 (vendor docs separate).
- _meta/glossary.md (ARK, LiteLLM, agent-sandbox, Envoy egress proxy, NATS JetStream, Letta,
  HolmesGPT, OPA / Gatekeeper, `CapabilitySet`, `Memory`, `MemoryStore`, RBAC-as-floor /
  OPA-as-restrictor).
- _meta/interface-contract.md §1 (CRDs referenced conceptually), §2 (eventing model), §6 (deferred
  overlay semantics).
- spec-C1.md (portal infrastructure, navigation slots, conventions, contribution workflow).
