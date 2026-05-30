# SPEC ADR-0012 — HolmesGPT as a first-class Platform Agent [PROPOSED]

> kind: ADR · workstream: — · tier: T1
> upstream: [A14] · downstream: [A16, B10, A1, A7, A18, B19] · adrs: [0012] · views: [6.10]
> canon-glossary: 6aadcc2a4f38 · canon-interface: 54f5ede58e5f

## 1. Purpose & Problem Statement
ADR 0012 is a settled decision: **HolmesGPT** runs on the platform as a Platform Agent (an `Agent` CRD), not as an external operator tool, so every governance layer that binds a tenant agent binds the platform's own diagnostics agent. This SPEC states what honoring that decision obliges: HolmesGPT is declared/reconciled like any agent, traverses LiteLLM, is gated by OPA at a single central decision point, emits audit through the standard adapter, runs in a `Sandbox`, exposes an A2A interface, reaches the Knowledge Base only via CapabilitySet, and progressively rewires from blunt bootstrap guardrails to platform-mediated enforcement. It does not re-argue on-platform-vs-external.

The problem the decision solves: a self-management/diagnostics story available from day one without standing up a parallel ungoverned surface, with read-only default and incremental, OPA-gated, approval-routed expansion into write actions.

## 2. Scope
### 2.1 In scope
- The obligation that HolmesGPT is a Platform Agent (`Agent` CRD), administered like any other.
- The three-state OPA permission model (allowed / upon-approval / denied) at **one** centrally-edited OPA decision point.
- The bootstrap→rewire trajectory: blunt external read-only guardrails early; rewiring through LiteLLM, OPA, audit adapter, Approval system, sandbox as components land.
- The "every component contributes a HolmesGPT toolset" deliverable invariant.
- The AlertManager → HolmesGPT Knative trigger flow as one of two initial v1.0 flows.
- HolmesGPT driving the ADR 0035 dynamic toggle through the OPA-gated path.

### 2.2 Out of scope (and where it lives instead)
- HolmesGPT install/config/toolset build — component **A14** SPEC.
- The Approval mechanism itself — ADR 0017 / component **B19**.
- The OPA engine + policy library — ADR 0002 / **A7**, **B3**, **B16**.
- Knowledge Base primitive — ADR 0022 / `platform-knowledge-base` RAGStore.
- Coach Component self-improvement — component **B10**.
- Autonomous-remediation expansion policy — design-time (architecture-backlog §3.11).

## 3. Context & Dependencies
Upstream consumed: **A14** (HolmesGPT install, early phase) implements the obligations. Downstream consumers: **A16** (Interactive Access Agent hands off via A2A), **B10** (Coach observes alongside), **A1** LiteLLM (routing), **A7** OPA (gating), **A18** audit adapter, **B19** Approval system (upon-approval routing).

ADR decisions honored:
- **ADR 0012** (this) — HolmesGPT as Platform Agent; three-state model; bootstrap→rewire.
- **ADR 0001** — declared as an `Agent` CRD reconciled by ARK.
- **ADR 0002** — every action OPA-gated at one central decision point.
- **ADR 0017** — "upon-approval" routes through the generalized Approval system.
- **ADR 0018** — RBAC ServiceAccount remains the ceiling; OPA only restricts.
- **ADR 0034** — every action emits audit to Postgres + S3 (Postgres-only on kind); OpenSearch advisory (ADR 0009).
- **ADR 0035** — HolmesGPT requests the dynamic toggle through the OPA-gated write path.
- **ADR 0038 / 0039** — policies authored via Headlamp editors, validated by the policy simulator.
- **ADR 0027** — HolmesGPT is in-scope of the threat model, not privileged.

## 4. Interfaces & Contracts
### 4.1 CRDs / XRDs
- `Agent` (namespaced, ARK-owned) — HolmesGPT is one instance; uses `capabilitySetRefs[]` (to include `platform-knowledge-base` RAGStore), `sandboxTemplateRef`, `exposes` (A2A), `triggers` (AlertManager flow).
- `Approval` (namespaced, B19) — created when OPA classifies an action as upon-approval.
- `LogLevel` (namespaced, ADR 0035) — toggle surface HolmesGPT drives; owned by ADR 0035.

### 4.2 APIs / SDK surfaces
- A2A interface exposed by HolmesGPT so other Platform Agents hand off troubleshooting; A2A traffic brokered by LiteLLM.
- Knowledge Base access via the Platform SDK `rag.*` surface (B6) through an included CapabilitySet — identical to any agent.
- Observability toolset consumes Tempo + Mimir + Loki + Langfuse (ADR 0015).

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
- Consumes AlertManager alerts routed as CloudEvents; a Knative Trigger filters diagnostic-relevant ones (unschedulable pods, high error rates, sustained gateway latency) and dispatches.
- OPA decisions under `platform.policy.*`; approval lifecycle under `platform.approval.*`; HolmesGPT actions emit audit under `platform.audit.*`. Per-event-type names deferred to B12 registry.

### 4.4 Data schemas / connection-secret contracts
N/A — HolmesGPT introduces no substrate primitive; diagnostic state and audit ride existing primaries (audit `audit_events` Postgres table, ADR 0034).

## 5. OSS-vs-Custom Decision
N/A — ADR. (Enforcement note: upstream **HolmesGPT** installed/configured as component A14, version-pinned, wrapped as a Platform Agent; not forked. Rejected alternative — running it as an external operator tool bypassing the governance stack — is not built.)

## 6. Functional Requirements
- REQ-ADR-0012-01: HolmesGPT MUST be declared as an ARK `Agent` CRD and administered through the same surfaces as any Platform Agent; no parallel external operator surface in steady state.
- REQ-ADR-0012-02: HolmesGPT MUST expose an A2A interface (via `exposes`) brokered by LiteLLM so other Platform Agents can hand off troubleshooting.
- REQ-ADR-0012-03: HolmesGPT MUST reach the Knowledge Base only by including the `platform-knowledge-base` RAGStore in its CapabilitySet, via the SDK `rag.*` API — never as a built-in.
- REQ-ADR-0012-04: Every HolmesGPT action MUST resolve to exactly one of {allowed, upon-approval, denied} at a single, centrally-edited OPA decision point; no per-deployment policy drift and no allowlist embedded in HolmesGPT.
- REQ-ADR-0012-05: An upon-approval action MUST be routed through the generalized Approval system (ADR 0017) and execute only after a human approves; a denied action MUST refuse and emit audit.
- REQ-ADR-0012-06: During bootstrap, HolmesGPT MUST run with read-only cluster ServiceAccount + read-only AWS IAM role (no secret-store / sensitive data-plane access) and no write paths; as components land, access MUST be rewired through LiteLLM, OPA, the audit adapter, the Approval system, and the sandbox rather than the bootstrap shortcut.
- REQ-ADR-0012-07: In steady state, all calls into/out of HolmesGPT MUST traverse LiteLLM, be OPA-gated, emit audit through the standard adapter, and run inside a `Sandbox`; read-only remains the default and write expansion is incremental, one capability at a time.
- REQ-ADR-0012-08: The AlertManager → HolmesGPT Knative trigger flow MUST be implemented as one of the two initial v1.0 trigger flows.
- REQ-ADR-0012-09: HolmesGPT MUST drive the ADR 0035 dynamic toggle only through the same OPA-gated path as any other write action.
- REQ-ADR-0012-10: Every component's deliverable list MUST contribute a HolmesGPT toolset (standard deliverable invariant).

## 7. Non-Functional Requirements
- Security/tenancy: HolmesGPT's RBAC ServiceAccount is the hard ceiling (ADR 0018); OPA edits can never grant beyond it. Treated as in-scope by the threat model (ADR 0027); the self-referential trade-off (read access to the policies that gate it, for loophole analysis) is accepted and bounded by the RBAC ceiling.
- Observability (§6.5): consumes both Tempo (general) and Langfuse (LLM) correlated by `trace_id` (ADR 0015).
- Versioning: HolmesGPT version pinned; `Agent` CRD versioning per ADR 0030 (A5-owned).
- Reversibility: write expansion is reversible per-capability by reverting the central OPA bundle.

## 8. Cross-Cutting Deliverable Checklist
N/A — ADR (verification map lives in the PLAN). The §14.1 standard set is owned by component A14; this decision additionally *imposes* the "every component contributes a HolmesGPT toolset" item on every other component's §8.

## 9. Acceptance Criteria
- AC-ADR-0012-01: Honored when HolmesGPT appears as a reconciled `Agent` CRD with no agent pod running outside ARK/sandbox control. (REQ-01/07)
- AC-ADR-0012-02: Honored when another Platform Agent hands off a question to HolmesGPT over A2A through LiteLLM and receives a response. (REQ-02)
- AC-ADR-0012-03: Honored when removing the `platform-knowledge-base` RAGStore from HolmesGPT's CapabilitySet removes its KB access. (REQ-03)
- AC-ADR-0012-04: Honored when a single OPA decision point classifies a sampled action set into exactly one of the three states, with no second allowlist present. (REQ-04)
- AC-ADR-0012-05: Honored when an upon-approval action creates an `Approval` CRD and blocks until human decision; a denied action refuses and writes an audit record. (REQ-05)
- AC-ADR-0012-06: Honored when, in the bootstrap config, every write attempt fails and IAM has no secret-store reach; and a post-rewire action flows through LiteLLM+OPA+audit. (REQ-06)
- AC-ADR-0012-07: Honored when a steady-state action produces a LiteLLM routing record, an OPA decision, and an audit record, executed inside a sandbox. (REQ-07)
- AC-ADR-0012-08: Honored when an AlertManager alert produces a filtered CloudEvent that dispatches to HolmesGPT and yields a finding/suggestion/auto-PR. (REQ-08)
- AC-ADR-0012-09: Honored when a HolmesGPT toggle request is denied by OPA when the requesting identity lacks the permission, and granted when it does. (REQ-09)
- AC-ADR-0012-10: Honored when a sampled set of component specs each ship a HolmesGPT toolset deliverable. (REQ-10)

## 10. Risks & Open Questions
- OQ-1 (med): Bootstrap-to-rewire cutover ordering depends on which components have landed; partial rewiring leaves a mixed governance posture transiently. `[PROPOSED]` sequencing tracked with A14 plan.
- R-1 (high): Self-referential read access to gating policies is a deliberate trade-off; blast radius bounded by the RBAC ceiling (ADR 0018) and threat-model scope (ADR 0027).
- OQ-2 (low): Autonomous-remediation graduation criteria (upon-approval → allowed) are design-time (architecture-backlog §3.11).

## 11. References
- ADR 0012 (`adr/0012-holmesgpt-first-class-agent.md`) — the decision enforced here.
- architecture-overview.md §6.7 (eventing), §6.10 (self-management with HolmesGPT).
- Enforcing/related components: A14 (HolmesGPT, owner), A1 (LiteLLM), A7 (OPA), A18 (audit), B19 (Approval), A16 (handoff), B10 (Coach).
- Related ADRs: 0001, 0002, 0009, 0015, 0017, 0018, 0027, 0034, 0035, 0038, 0039.
