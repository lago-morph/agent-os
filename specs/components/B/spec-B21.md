# SPEC B21 — Development environment for agents

> kind: COMPONENT · workstream: B · tier: T2
> upstream: [B7, B6, B9] · downstream: [] · adrs: [0019, 0033, 0040, 0028, 0030, 0011] · views: [6.2, 6.11]
> canon-glossary: see _meta/glossary.md · canon-interface: see _meta/interface-contract.md

## 1. Purpose & Problem Statement

Developers building new Platform Agents need a supported, documented path from empty repo to a
running agent — the SDK to use, how to wire capabilities, how to run and debug locally, and how to
push to a shared cluster. Without a curated development environment, every developer improvises a
setup, diverges from the platform's interaction surfaces (§6.2), and discovers governance
constraints (LiteLLM / OPA / Envoy perimeter) only at deploy time.

B21 ships the **documentation, tooling, and conventions** for agent development across two
supported paths — a **local kind environment** and a **shared dev cluster** — without mandating
either platform-wide. Per §14.2 B21 the local-vs-shared choice is design-time and per-developer;
B21's job is to make **both** paths work and to converge them on the same SDKs (B7/B6), the same
CLI (B9), and the same GitOps deploy loop (§7.4).

## 2. Scope

### 2.1 In scope
- Developer documentation: getting started, project layout, how to author an `Agent` against the
  platform interaction surfaces (§6.2), how to use the agent SDK (B7, `deep-agents` default) and
  Platform SDK (B6, `memory.*`/`rag.*`).
- Tooling/conventions to bootstrap, run, and iterate an agent on **both** a **local kind**
  environment and a **shared dev cluster** (§14.2 B21), built on the `agent-platform` CLI (B9).
- The local-vs-shared decision guide (trade-offs, when to use which) — neither is platform-wide-mandated.
- Developer cluster access conventions via Keycloak-OIDC `kubectl` (§6.11 Flow 3), supported on kind.
- The within-environment iteration loop (Git → CI evals → ArgoCD; §7.4) documented from the developer's seat.
- Conventions that keep developer agents inside the platform perimeter (LiteLLM/OPA/Envoy) so local
  work matches cluster behavior.

### 2.2 Out of scope
- The agent SDK harness itself — **B7**; the Platform SDK — **B6**; the CLI implementation — **B9**.
- Cluster install/bootstrap (kind/EKS baseline, OIDC issuer config) — **k8-platform** / Workstream A / A15.
- CI/CD reference pipeline — **B15**; the test framework — **B14** (B21 documents usage, doesn't build them).
- Environment-to-environment promotion — **Kargo / A23** (ADR 0040); B21 covers within-environment iteration only.
- Tenant provisioning for a dev namespace — **A21** (`TenantOnboarding`, ADR 0037).
- The agent profile library / recommended compositions used as starting points — **B17 / B18**.

## 3. Context & Dependencies

Upstream consumed:
- **B7** (agent SDK) — the LangGraph / Deep Agents authoring surface and supported `Agent.sdk`
  values developers build against. What is consumed: the SDK + harness shape.
- **B6** (Platform SDK) — `memory.*`/`rag.*`/OTel/A2A-registration surfaces the dev agent calls;
  B21 documents their use and the version-compat matrix.
- **B9** (`agent-platform` CLI) — the developer/operator/CI entrypoint B21's tooling and docs drive.

ADR decisions honored:
- **ADR 0019** — `deep-agents` is the opinionated default the dev experience leads with; `langgraph` supported.
- **ADR 0033 (initial targets — AWS/EKS + GitHub) + dual-mode (§6.3)** — kind is functionally
  complete without cloud services (§6.3 dual-mode hosting), so the local kind path is viable; the
  shared dev cluster mirrors the AWS substrate where needed.
- **ADR 0011** — three-layer testing (Chainsaw/Playwright/PyTest) orchestrated by the CLI; B21
  documents how a developer runs them locally.
- **ADR 0040** — within-environment deploy is PR + ArgoCD; promotion across Stages is Kargo (out of scope here).
- **ADR 0028 / §6.11 Flow 3** — developer `kubectl` access is Keycloak-OIDC, supported on kind.
- **ADR 0030** — SDK/CLI versioning is semantic; B21 docs pin to the compat matrix.

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
N/A — B21 introduces no CRD/XRD. It documents authoring of existing `Agent` (and referenced
`CapabilitySet`) resources whose fields are owned by A5/B13 (interface-contract §1.2 / §1.4).

### 4.2 APIs / SDK surfaces
N/A (no new surface) — B21 consumes and documents existing surfaces: the agent SDK (B7), the
Platform SDK groups `memory.*`/`rag.*`/OTel/A2A-registration (B6, interface-contract §3.1), and the
`agent-platform` CLI subcommand surface (B9, interface-contract §3.3). B21 defines no new method
signatures; any developer-workflow command names beyond B9's shipped surface are
`[PROPOSED — not in source]`.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
N/A — B21 is docs/tooling/conventions; it emits and consumes no CloudEvents. A dev agent's runs
emit `platform.lifecycle.*` via ARK/sandbox like any agent — owned upstream, not by B21.

### 4.4 Data schemas / connection-secret contracts
N/A — B21 owns no data backend. On the kind path, local substrate backings follow the ADR 0041
connection-secret contract owned by B4; B21 only documents how a developer consumes them.

## 5. OSS-vs-Custom Decision
**Build-new (docs + thin tooling/conventions), no new platform service.** B21 wraps existing
pieces — kind, the `agent-platform` CLI (B9), the SDKs (B6/B7), ArgoCD — into a documented,
convention-driven developer experience. No new OSS project is forked; thin glue (scaffolds,
make/justfile-style task conventions, local-up scripts) sits over B9. Rationale: the building
blocks exist; the gap is a coherent, supported on-ramp covering both local and shared paths.

## 6. Functional Requirements
- REQ-B21-01: B21 SHALL document and tool a path to bootstrap, run, and iterate a new Platform
  Agent on a **local kind** environment (§14.2 B21, ADR 0033).
- REQ-B21-02: B21 SHALL document and tool the equivalent path on a **shared dev cluster**, and SHALL
  provide a decision guide for choosing between local and shared (neither mandated platform-wide).
- REQ-B21-03: The development experience SHALL lead with the agent SDK `deep-agents` default while
  documenting `langgraph` (ADR 0019), and SHALL use the Platform SDK surfaces (B6) for in-platform calls.
- REQ-B21-04: Tooling SHALL be built on the `agent-platform` CLI (B9) rather than a parallel toolchain.
- REQ-B21-05: B21 SHALL document developer cluster access via Keycloak-OIDC `kubectl` (§6.11 Flow 3),
  including the kind-supported path.
- REQ-B21-06: B21 SHALL document the within-environment iteration loop (Git → CI evals → ArgoCD,
  §7.4) and SHALL make clear that cross-Stage promotion is Kargo's job (out of scope).
- REQ-B21-07: Conventions SHALL keep a developer agent inside the platform perimeter (LiteLLM for
  LLM/MCP/A2A, OPA for policy, Envoy for egress) so local behavior matches cluster behavior (§6.2).
- REQ-B21-08: B21 SHALL document running the three test layers (Chainsaw/Playwright/PyTest) for a
  dev agent via the CLI (ADR 0011), and SHALL pin to the SDK/CLI version-compat matrix (ADR 0030).
- REQ-B21-09: B21 SHALL recommend starting from B17 profiles / B18 compositions and SHALL NOT
  duplicate that library content.

## 7. Non-Functional Requirements
- Security / multi-tenancy (§6.9): dev agents run in an approved namespace; B21 conventions never
  instruct bypassing the perimeter. Local kind still authenticates `kubectl` via Keycloak-OIDC (§6.11).
- Observability (§6.5): B21 documents emitting OTel via the Platform SDK so dev runs are traceable in Langfuse/Tempo.
- Scale: developer-facing only; no runtime scaling concern. Shared dev cluster capacity is an operations concern (Workstream A/F).
- Versioning (ADR 0030): docs/tooling pin to the SDK + CLI semantic versions and the compat matrix;
  updated as those surfaces version.

## 8. Cross-Cutting Deliverable Checklist
- Helm/manifests: **Applicable (thin)** — local-up / scaffold manifests + example dev-namespace overlay.
- Per-product docs (10.5): **Applicable** — the core deliverable: getting-started + conventions + decision guide.
- Runbook (10.7): **N/A — no runtime service; dev-environment troubleshooting ships as how-to docs, not an operator runbook.**
- Alerts: **N/A — no runtime component emits metrics here.**
- Grafana dashboard (Crossplane XR): **N/A — developer dashboards are owned by D2, not B21.**
- Headlamp plugin: **N/A — no dedicated plugin.**
- OPA/Rego integration: **N/A — B21 documents the perimeter; it authors no Rego (content owned by B16/B3).**
- Audit emission (ADR 0034): **N/A — B21 is docs/tooling; dev agents audit via the adapter like any agent.**
- Knative trigger flow: **N/A — no originating trigger flow.**
- HolmesGPT toolset: **N/A — no toolset contribution.**
- 3-layer tests (Chainsaw/Playwright/PyTest): **Applicable** — B21 ships example test scaffolds and documents running all three via the CLI; PyTest validates the scaffold/bootstrap tooling itself.
- Tutorials & how-tos: **Applicable** — "build your first Platform Agent (kind)" and "deploy to the shared dev cluster" (content in C2/C3, seeded here).

## 9. Acceptance Criteria
- AC-B21-01 (REQ-B21-01): Following the docs, a developer bootstraps and runs a new agent on local kind end-to-end.
- AC-B21-02 (REQ-B21-02): The shared-dev-cluster path is documented and exercisable, and a decision guide comparing the two paths exists.
- AC-B21-03 (REQ-B21-03): The getting-started flow uses `deep-agents` by default, documents `langgraph`, and shows a Platform SDK `memory.*`/`rag.*` call.
- AC-B21-04 (REQ-B21-04): The bootstrap/iterate tooling invokes the `agent-platform` CLI (no parallel toolchain).
- AC-B21-05 (REQ-B21-05): The docs include a working Keycloak-OIDC `kubectl` login flow that succeeds on kind.
- AC-B21-06 (REQ-B21-06): The docs describe the Git → CI evals → ArgoCD loop and explicitly defer cross-Stage promotion to Kargo.
- AC-B21-07 (REQ-B21-07): A dev agent following the conventions reaches LLM/MCP/A2A only via LiteLLM and outbound HTTP only via Envoy-allowlisted `EgressTarget`s (verified by attempting and failing an off-perimeter call).
- AC-B21-08 (REQ-B21-08): A developer runs all three test layers for the sample agent via the CLI, against the pinned compat matrix.
- AC-B21-09 (REQ-B21-09): The getting-started flow starts from a B17 profile / B18 composition and references, not copies, that library.

## 10. Risks & Open Questions
- R1 (med): Specific developer-workflow **command names / scaffold layout** are not in Canon —
  `[PROPOSED — not in source]`; they must align with B9's shipped subcommand surface. Reconciliation
  note: B9 owns the CLI surface; B21 documents it and must rebase on B9.
- R2 (med): The **shared dev cluster** topology (who provisions, capacity, multi-tenant isolation for
  developers) is not architecture-specified. Open question for Workstream A/F; B21 documents conventions, not the cluster.
- R3 (low): Parity gaps between kind and the shared cluster (substrate capability-parity caveat, §6.3
  / ADR 0041) may surface behavior differences. Open question: document known gaps per the Composition docs.
- R4 (low): Local "third-party / custom harness images" are documented-but-unsupported (§6.2); B21
  must mark that path clearly as unsupported to avoid implying official support.

## 11. References
- architecture-overview.md §6.2 Agent runtime architecture (~L247–292; interaction surfaces L286–292; custom-harness note L292).
- architecture-overview.md §6.11 Identity federation, Flow 3 developer `kubectl` (~L876–885).
- architecture-overview.md §7.4 Developer iteration via GitOps (~L1219–1226).
- architecture-overview.md §14.2 Workstream B, B21 row (~L1715); B7 (~L1701); B6 (~L1700); B9 (~L1703).
- ADR 0019 (LangGraph + Deep Agents default), ADR 0033 (initial targets — AWS/EKS + GitHub; kind functionally complete per §6.3),
  ADR 0011 (three-layer testing via CLI), ADR 0040 (Kargo promotion — out of scope), ADR 0028 (identity federation), ADR 0030 (versioning).
- _meta/interface-contract.md §3.1 (Platform SDK surfaces), §3.2 (agent SDK), §3.3 (CLI versioned surface).
- _meta/glossary.md (Platform Agent, k8-platform, substrate, ESO, IRSA / Workload Identity).
