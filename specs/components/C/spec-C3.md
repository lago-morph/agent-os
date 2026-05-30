# SPEC C3 — Diataxis how-to guides

> kind: COMPONENT · workstream: C · tier: T1
> upstream: [C1] · downstream: [C8] · adrs: [0010, 0033, 0011, 0019, 0008, 0002, 0003, 0024] · views: [6.4]
> canon-glossary: see _meta/glossary.md · canon-interface: see _meta/interface-contract.md

## 1. Purpose & Problem Statement

How-to guides are the **problem-oriented** Diataxis quadrant (§10.2): goal-directed recipes for a
reader who already knows the platform and needs to accomplish a specific task. Unlike tutorials,
they assume context and solve one concrete problem each. The flagship guide is the **GitHub Actions
reference pipeline** — the single v1.0 CI/CD reference (ADR 0010 / ADR 0033), built around the
`agent-platform` CLI contract; GitLab CI and Jenkins are explicitly out of scope for v1.0.

C3 owns the authored how-to corpus that fills the how-to slot defined by C1. Per the
docs-before-implementation commitment (§10), each guide is authored before (or alongside) the
component it exercises and embeds concrete test expectations. Without a complete, structurally
consistent how-to set — and a single authoritative GitHub Actions pipeline guide — operators and
developers lack the recipe layer between tutorials and reference, and the `platform-knowledge-base`
`RAGStore` (§6.4) is missing the problem-oriented content.

## 2. Scope

### 2.1 In scope
- The full set of §10.2 how-to guides, each problem-oriented and goal-directed:
  - Issue a `VirtualKey` with custom `BudgetPolicy` and model allowlist.
  - Write an OPA policy for tool access.
  - **Integrate with GitHub Actions — the v1.0 reference pipeline** (ADR 0010 / ADR 0033; GitLab CI
    and Jenkins out of scope for v1.0).
  - Debug an agent using traces in Langfuse and Tempo.
  - Write a Headlamp plugin.
  - Roll out an agent change with canary using Langfuse A/B labels.
  - Bring your own SDK into the BYO harness (the raw-LangGraph / third-party-harness path).
  - Expose an agent over A2A.
  - Choose between gVisor and Kata for a `SandboxTemplate` class.
  - Write a Crossplane Composition for a new `AgentEnvironment`.
  - Register a new Keycloak client.
  - Red-team an agent before promotion.
  - Set per-agent egress rules (`EgressTarget` via the Envoy egress proxy).
  - Add a toolset to HolmesGPT for a new component.
- The GitHub Actions reference pipeline guide MUST document the security-first posture (ADR 0010):
  scoped tokens with OIDC federation to AWS, GitHub Environments gating production-affecting jobs,
  SHA-pinned third-party actions, separate build vs deploy runners, signed artifacts, and required
  scan steps (Trivy/Grype, dependency scanning, OPA bundle linting, CRD/CloudEvent schema
  validation), all expressed as native syntax around the `agent-platform` CLI subcommands
  (`validate`, `eval`, `redteam`, `package`, `test`, `deploy-preview`, `promote`, `scan`,
  `update-base`).
- Uniform how-to structure: goal statement, prerequisites, ordered steps, verification.
- Embedded concrete test expectations per guide (§10), reviewed/signed off before the documented
  component is implemented; mock-out instructions where the dependency has not yet landed.

### 2.2 Out of scope (and where it lives instead)
- Portal infrastructure, nav skeleton, search, contribution-workflow check — **C1**.
- The CI/CD reference pipeline **implementation** (the GitHub Actions workflows + the
  `agent-platform` CLI integration contract) — **B15** (B9 owns the CLI). C3 documents it; B15
  builds it.
- Learning-oriented tutorials — **C2**. Information-oriented reference (CLI command reference, CRD
  reference, OPA policy library reference) — **C4**. Understanding-oriented explanation
  (e.g. why GitHub-Actions-only, why Envoy egress) — **C5**.
- Per-product docs (§10.5) / per-product runbooks (§10.7) — **Workstream A** owners. Cross-cutting
  runbooks — **C6**. Maintainer/extender docs — **C7**.
- The RAG **indexing pipeline** — **C8** (C3 only emits re-indexable Markdown).

## 3. Context & Dependencies

Upstream consumed: **C1** — portal infrastructure, how-to navigation slot, authoring conventions,
contribution-workflow check.

Continuous (non-blocking) inputs: the components each guide exercises — **B15** (CI/CD reference
pipeline), **B9** (`agent-platform` CLI), **A1** LiteLLM (`VirtualKey`/`BudgetPolicy`), **A7** OPA,
**A2** Langfuse + **A13** Tempo, **A9** Headlamp, **A6** agent-sandbox + Envoy egress proxy, **B4**
Crossplane Compositions, **Keycloak** (ADR 0028/0029), **A14** HolmesGPT, **B7** agent SDK
(BYO/LangGraph). Authored ahead of these per docs-before-implementation; consumed continuously as
they land, mocks swapped for real dependencies. Recorded as continuous inputs, not blockers.

Downstream consumers: **C8** (indexes the how-to Markdown into `platform-knowledge-base`).

ADR decisions honored:
- **ADR 0010 / 0033** — GitHub Actions is the *only* v1.0 reference CI; the how-to documents one
  GitHub Actions pipeline, not three; GitLab CI / Jenkins guides are dropped from v1.0 scope.
  Adopters on other CI hand-write against the documented CLI contract.
- **ADR 0011** — the CLI is the integration contract and the same CLI drives local, harness, and CI
  execution; how-to test expectations map to the three test layers.
- **ADR 0019** — the BYO-SDK how-to is where raw **LangGraph** / third-party harness usage is
  documented (kept out of tutorials, which are Deep-Agents-only); third-party harness images are NOT
  officially supported in v1.0.
- **ADR 0002** — the OPA-policy-for-tool-access guide reflects RBAC-as-floor / OPA-as-restrictor.
- **ADR 0003** — the per-agent egress guide uses the Envoy egress proxy / `EgressTarget`, not CNI
  L7 policy.
- **ADR 0008** — docs-as-code Markdown-in-repo, PR review, re-indexable by C8.
- **ADR 0024** — references platform docs only; vendor-doc acquisition is the companion project.

## 4. Interfaces & Contracts

Use ONLY Canon names. C3 references (does not define): `VirtualKey`, `BudgetPolicy`, `EgressTarget`,
`SandboxTemplate`, `AgentEnvironment` (XR), `Agent`, A2A, the `agent-platform` CLI, Envoy egress
proxy, OPA / Gatekeeper, Langfuse, Tempo, Headlamp, HolmesGPT, Keycloak, LangGraph.

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
N/A — C3 defines no CRD/XRD. It documents use of Canon CRDs/XRs (`VirtualKey`, `BudgetPolicy`,
`EgressTarget`, `SandboxTemplate`, `AgentEnvironment`) owned by their reconciler components.

### 4.2 APIs / SDK surfaces
N/A as an owned surface — C3 documents the `agent-platform` CLI subcommand surface
(`validate`/`eval`/`redteam`/`package`/`test`/`deploy-preview`/`promote`/`scan`/`update-base`,
interface-contract §3.3, ADR 0010) and the agent SDK BYO surface (B7). The CLI subcommand list
above is source-stated (ADR 0010); any CLI flag/option beyond the named subcommands is
`[PROPOSED — not in source]` until B9 publishes it.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
N/A — C3 emits no CloudEvents. The red-team and promotion guides reference `platform.evaluation.*`
and `platform.approval.*` paths conceptually; concrete per-event-type names are deferred to the B12
registry and MUST be referenced, not invented.

### 4.4 Data schemas / connection-secret contracts
N/A — C3 writes no data/connection-secret. The only contract is the re-indexable Markdown corpus
shape consumed by C8 (co-owned with C1/C8).

## 5. OSS-vs-Custom Decision
N/A — C3 is authored content, not software. It consumes the **Material for MkDocs** portal
(configured by C1, ADR 0008) and documents the **GitHub Actions** reference pipeline (built by B15)
around the `agent-platform` CLI (B9). No OSS project is forked, wrapped, or built by C3.

## 6. Functional Requirements
- REQ-C3-01: C3 SHALL author every how-to guide enumerated in §10.2, each problem-oriented and
  goal-directed with a single clear task.
- REQ-C3-02: C3 SHALL author a single GitHub Actions reference-pipeline how-to documenting the
  security-first posture (scoped OIDC tokens to AWS, GitHub Environments gating, SHA-pinned actions,
  separate build/deploy runners, signed artifacts, required scan steps) expressed as native syntax
  around the `agent-platform` CLI subcommands (ADR 0010 / 0033).
- REQ-C3-03: C3 SHALL NOT author GitLab CI or Jenkins reference pipelines (out of v1.0 scope, ADR
  0010); where adopters use other CI, the guide SHALL point at the documented CLI contract instead.
- REQ-C3-04: Every guide SHALL follow a uniform structure — goal, prerequisites, ordered steps,
  verification — authored against C1's conventions and navigation slot.
- REQ-C3-05: Every guide SHALL embed concrete test expectations (§10) reviewed/signed off before the
  documented component is implemented; guides with un-implemented dependencies SHALL include
  mock-out instructions (§10).
- REQ-C3-06: The BYO-SDK how-to SHALL be the documented home for raw **LangGraph** and third-party
  harness usage, noting third-party harness images are NOT officially supported in v1.0 (ADR 0019).
- REQ-C3-07: Guides SHALL use ONLY Canon names; any element not in Canon SHALL be tagged
  `[PROPOSED — not in source]`.
- REQ-C3-08: How-to Markdown SHALL be docs-as-code (in-repo, PR-reviewed, passing the C1 check) and
  conform to the clean-Markdown corpus contract so C8 can re-index it without preprocessing.
- REQ-C3-09: The egress how-to SHALL use the Envoy egress proxy / `EgressTarget` (ADR 0003) and the
  OPA-policy how-to SHALL reflect RBAC-as-floor / OPA-as-restrictor (ADR 0002 / 0018).

## 7. Non-Functional Requirements
- Security: the GitHub Actions guide is itself the security-first pipeline reference (ADR 0010);
  guides show secrets via ESO and capabilities declared via CRDs; the docs check runs SHA-pinned.
- Multi-tenancy (§6.9): guides operate within approved namespaces; the `VirtualKey`/`BudgetPolicy`
  and egress guides illustrate per-tenant/per-key scoping; no tenant data is embedded.
- Observability (§6.5): the Langfuse+Tempo debug guide and the canary/A-B guide exercise the
  trace_id correlation (ADR 0015); build status visible via the GitHub Actions docs check.
- Scale: corpus grows with the guide set; per-version builds are additive (C1).
- Versioning (ADR 0030): the CLI subcommand surface is the versioned contract (ADR 0030 §3.3);
  guides pin to documented CLI/CRD versions via C1 per-version builds.

## 8. Cross-Cutting Deliverable Checklist
- Helm/manifests: **N/A — authored content; build/deploy owned by C1.**
- Per-product docs (10.5): **N/A — Workstream A; C3 authors Diataxis how-tos only.**
- Runbook (10.7): **N/A — runbooks are C6 + Workstream A.**
- Alerts: **N/A — content piece, no runtime failure conditions.**
- Grafana dashboard (Crossplane XR): **N/A — no runtime metrics.**
- Headlamp plugin: **N/A — C3 authors the *how-to-write-a-plugin* guide; the framework is A9, plugins are their owning components.**
- OPA/Rego integration: **N/A — C3 documents how to write a tool-access OPA policy; the policy library is B3/B16.**
- Audit emission (ADR 0034): **N/A — no audit-relevant runtime actions.**
- Knative trigger flow: **N/A — re-index triggering is C8; guides only describe flows.**
- HolmesGPT toolset: **N/A — C3 authors the *add-a-toolset* guide; toolsets ship with their owning components.**
- 3-layer tests (Chainsaw/Playwright/PyTest): **Applicable** — embedded test expectations map to the
  three layers (ADR 0011); Playwright covers in-portal render/nav; the CI guide is validated against
  the real GitHub Actions pipeline (B15).
- Tutorials & how-tos: **Applicable — this IS the how-to deliverable (§10.2); tutorials are C2.**

## 9. Acceptance Criteria
- AC-C3-01 (REQ-C3-01): Every §10.2 how-to exists, is problem-oriented, and solves its single stated
  task when followed.
- AC-C3-02 (REQ-C3-02): The GitHub Actions reference-pipeline guide documents every required
  security-first behavior and expresses each step as native syntax around an `agent-platform` CLI
  subcommand.
- AC-C3-03 (REQ-C3-03): No GitLab CI or Jenkins reference pipeline guide exists; the CI guide points
  other-CI adopters at the CLI contract.
- AC-C3-04 (REQ-C3-04): Every guide contains goal, prerequisites, ordered steps, and verification.
- AC-C3-05 (REQ-C3-05): Every guide's test expectations are signed off before the documented
  component's implementation begins; guides with un-implemented dependencies include mock-out
  instructions that run green against the mock and unchanged against the real dependency.
- AC-C3-06 (REQ-C3-06): The BYO-SDK guide documents raw LangGraph / third-party harness usage and
  states third-party harness images are not officially supported.
- AC-C3-07 (REQ-C3-07): A Canon-name lint finds zero non-Canon names except those tagged
  `[PROPOSED — not in source]`.
- AC-C3-08 (REQ-C3-08): A how-to PR merges through the docs-as-code path, passes the C1 docs check,
  and C8 indexes the output into `platform-knowledge-base` with no preprocessing (verified with C8).
- AC-C3-09 (REQ-C3-09): The egress guide uses `EgressTarget` + Envoy egress proxy and the OPA guide
  reflects RBAC-as-floor / OPA-as-restrictor.

## 10. Risks & Open Questions
- R1 (med): The CI guide depends on B15 (pipeline impl) + B9 (CLI). Open question: CLI flag-level
  detail beyond the source-stated subcommands — `[PROPOSED — not in source]` until B9 publishes;
  guide authored against the subcommand contract, flag detail filled when B9 lands.
- R2 (med): Guides authored before their components land (§10); mock-out drift risk — mitigated by
  the docs-before-implementation sign-off and re-run-on-real-dependency gate.
- R3 (low): Per-event-type names for the red-team / promotion guides are deferred to B12
  (interface-contract §2/§6); guides reference namespaces conceptually, no invented names.
- R4 (low): Guide folder naming depends on C1's `[PROPOSED]` directory layout (spec-C1 R1); C3
  adopts C1's published layout verbatim.

## 11. References
- architecture-overview.md §10 Documentation plan (~L1425), docs-before-implementation + mock-out
  (~L1433).
- architecture-overview.md §10.2 How-to guides, full list incl. GitHub Actions reference pipeline
  (~L1452–1467).
- architecture-overview.md §8 CI/CD integration requirements (security-first pipeline behaviors).
- architecture-overview.md §14.3 Workstream C table, C3 row (~L1726); B15 row §14.2 (~L1709).
- architecture-overview.md §6.4 Knowledge Base (`platform-knowledge-base` RAGStore).
- ADR 0010 (GitHub Actions only; CLI subcommand contract; security-first pipeline), ADR 0033
  (AWS + GitHub initial targets), ADR 0011 (three-layer testing / CLI orchestration), ADR 0019
  (LangGraph / Deep Agents; BYO harness not supported), ADR 0002 / 0018 (OPA / RBAC-floor),
  ADR 0003 (Envoy egress), ADR 0008 (Material for MkDocs), ADR 0024 (vendor docs separate).
- _meta/glossary.md (`agent-platform`, Envoy egress proxy, OPA / Gatekeeper, Langfuse, Tempo,
  Headlamp, HolmesGPT, Keycloak, LangGraph, A2A).
- _meta/interface-contract.md §1.4 (`VirtualKey`, `BudgetPolicy`, `EgressTarget`), §1.3
  (`SandboxTemplate`), §1.6 (`AgentEnvironment`), §3.3 (CLI subcommand surface), §2 (CloudEvents).
- spec-C1.md (portal infrastructure, navigation slots, conventions, contribution workflow).
