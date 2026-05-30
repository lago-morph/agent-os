# SPEC C4 — Diataxis reference

> kind: COMPONENT · workstream: C · tier: T1
> upstream: [C1] · downstream: [C8] · adrs: [0019, 0030, 0031, 0034, 0013, 0041, 0008, 0024, 0007] · views: [6.4]
> canon-glossary: see _meta/glossary.md · canon-interface: see _meta/interface-contract.md

## 1. Purpose & Problem Statement

Reference is the **information-oriented** Diataxis quadrant (§10.3): dry, complete, accurate
descriptions of the platform's surfaces — CRDs, XRs, APIs, schemas, catalogs — that a reader
consults to look up an exact fact. It does not teach or persuade; it enumerates. Reference docs
**cover both LangGraph and Langchain Deep Agents** (ADR 0019): LangGraph as the supported v1.0 SDK
surface and Deep Agents as the opinionated layer on top — readers using tutorials see Deep Agents,
readers needing the underlying graph primitives find them here.

C4 owns the authored reference corpus that fills the reference slot defined by C1. Because reference
material spans many components, it is scheduled with the **last component in each flow** (§10) while
partial reference content is encouraged from earlier components. Without complete, accurate,
version-pinned reference, developers and Platform Agents cannot resolve exact field names, schemas,
and command surfaces, and the `platform-knowledge-base` `RAGStore` (§6.4) is missing the
information-oriented corpus.

## 2. Scope

### 2.1 In scope
- The full set of §10.3 reference documents, each information-oriented and complete:
  - ARK CRD reference (`Agent`, `AgentRun`, `Team`, `Tool`, `Memory`, `Evaluation`, `Query`).
  - Sandbox CRD reference (`Sandbox`, `SandboxTemplate`).
  - Capability CRD reference (`MCPServer`, `A2APeer`, `RAGStore`, `EgressTarget`, `Skill`,
    `CapabilitySet`; also `VirtualKey`, `BudgetPolicy`).
  - Crossplane XR catalog (`MemoryStore`, `AgentEnvironment`, `SyntheticMCPServer`,
    `GrafanaDashboard`, `AuditLog`, `TenantOnboarding`, `XAgentDatabase`, `XPostgres`,
    `XSearchIndex`, `XObjectStore`, `XMongoDocStore`).
  - LiteLLM gateway and admin API reference.
  - Platform SDK reference (Python, TypeScript) — the named surfaces `memory.*`, `rag.*`, OTel
    emission, A2A registration helpers.
  - OPA policy library reference.
  - CloudEvent schema registry reference (the ten `platform.*` top-level namespaces, ADR 0031).
  - Audit event schema reference (ADR 0034).
  - Metrics catalog. Trace span attribute reference.
  - Headlamp plugin extension points used. CI CLI command reference (`agent-platform`).
  - LibreChat configuration reference (the locked-down profile, ADR 0007).
  - Langfuse dataset and evaluator schemas.
- Coverage of **both LangGraph and Langchain Deep Agents** SDK surfaces (ADR 0019).
- Field-by-field accuracy against Canon: every documented CRD/XRD field, event namespace,
  connection-secret field, and CLI subcommand SHALL match `_meta/interface-contract.md` exactly;
  any field/method not source-stated SHALL be tagged `[PROPOSED — not in source]` or recorded as
  "not specified in source — deferred to <owning component>" rather than invented.
- Version pinning per ADR 0030 via C1's per-version builds.

### 2.2 Out of scope (and where it lives instead)
- Portal infrastructure, nav skeleton, search, contribution-workflow check — **C1**.
- Learning-oriented tutorials — **C2**. Problem-oriented how-tos — **C3**. Understanding-oriented
  explanation — **C5**.
- The **authoritative source of truth** for each surface — owned by the component that owns the
  reconciler/API (e.g. ARK CRD shapes = A5, capability CRDs = B13, XRs = B4, CLI = B9, CloudEvent
  schemas = B12, audit schema = A18). C4 documents these surfaces; it does not define them.
- Per-product docs (§10.5) / per-product runbooks (§10.7) — **Workstream A** owners. Cross-cutting
  runbooks — **C6**. Maintainer/extender docs — **C7**.
- The RAG **indexing pipeline** — **C8** (C4 only emits re-indexable Markdown).
- Concrete CloudEvent per-event-type schemas, SDK method signatures, connection-secret per-primitive
  fields, `Tool`/`Query` field sets — deferred to owning components per interface-contract §6; C4
  documents what is published and records the rest as deferred, not invented.

## 3. Context & Dependencies

Upstream consumed: **C1** — portal infrastructure, reference navigation slot, authoring conventions,
contribution-workflow check.

Continuous (non-blocking) inputs: the surface-owning components whose interfaces C4 documents —
**A5** (ARK CRDs), **A6** (Sandbox CRDs), **B13** (capability CRDs), **B4** (Crossplane XRs/XRDs),
**A1** (LiteLLM gateway + admin API), **B6** (Platform SDK), **B16/B3** (OPA policy library), **B12**
(CloudEvent schema registry), **A18** (audit event schema), **A13** (metrics catalog, trace span
attributes), **A9** (Headlamp extension points), **B9** (CLI command reference), **A8** (LibreChat
config, ADR 0007), **A2** (Langfuse dataset/evaluator schemas), **B7** (LangGraph + Deep Agents SDK
surfaces). C4 finalizes per flow as the last component lands; partial reference is contributed
earlier. Recorded as continuous inputs, not blockers.

Downstream consumers: **C8** (indexes the reference Markdown into `platform-knowledge-base`).

ADR decisions honored:
- **ADR 0019** — reference covers both LangGraph (supported v1.0 SDK) and Deep Agents (opinionated
  layer); `Agent.sdk` accepts `langgraph` and `deep-agents`.
- **ADR 0030** — CRD/API/event/SDK/CLI versioning policy; reference is version-pinned via C1
  per-version builds and documents per-component versioning ownership.
- **ADR 0031** — CloudEvent reference documents exactly the ten top-level `platform.*` namespaces;
  per-event-type names/schemas are deferred to B12 and recorded as such, not invented.
- **ADR 0034** — audit event schema reference reflects the Postgres + S3 system of record,
  OpenSearch advisory fanout; retention/redaction deferred to F1.
- **ADR 0013 / 0041** — capability-CRD and substrate-XRD reference matches the interface contract;
  connection-secret reference uses the canonical `host`/`port`/`user`/`password`/`dbname` shape.
- **ADR 0007** — LibreChat configuration reference documents the locked-down profile.
- **ADR 0008** — docs-as-code Markdown-in-repo, PR review, re-indexable by C8. **ADR 0024** — vendor
  docs are the companion project; C4 documents platform surfaces only.

## 4. Interfaces & Contracts

Use ONLY Canon names — C4 is the quadrant most exposed to interface drift, so every name MUST be
reproduced verbatim from `_meta/interface-contract.md` / `_meta/glossary.md`.

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
N/A as an owned definition — C4 documents, but does not define, every CRD/XRD in interface-contract
§1: ARK CRDs (§1.2), Sandbox CRDs (§1.3), capability CRDs (§1.4), `Approval`/`LogLevel` (§1.5),
Crossplane XRs/XRDs (§1.6). Reference reproduces the source-stated key fields verbatim; fields
marked "not specified in source" (e.g. `Tool`, `Query`) are documented as deferred to ARK, not
invented.

### 4.2 APIs / SDK surfaces
N/A as an owned surface — C4 documents the Platform SDK named surfaces (`memory.*`, `rag.*`, OTel
emission, A2A registration helpers — §3.1), the agent SDK surfaces (LangGraph + Deep Agents — §3.2),
the `agent-platform` CLI subcommand surface (§3.3), and the LiteLLM admin API. Method signatures
beyond the named surface groups are **not specified in source** and are documented as deferred to
B6/B7, tagged `[PROPOSED — not in source]` where a placeholder is unavoidable.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
N/A as an emitter — C4 documents the closed set of ten `platform.*` top-level namespaces
(interface-contract §2) and the `specversion` + `schemaVersion` versioning rule; per-event-type
names/schemas are owned by B12 and documented as deferred, never invented.

### 4.4 Data schemas / connection-secret contracts
N/A as an owned schema — C4 documents the uniform connection-secret shape
(`host`/`port`/`user`/`password`/`dbname`, interface-contract §4), the substrate-agnostic XR status
fields (`ready`/`endpoint`/`version`), and the audit event schema surface (§5). Per-primitive field
lists beyond the canonical five are documented as "not enumerated in source."

## 5. OSS-vs-Custom Decision
N/A — C4 is authored content, not software. It consumes the **Material for MkDocs** portal
(configured by C1, ADR 0008) and authors reference Markdown describing surfaces owned by other
components. No OSS project is forked, wrapped, or built.

## 6. Functional Requirements
- REQ-C4-01: C4 SHALL author every reference document enumerated in §10.3, each information-oriented
  and complete for the surface it covers.
- REQ-C4-02: Reference SHALL cover **both** LangGraph and Langchain Deep Agents SDK surfaces (ADR
  0019).
- REQ-C4-03: Every documented CRD/XRD field, CloudEvent namespace, connection-secret field, CLI
  subcommand, and schema name SHALL match `_meta/interface-contract.md` / `_meta/glossary.md`
  exactly.
- REQ-C4-04: Any field, method, or schema element not present in Canon SHALL be tagged
  `[PROPOSED — not in source]` or recorded as deferred to its owning component — never silently
  invented (interface-contract §6).
- REQ-C4-05: The CloudEvent reference SHALL document exactly the ten `platform.*` top-level
  namespaces and the `specversion`/`schemaVersion` versioning rule (ADR 0031), deferring
  per-event-type schemas to B12.
- REQ-C4-06: The connection-secret reference SHALL document the canonical
  `host`/`port`/`user`/`password`/`dbname` shape and the substrate-agnostic status fields
  `ready`/`endpoint`/`version` (ADR 0041).
- REQ-C4-07: Reference SHALL be version-pinned via C1 per-version builds and document per-component
  versioning ownership (ADR 0030).
- REQ-C4-08: Reference Markdown SHALL be docs-as-code (in-repo, PR-reviewed, passing the C1 check)
  and conform to the clean-Markdown corpus contract so C8 can re-index it without preprocessing.
- REQ-C4-09: Reference content SHALL be scheduled with the last component in each flow (§10), with
  partial reference encouraged from earlier components.

## 7. Non-Functional Requirements
- Security: reference documents secret handling via ESO and capability declaration via CRDs;
  contains no live secrets; the docs check runs SHA-pinned (ADR 0010).
- Multi-tenancy (§6.9): reference documents namespaced CRDs (all v1.0 platform CRDs are namespaced)
  and the Platform JWT claim schema surfaces it references; carries no tenant data.
- Observability (§6.5): the metrics catalog and trace-span-attribute reference document the
  observability surfaces (Tempo/Mimir/Langfuse, trace_id correlation per ADR 0015).
- Scale: corpus grows with surface count; per-version builds are additive (C1).
- Versioning (ADR 0030): reference is the primary doc surface for versioning policy; per-version
  builds track CRD/API/event/SDK/CLI versions and document conversion-webhook/deprecation rules.

## 8. Cross-Cutting Deliverable Checklist
- Helm/manifests: **N/A — authored content; build/deploy owned by C1.**
- Per-product docs (10.5): **N/A — Workstream A; C4 authors Diataxis reference only.**
- Runbook (10.7): **N/A — runbooks are C6 + Workstream A.**
- Alerts: **N/A — content piece, no runtime failure conditions.**
- Grafana dashboard (Crossplane XR): **N/A — C4 documents the metrics catalog; dashboards are Workstream D.**
- Headlamp plugin: **N/A — C4 documents Headlamp extension points; plugins are their owning components.**
- OPA/Rego integration: **N/A — C4 documents the OPA policy library reference; policies are B3/B16.**
- Audit emission (ADR 0034): **N/A — C4 documents the audit event schema; emission is A18.**
- Knative trigger flow: **N/A — re-index triggering is C8.**
- HolmesGPT toolset: **N/A — KB access is via the RAGStore (C8), not a C4 toolset.**
- 3-layer tests (Chainsaw/Playwright/PyTest): **Partial** — PyTest validates Canon-name/field
  conformance against the interface contract; Playwright validates in-portal render/nav; Chainsaw
  **N/A — C4 defines no CRD** (CRD shape conformance is owned by the reconciler components).
- Tutorials & how-tos: **N/A — C4 is the reference quadrant (§10.3); tutorials=C2, how-tos=C3.**

## 9. Acceptance Criteria
- AC-C4-01 (REQ-C4-01): Every §10.3 reference document exists and completely covers its surface.
- AC-C4-02 (REQ-C4-02): The SDK reference documents both LangGraph and Deep Agents surfaces.
- AC-C4-03 (REQ-C4-03): An automated conformance check finds zero CRD/XRD field, event-namespace,
  connection-secret, or CLI-subcommand name in the reference that diverges from the interface
  contract / glossary.
- AC-C4-04 (REQ-C4-04): Every non-Canon element in the reference is tagged `[PROPOSED — not in
  source]` or recorded as deferred; the check finds zero silently-invented fields.
- AC-C4-05 (REQ-C4-05): The CloudEvent reference lists exactly the ten `platform.*` namespaces and
  the versioning rule, with per-event-type schemas marked deferred to B12.
- AC-C4-06 (REQ-C4-06): The connection-secret reference documents the canonical five fields and the
  three substrate-agnostic status fields.
- AC-C4-07 (REQ-C4-07): A reference page is servable at two distinct documented versions via C1's
  version selector and states per-component versioning ownership.
- AC-C4-08 (REQ-C4-08): A reference PR passes the C1 docs check and C8 indexes the output into
  `platform-knowledge-base` with no preprocessing (verified with C8).
- AC-C4-09 (REQ-C4-09): Each multi-component reference doc is finalized with the last component in
  its flow; partial reference from earlier components is present where contributed.

## 10. Risks & Open Questions
- R1 (high): C4 is the quadrant most exposed to interface drift across ~30 parallel specs. Open
  question: keeping reference field-accurate as CRD/XRD/event/SDK surfaces firm up — mitigated by
  the automated interface-contract conformance check (AC-C4-03/04) and flow-last scheduling (§10).
- R2 (med): Many surfaces are explicitly "not specified in source" (Tool/Query fields, SDK method
  signatures, per-event-type schemas, per-primitive secret fields — interface-contract §6). These
  are documented as deferred to their owning components, never invented; `[PROPOSED — not in source]`
  where a placeholder is unavoidable.
- R3 (med): Reference is scheduled flow-last (§10), so the corpus lands late; partial reference from
  earlier components reduces final-component burden — blast radius bounded by encouraging partials.
- R4 (low): Reference folder naming depends on C1's `[PROPOSED]` directory layout (spec-C1 R1); C4
  adopts C1's published layout verbatim.

## 11. References
- architecture-overview.md §10 Documentation plan (~L1425), doc timing / flow-last scheduling
  (~L1431).
- architecture-overview.md §10.3 Reference, both-SDK coverage + full list (~L1469–1487).
- architecture-overview.md §14.3 Workstream C table, C4 row (~L1727).
- architecture-overview.md §6.4 Knowledge Base (`platform-knowledge-base` RAGStore); §6.12 CRD
  inventory; §6.13 versioning; §6.7 eventing.
- ADR 0019 (LangGraph + Deep Agents, both covered), ADR 0030 (versioning), ADR 0031 (CloudEvent
  taxonomy), ADR 0034 (audit pipeline / schema), ADR 0013 (capability CRDs), ADR 0041 (substrate
  XRDs / connection-secret), ADR 0007 (LibreChat locked-down profile), ADR 0008 (Material for
  MkDocs), ADR 0024 (vendor docs separate).
- _meta/glossary.md (all CRD/XRD kinds, products, `platform-knowledge-base`).
- _meta/interface-contract.md §1 (CRDs/XRDs), §2 (CloudEvents), §3 (SDK/CLI surfaces), §4
  (connection-secret), §5 (audit adapter), §6 (gaps left to component design).
- spec-C1.md (portal infrastructure, navigation slots, conventions, contribution workflow).
