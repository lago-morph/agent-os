# SPEC A14 — HolmesGPT (early-phase platform self-management agent)

> kind: COMPONENT · workstream: A · tier: T1
> upstream: [] · downstream: [A16, B10] · adrs: [0012] · views: [6.10, 6.7]
> canon-glossary: b0edae10 · canon-interface: 0ce201d5

## 1. Purpose & Problem Statement

A14 deploys **HolmesGPT**, the platform's self-management agent (ADR 0012; §6.10). HolmesGPT is itself a Platform Agent — declared as an `Agent` CRD and administered like any other agent — with broad **read-only** access to platform state (OPA policies, audit data, traces, metrics, the Knowledge Base) and policy-gated write access for remediation. It exposes an A2A interface so other Platform Agents can hand off troubleshooting. The defining constraint of A14 is its **phased trajectory**: it lands very early — before most of the platform exists — running on the raw Kubernetes cluster with a read-only ServiceAccount + read-only AWS IAM role and no secret-store access, useful for diagnostics during the platform implementation itself, then is **incrementally rewired** through the platform layers (LiteLLM, audit adapter, OPA decision points) and grown with toolsets as each component lands (ADR 0012; waves.md foundation note).

The problem A14 solves is platform self-diagnosis from day one without creating a parallel governance surface. Running HolmesGPT on-platform forces the architecture to eat its own dog food: every constraint that applies to a tenant agent eventually applies to the platform's own diagnostic agent. The phased approach reconciles "useful immediately" with "fully governed eventually" — blunt external guardrails first (RBAC-floor ServiceAccount, restricted IAM), platform-mediated guardrails (LiteLLM routing, OPA three-state gating, audit-adapter emission) as they become available.

A14's spec must therefore describe **two states**: phase-1 raw-cluster read-only, and the rewired end-state, plus the migration between them. Read-only is the default throughout; write actions expand incrementally under the three-state OPA model.

## 2. Scope

### 2.1 In scope

- **Phase 1 (raw cluster, foundation wave):** deploy HolmesGPT with a read-only cluster ServiceAccount (read across all namespaces, write nothing), a read-only AWS IAM role (read tags, describe resources, read CloudWatch — no Secrets Manager, no writes), and **no secret-store access of any kind** (ADR 0012; §6.10).
- HolmesGPT declared as an `Agent` CRD once ARK (A5) is available — administered like any other Platform Agent (ADR 0012).
- HolmesGPT **A2A interface** so other Platform Agents hand off troubleshooting; calls into HolmesGPT traverse LiteLLM and are audited like any agent call (once rewired).
- **Phase-N rewiring:** move LLM calls behind LiteLLM (A1); move audit emission to the audit adapter/endpoint (A18); move tool authorization under OPA decision points (A7); A2A handoffs through the gateway — each as that layer lands.
- **Three-state OPA permission model** (allowed / upon-approval / denied) at a single central OPA decision point for every HolmesGPT action; upon-approval routes through the generalized Approval system (B19, ADR 0017).
- v1.0 write surface: **read-only diagnostics + auto-PR / Headlamp suggestion cards**; narrow autonomous remediations only as OPA graduates a pattern from upon-approval to allowed.
- HolmesGPT consumes the **Knowledge Base** (`platform-knowledge-base` RAGStore) via the SDK `rag.*` path (separate primitive — not part of HolmesGPT).
- HolmesGPT **drives the dynamic diagnostics toggle** (`LogLevel`, ADR 0035) through the same OPA-gated path as any write action.
- The **AlertManager → HolmesGPT** Knative trigger flow — one of the two initial v1.0 trigger flows (ADR 0012; §6.7).
- Read access to OPA policies (for loophole/gap analysis) — explicit accepted self-referential trade-off (ADR 0012; §6.10).
- Standard Workstream A deliverables (§14.1).

### 2.2 Out of scope (and where it lives instead)

- The Approval CRD / OPA elevation logic / Argo Workflow integration — **B19** (ADR 0017). A14 is a consumer of the approval path.
- The audit adapter library + endpoint — **A18** (ADR 0034). A14 links against it during rewiring.
- LiteLLM gateway + callbacks — **A1** / **B2**. A14's calls move behind it during rewiring.
- OPA engine + the central decision point + the bundles authored in Headlamp — **A7** / **B16** / ADR 0039 editors. A14 contributes its policy targets and consumes the decision point.
- The Knowledge Base RAGStore itself + its indexing pipeline — **A8 endpoint via A16/§6.4**, **B4** (RAGStore composition), **C8** (indexing). A14 only includes it in its CapabilitySet.
- The `LogLevel` reconciler mechanics — ADR 0035 / per-component; A14 only triggers it.
- The Interactive Access Agent — **A16** (downstream; shares the KB).
- The Coach Component — **B10** (downstream; consumes HolmesGPT/observability).
- Per-component HolmesGPT **toolsets** for other components — owned by those components (each component contributes its own toolset; A14 owns the agent + the toolset-registration mechanism, not every toolset).

## 3. Context & Dependencies

**Upstream consumed (HARD):** none in the CSV — A14 is a **foundation-wave (W0)** component that lands on the raw cluster before platform dependencies exist (waves.md; CSV upstream empty). Its dependencies are **acquired over time** through rewiring, not pre-required.

**Upstream consumed (over time, non-blocking / dotted):** A5 (`Agent` CRD to declare it), A1 (LiteLLM), A18 (audit adapter), A7 (OPA decision point), B19 (Approval), A13/A2 (observability toolset — dotted edges `A13 -.-> A14`, `A2 -.-> A14`, non-blocking per waves.md), the Knowledge Base RAGStore, B8 (AlertManager→HolmesGPT adapter), ADR 0035 `LogLevel`. Each rewires HolmesGPT incrementally as it lands.

**Downstream consumers:** **A16** (Interactive Access Agent — shares the KB), **B10** (Coach Component — consumes HolmesGPT + observability).

**ADR decisions honored:**
- **ADR 0012** — HolmesGPT is a first-class Platform Agent; A2A interface; three-state OPA model at one central decision point edited centrally via Headlamp (ADR 0039) and validated by the simulator (ADR 0038); phased trajectory (raw cluster → rewired); read-only default, incremental write expansion; AlertManager→HolmesGPT trigger flow; drives `LogLevel` toggle; in-scope (not privileged) for the threat model (ADR 0027); KB is a separate primitive.
- **ADR 0017** — upon-approval actions route through the generalized Approval system.
- **ADR 0002 / §6.6** — OPA gating of HolmesGPT tools/actions.
- **ADR 0034** — audit through the adapter to Postgres+S3 system of record (OpenSearch advisory).
- **ADR 0035** — dynamic diagnostics toggle.
- **ADR 0031 / 0030** — CloudEvent taxonomy + versioning.

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)

A14 **owns no CRD**. It is declared as an `Agent` CRD instance (owned by A5):

| Resource | Owner | Fields A14 relies on (source-stated) |
|---|---|---|
| `Agent` | A5 (ARK) | `capabilitySetRefs[]` (includes the KB RAGStore + toolsets), `exposes` (A2A interface), `modelRef`, `sandboxTemplateRef`, `triggers` (AlertManager flow) |
| `LogLevel` | per-component (ADR 0035) | `componentSelector`, `level`, `traceGranularity`, `scope`, `expiresAt` — HolmesGPT creates/requests these as an OPA-gated write |
| `Approval` | B19 | `requestingAgent`, `actionType`, `actionAttributes`, `decision`, … — created for upon-approval actions |

`[PROPOSED — not in source]` no HolmesGPT-specific `Agent` field is invented; phase-1 raw-cluster mode predates the `Agent` CRD and is configured via plain Helm/ServiceAccount/IAM, not a CRD.

### 4.2 APIs / SDK surfaces

- HolmesGPT **exposes an A2A interface** (`exposes` on its `Agent`), versioned per interface-contract §3.3 (e.g. a declared agent A2A version; peers pin a major). The concrete interface name/version is design-time. `[PROPOSED — not in source]` the specific A2A interface identifier is not in Canon.
- HolmesGPT consumes the **Knowledge Base** via Platform SDK `rag.*` (B6), through LiteLLM (once rewired).
- A HolmesGPT-callable **policy-simulator skill** (ADR 0038 / §6.6) — the simulator exposed as a skill it invokes when diagnosing denials.
- Toolsets are contributed by each component; A14 owns the registration surface that aggregates them (LiteLLM/ARK/observability/OPA/sandbox/eventing toolsets per §6.10 diagram).

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)

- **Consumed:** **AlertManager alerts consumed via the event bus** — AlertManager (alarming off Mimir metrics AND Loki logs) publishes its alerts onto the bus under `platform.observability` (schema owned by A13); a Knative Trigger filters diagnostic-relevant ones (unschedulable pods, high error rates, sustained gateway latency) and dispatches to HolmesGPT (ADR 0012; §6.7). **There is no direct `AlertManager → HolmesGPT` wire — consumption is bus-mediated.** Separately, HolmesGPT pulls **metrics from Mimir directly** as a data stream for doing its job; that direct query is **not** events and does **not** ride the bus. The AlertManager source is environment-specific (ADR 0023).
- **Emitted:** HolmesGPT actions (queries run, recommendations, actions taken) emit audit under **`platform.audit.*`** via the adapter (§6.6); audit emission is gated on the audit-adapter freeze-gate (D-05). OPA decisions on HolmesGPT actions surface under **`platform.policy.*`**; approval-routed actions surface under **`platform.approval.*`** (via B19). Auto-PR / suggestion-card emission is audited. A14 owns no `platform.*` namespace — it takes an explicit dependency on each owner (A18 audit, A7 policy/security, A13 observability, B19 approval) and does not co-own them. For any security-relevant event A14 detects, it MUST handle it locally AND additionally emit under `platform.security` (schema owned by A7, QN-03).
- Per-event-type names deferred to B12. `[PROPOSED — not in source]` concrete event-type names not in Canon.

### 4.4 Data schemas / connection-secret contracts

- N/A — A14 owns no datastore. In **phase 1 it has no secret-store access at all** (ADR 0012). It reads platform state read-only; it persists nothing of record. Auto-PRs land in Git via the normal PR flow; suggestion cards surface in Headlamp.

## 5. OSS-vs-Custom Decision

- **Upstream project:** HolmesGPT (self-management agent), run on-platform as a Platform Agent rather than as an external operator tool (ADR 0012 rationale — avoids a parallel governance surface).
- **Mode:** **config + wrap + phased rewire**. Phase 1: plain install on raw cluster (Helm + read-only ServiceAccount + read-only IAM). Phase N: re-declared as an `Agent` CRD, calls moved behind LiteLLM, audit behind the adapter, tools under OPA. No fork.
- **Rationale:** on-platform governance, dog-fooding, toolset-grows-with-platform (ADR 0012).
- `[PROPOSED — not in source]` exact HolmesGPT version/chart coordinates not in Canon; pinned at install.

## 6. Functional Requirements

- REQ-A14-01: Phase-1 deploy runs HolmesGPT on the raw cluster with a read-only cluster ServiceAccount (read all namespaces, write nothing), a read-only AWS IAM role (read tags/describe/CloudWatch; no Secrets Manager; no writes), and no secret-store access.
- REQ-A14-02: Once ARK (A5) is available, HolmesGPT is declared and managed as an `Agent` CRD, administered like any other Platform Agent.
- REQ-A14-03: HolmesGPT exposes an A2A interface (`exposes`) through which other Platform Agents hand off troubleshooting; once rewired, those calls traverse LiteLLM and are audited.
- REQ-A14-04: Every HolmesGPT action resolves to exactly one of allowed / upon-approval / denied at a single central OPA decision point; upon-approval routes through the generalized Approval system (B19).
- REQ-A14-05: Read-only is the default; v1.0 write surface is auto-PR + Headlamp suggestion cards; autonomous remediation occurs only for patterns OPA has graduated to allowed.
- REQ-A14-06: HolmesGPT includes the Knowledge Base (`platform-knowledge-base` RAGStore) in its CapabilitySet and reaches it via the SDK `rag.*` path.
- REQ-A14-07: The rewiring path is implemented and documented: LLM calls move behind LiteLLM, audit emission to the audit adapter/endpoint, tool authorization under OPA, A2A through the gateway — each as the corresponding component lands; read-only stays default; write expands incrementally.
- REQ-A14-08: The AlertManager → HolmesGPT Knative trigger flow is wired: AlertManager alerts arrive as CloudEvents, a Trigger filters diagnostic-relevant ones and dispatches to HolmesGPT, which produces findings/suggestions/auto-PRs.
- REQ-A14-09: HolmesGPT can request the dynamic diagnostics toggle (`LogLevel`) through the same OPA-gated path as any other write action.
- REQ-A14-10: HolmesGPT's actions (queries, recommendations, actions taken, auto-PR/suggestion emission) emit audit under `platform.audit.*` via the adapter (after rewiring); OPA decisions surface under `platform.policy.*`.
- REQ-A14-11: HolmesGPT has read access to the OPA policies that gate it and can invoke the policy-simulator skill to analyze them.
- REQ-A14-12: A14 owns the toolset-registration surface that aggregates per-component HolmesGPT toolsets (LiteLLM/ARK/observability/OPA/sandbox/eventing).
- REQ-A14-13: Standard Workstream A deliverables: Helm/manifests, docs, runbook, alerts, `GrafanaDashboard` XR, Headlamp plugin (where useful — suggestion cards), OPA integration, audit, the trigger flow, tests, tutorials/how-tos.

## 7. Non-Functional Requirements

- **Security / threat model (§6.6, ADR 0027):** HolmesGPT is **in-scope, not privileged** — same sandbox, gateway, OPA, audit as a tenant agent (after rewiring). Phase-1 blunt guardrails (read-only SA + restricted IAM + no secrets) are the floor until platform-mediated controls land. The self-referential OPA-read trade-off is accepted (find self-bypass paths rather than leave them hidden).
- **Multi-tenancy (§6.9):** HolmesGPT reads across namespaces by design (diagnostics) but writes only through OPA-gated paths; its broad read is the principal residual-risk item for B22.
- **Observability (§6.5):** consumes Tempo/Mimir/Loki/Langfuse toolsets (dotted, non-blocking); emits OTel itself once rewired.
- **Phasing / reversibility:** each rewiring step must be independently shippable and reversible; read-only default must never regress during a rewire.
- **Versioning (ADR 0030):** the A2A interface declares a version; peers pin a major.

## 8. Cross-Cutting Deliverable Checklist

| Deliverable (§14.1) | Status |
|---|---|
| Helm values / manifests in Git | Applicable — phase-1 install + later `Agent` CRD form |
| Per-product docs (10.5) | Applicable |
| Operator runbook (10.7) | Applicable — incl. rewiring procedure |
| Backup / restore | N/A — HolmesGPT holds no system-of-record state |
| Alert rules | Applicable — HolmesGPT unavailability, failed diagnostic dispatch |
| Grafana dashboard (Crossplane XR) | Applicable |
| Headlamp plugin | Applicable — suggestion cards surface in Headlamp (with B19 approval workflow) |
| OPA / Rego integration | Applicable — three-state model at central decision point; targets to B16 |
| Audit emission (ADR 0034) | Applicable — queries/recommendations/actions/auto-PR (after rewire) |
| Knative trigger flow | Applicable — AlertManager → HolmesGPT |
| HolmesGPT toolset | Applicable (special) — A14 owns the aggregation surface; other components contribute toolsets |
| 3-layer tests | Applicable — Chainsaw (Agent CRD + trigger), Playwright (suggestion cards UI), PyTest (three-state decision + rewiring logic) |
| Tutorials & how-tos | Applicable |

## 9. Acceptance Criteria

- AC-A14-01 (REQ-A14-01): On a raw cluster, HolmesGPT runs with a SA that can read all namespaces and write nothing, an IAM role with no Secrets Manager and no write actions, and no mounted secret store.
- AC-A14-02 (REQ-A14-02): After A5 lands, HolmesGPT is reconciled from an `Agent` CRD and appears in the ARK-managed inventory.
- AC-A14-03 (REQ-A14-03): A second Platform Agent hands off a troubleshooting question via the A2A interface; post-rewire the call traverses LiteLLM and is audited.
- AC-A14-04 (REQ-A14-04): A representative action evaluates to each of allowed / upon-approval / denied; the upon-approval case creates an `Approval` and blocks until a human decides.
- AC-A14-05 (REQ-A14-05): A non-graduated remediation produces an auto-PR / suggestion card rather than executing; a graduated (allowed) pattern executes autonomously.
- AC-A14-06 (REQ-A14-06): HolmesGPT answers a KB-grounded question via the `rag.*` path against `platform-knowledge-base`.
- AC-A14-07 (REQ-A14-07): The rewiring runbook is executed in a staged environment: LLM calls verified behind LiteLLM, audit at the endpoint, tool auth under OPA — with read-only default unchanged at each step.
- AC-A14-08 (REQ-A14-08): An AlertManager alert for a diagnostic-relevant condition arrives as a CloudEvent, is filtered by a Trigger, dispatched to HolmesGPT, and yields a finding/suggestion.
- AC-A14-09 (REQ-A14-09): A HolmesGPT request to raise diagnostics verbosity creates/updates a `LogLevel` only when OPA allows, and is audited.
- AC-A14-10 (REQ-A14-10): HolmesGPT actions emit `platform.audit.*` events (post-rewire) and OPA decisions emit `platform.policy.*`; namespaces are correct and versions present.
- AC-A14-11 (REQ-A14-11): HolmesGPT reads the gating OPA policies and returns a simulator-backed analysis of a denial.
- AC-A14-12 (REQ-A14-12): A new per-component toolset registers through A14's aggregation surface and becomes invokable.
- AC-A14-13 (REQ-A14-13): Docs, runbook (incl. rewiring), alerts, `GrafanaDashboard` XR, suggestion-card plugin, and tests exist.

## 10. Risks & Open Questions

- R-A14-1 (high): broad cross-namespace read access is a standing exfiltration/recon surface (compromised-agent / prompt-injection classes). Mitigation: read-only default + OPA-gated writes + full audit; principal item for B22 threat model. Blast radius high.
- R-A14-2 (med): phased rewiring spans many later components — risk of read-only default silently regressing during a rewire. Mitigation: each step independently tested + reversible (REQ-A14-07).
- R-A14-3 (med): self-referential OPA-read trade-off (HolmesGPT could find a self-bypass). Accepted explicitly (ADR 0012); mitigated by wanting it found/fixed + simulator analysis.
- R-A14-4 (low): A14 is W0/foundation with empty CSV upstream, yet functionally depends on A1/A5/A7/A18/B19 over time. Reconciliation note: these are acquired via rewiring (non-blocking dotted edges per waves.md), not pre-required — consistent with the phased trajectory.
- Open question (low): the concrete A2A interface identifier/version for HolmesGPT is design-time. `[PROPOSED — not in source]`; declare and pin at implementation.
- Open question (low): per-event-type names under `platform.audit.*`/`platform.policy.*` deferred to B12. `[PROPOSED]`.

## 11. References

- architecture-overview.md §6.10 (HolmesGPT, lines 774–836), §6.7 (eventing / AlertManager flow, 553+), §6.6 (OPA/audit points + policy simulator, 511–551), §6.4 (KB access, 349–368), §6.5 (observability), §14.1 (A14 row 1680).
- ADR 0012 (HolmesGPT first-class agent — phased trajectory, three-state model); ADR 0017 (Approval system); ADR 0002 (OPA gating); ADR 0034 (audit adapter); ADR 0035 (diagnostics toggle); ADR 0038 (policy simulator); ADR 0039 (Headlamp editors); ADR 0027 (threat model); ADR 0023 (env-specific Knative sources); ADR 0031/0030.
- interface-contract §1.2 (`Agent`), §1.5 (`LogLevel`, `Approval`), §2 (CloudEvent taxonomy + the AlertManager→HolmesGPT trigger flow), §3.1/§3.3 (SDK + A2A versioning).
- glossary (HolmesGPT, Interactive Access Agent, Knowledge Base, AlertManager). Related: A16, B10, A1, A7, A18, B19, A5.
