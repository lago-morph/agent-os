# SPEC ADR-0017 — Generalized approval system via Approval CRD + Argo Workflows + OPA elevation [PROPOSED]

> kind: ADR · workstream: — · tier: T0
> upstream: [B19, A3, A7] · downstream: [A14, B10, A9, A23, B16, A18, B12] · adrs: [0017] · views: [7.5]
> canon-glossary: 6aadcc2a4f38 · canon-interface: 54f5ede58e5f

## 1. Purpose & Problem Statement
ADR 0017 is a settled T0 decision: the platform ships **one** approval mechanism in four parts — an `Approval` CRD (the request), Argo Workflows (suspend/resume, timeout, decision-event emission), OPA (consulted on creation, may **only elevate** the required level, never lower it), and a Headlamp plugin (the approval queue UI filtered to holders of the resolved level). New approval needs are met by writing an `Approval` from the requesting agent, never by building parallel machinery. This SPEC states the constraints, interfaces, and conformance the decision imposes as a platform-wide invariant. It does not re-argue generalize-now vs rebuild-on-need.

The problem the decision solves: two parallel approval flows (Coach skill changes, HolmesGPT remediation) already create governance/UX divergence, and more approval surfaces (capability registration, budget exceptions, dynamic MCP registration, virtual-key overrides) are expected; one mechanism is cheaper than three.

## 2. Scope
### 2.1 In scope
- The invariant: all approvals flow through `Approval` CRD + Argo Workflows + Headlamp plugin; new types reuse it.
- The four-part contract: CRD shape obligations, Argo suspend/resume/timeout/decision-event, OPA elevate-only, Headlamp queue filtered to the resolved level.
- The "request a level, OPA may elevate, anyone with that level decides" model; RBAC remains the floor.
- Decision emits a `platform.approval.*` CloudEvent back to the requesting agent + an audit event (ADR 0034).
- The three concrete consumers: Coach skill changes, HolmesGPT remediation (incl. its "upon-approval" state, ADR 0012), and Kargo's human gate (ADR 0040).
- Reuse pressure: any future approval surface must justify not using this mechanism.

### 2.2 Out of scope (and where it lives instead)
- Approval system implementation — component **B19** SPEC.
- Exact `Approval` CRD schema (first-class fields vs metadata blob), timeout/escalation behavior, shared-vs-per-type Argo template, approve/reject audit detail level — design-time (architecture-backlog §1.5).
- M-of-N approvers, delegation, escalation timers, complex routing — out of v1.0 (future-enhancements §4).
- Argo Workflows install — component **A3**; OPA engine — **A7**; `approval.elevation` policy content — **B16**.
- Policy-change preview — ADR 0038 (Headlamp policy simulator).

## 3. Context & Dependencies
Upstream consumed: **B19** (generalized approval system) implements the mechanism; **A3** Argo Workflows (suspend/resume engine underneath each `Approval`); **A7** OPA (elevation decision). Downstream consumers: **A14** HolmesGPT (upon-approval), **B10** Coach (skill changes), **A9** Headlamp (queue plugin), **A23** Kargo (human gate), **B16** OPA `approval.elevation` content, **A18** audit, **B12** schema registry.

ADR decisions honored:
- **ADR 0017** (this) — one mechanism; CRD + Argo + OPA-elevate-only + Headlamp; new types reuse.
- **ADR 0002** — OPA engine evaluates `approval.elevation`; Gatekeeper admits the `Approval` CRD.
- **ADR 0018** — OPA may only elevate the required level, never lower it; RBAC is the floor; an action stays unapprovable by anyone RBAC would not allow to act.
- **ADR 0012** — HolmesGPT's "upon-approval" state's entire mechanic is creating an `Approval`.
- **ADR 0031** — decisions emit under `platform.approval.*` (escalation deferred).
- **ADR 0034** — decision audit rides Postgres + S3 (Postgres-only on kind); OpenSearch advisory.
- **ADR 0038** — policy simulator dry-runs `approval.elevation` before submission.
- **ADR 0040** — Kargo composes with (does not subsume) this mechanism as its human gate.

## 4. Interfaces & Contracts
### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
- `Approval` (namespaced; reconciled by Argo Workflow + B19) — `requestingAgent`, `actionType`, `actionAttributes`, `defaultLevel`, `evidenceRefs[]`, `decision`, `decidedBy`, `decidedAt`. Source-stated fields; whether first-class vs metadata-blob is design-time (architecture-backlog §1.5) — `[PROPOSED — not in source]` for any field beyond this set.
- Versioning: `v1alpha1`→`v1` per ADR 0030; breaking change = new `vN` group + conversion webhook; B19 owns the lifecycle.

### 4.2 APIs / SDK surfaces
- A requesting agent creates an `Approval` (via the platform SDK / kubectl path) rather than invoking bespoke approval machinery; the Argo Workflow underneath manages suspend/resume, timeout, and decision-event emission.
- Headlamp approval-queue plugin filtered to anyone holding the resolved (possibly OPA-elevated) level.
- Kargo creates an `Approval` instance for a Stage gate; the backing Argo Workflow reports the decision back to Kargo (ADR 0040).

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
- Emits under `platform.approval.*`: approval requested, OPA-elevated, decided (approved/rejected), timed out — back to the requesting agent. Escalation events deferred (ADR 0017). Per-event-type schemas live in B12 registry.
- Emits an audit event under `platform.audit.*` per decision.

### 4.4 Data schemas / connection-secret contracts
- Decision audit rows ride the audit `audit_events` Postgres table (ADR 0034); no new connection secret introduced.

## 5. OSS-vs-Custom Decision
N/A — ADR. (Enforcement note: composes upstream **Argo Workflows** (A3) for suspend/resume and **OPA** (A7) for elevation, with a custom **Headlamp** queue plugin and the custom `Approval` CRD reconciler (B19) — build-new for the CRD/plugin, config/wrap for Argo+OPA. Rejected alternative — rebuild-on-need with parallel per-flow machinery — is forbidden by the invariant.)

## 6. Functional Requirements
- REQ-ADR-0017-01: All human-approval needs MUST flow through one mechanism — an `Approval` CRD reconciled by an Argo Workflow with a Headlamp queue plugin; new approval types MUST reuse it rather than build parallel machinery.
- REQ-ADR-0017-02: Each `Approval` MUST carry the requesting agent, action attributes/metadata sufficient to act on the proposal independently, a default approval level, and evidence references.
- REQ-ADR-0017-03: An Argo Workflow MUST run underneath each `Approval`, managing suspend/resume mechanics, timeout, and decision-event emission.
- REQ-ADR-0017-04: OPA MUST be consulted on `Approval` creation and MAY only **elevate** the required approval level — never lower it; RBAC MUST remain the floor and OPA MUST NOT make an action approvable by someone RBAC would not allow to act (ADR 0018).
- REQ-ADR-0017-05: A Headlamp plugin MUST provide the approval queue UI filtered to anyone holding the resolved (post-elevation) level; "request a level, OPA may elevate, anyone with that level decides" is the model.
- REQ-ADR-0017-06: A decision MUST emit a `platform.approval.*` CloudEvent back to the requesting agent AND an audit event through the platform audit adapter (Postgres + S3 SoR; Postgres-only on kind; OpenSearch advisory).
- REQ-ADR-0017-07: HolmesGPT's "upon-approval" state (ADR 0012) MUST be realized solely by creating an `Approval` here; Coach skill changes and HolmesGPT remediation MUST be the two initial use cases.
- REQ-ADR-0017-08: Kargo (ADR 0040) MUST use the `Approval` CRD as its human Stage gate, with the backing Argo Workflow reporting the decision back to Kargo; neither system subsumes the other.
- REQ-ADR-0017-09: M-of-N approvers, delegation, escalation timers, and complex routing MUST NOT be in v1.0 scope.

## 7. Non-Functional Requirements
- Security: OPA-elevate-only is the security contract (ADR 0018) — an elevation can never widen who may act; couples B16 to the `approval.elevation` policy surface.
- Observability (§6.5): approval traceability is uniform across action types via `platform.approval.*` + audit; admins preview elevation impact via the policy simulator (ADR 0038) before submitting a policy change.
- Versioning (ADR 0030): `Approval` CRD versioned per-component (B19-owned), conversion-webhook-backed.
- Reuse: any future approval surface MUST justify why it would not flow through this mechanism; the default is "use the `Approval` CRD."

## 8. Cross-Cutting Deliverable Checklist
N/A — ADR (verification map in the PLAN). The §14.1 set is owned by the enforcing components (B19, A3, A7, A9, B16).

## 9. Acceptance Criteria
- AC-ADR-0017-01: Honored when a new approval need is satisfied by writing an `Approval` CRD with no new approval-specific controller/UI built. (REQ-01)
- AC-ADR-0017-02: Honored when an `Approval` carries requesting agent + action attributes + default level + evidence refs sufficient for an approver to decide independently. (REQ-02)
- AC-ADR-0017-03: Honored when an `Approval` suspends via its Argo Workflow, times out per spec, and emits a decision event on resume. (REQ-03)
- AC-ADR-0017-04: Honored when OPA can raise the required level on creation but a policy attempting to lower it (or grant beyond RBAC) is rejected/ineffective. (REQ-04)
- AC-ADR-0017-05: Honored when the Headlamp queue shows the `Approval` only to identities holding the resolved level, and any such identity can decide. (REQ-05)
- AC-ADR-0017-06: Honored when a decision emits a `platform.approval.*` event to the requester and one audit record to the SoR. (REQ-06)
- AC-ADR-0017-07: Honored when HolmesGPT upon-approval and Coach skill-change flows each create `Approval` CRDs and no parallel mechanism. (REQ-07)
- AC-ADR-0017-08: Honored when a Kargo Stage gate creates an `Approval` and the Argo Workflow reports the decision back to Kargo. (REQ-08)
- AC-ADR-0017-09: Honored when no M-of-N / delegation / escalation-timer / routing code ships in v1.0. (REQ-09)

## 10. Risks & Open Questions
- OQ-1 (med): Exact `Approval` CRD schema (first-class fields vs generic metadata blob), timeout/escalation behavior, shared-vs-per-type Argo template, and approve/reject audit detail level are design-time (architecture-backlog §1.5); several ACs are partial until B19 design lands. `[PROPOSED]`
- R-1 (low): Reuse pressure depends on review discipline; a team could re-implement approval out-of-band — mitigated by the invariant being an audit/review gate.
- R-2 (low): Richer features (multi-party sign-off, time-bounded escalation, requester-history routing) are revisit triggers (architecture-backlog §3.3), added where the mechanism lives, not by rebuilding.

## 11. References
- ADR 0017 (`adr/0017-generalized-approval-system.md`) — the decision enforced here.
- architecture-overview.md §7.5 (approvals — generalized); architecture-backlog.md §1.5, §3.3, §6; future-enhancements.md §4.
- Enforcing components: B19 (approval system, owner), A3 (Argo Workflows), A7 (OPA), A9 (Headlamp queue), B16 (`approval.elevation`), A18 (audit), A23 (Kargo gate).
- Related ADRs: 0002, 0012, 0018, 0031, 0034, 0038, 0040.
