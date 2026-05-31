# SPEC B19 — Generalized approval system (B19-core)

> kind: COMPONENT · workstream: B · tier: T0
> upstream: [A3, A7, A9, B5, B12] · downstream: [A23] · adrs: [0017, 0018, 0034, 0031, 0030, 0038, 0040, 0012] · views: [7.5, 6.6]
> canon-glossary: b0edae10a2e649ba06e2b184dc938235aab758e3 · canon-interface: 0ce201d5d5af5cffcf09b647ea4a902a47596d36

## 1. Purpose & Problem Statement

B19 **owns the `platform.approval` event namespace** (D-02 / QN-03): it authors the `platform.approval.*` CloudEvent schemas and registers them in the event catalogue (B12). Consumers (promotion pipeline, dashboard, and any other approval-event consumer) take an explicit dependency on B19 for these schemas and never co-own the namespace. Approval decisions are recorded in the audit log; audit emission is gated on the audit-adapter freeze-gate (D-05). Any security-relevant event B19 detects (e.g. approval-gate bypass or tampering) MUST also be emitted under `platform.security` (schema owned by A7), in addition to local handling.

B19 is the platform's **one generalized approval mechanism**: a human gate any Platform Agent (or
external system like Kargo) can wire into without rebuilding machinery. ADR 0017 reverses the
earlier "do not generalize" stance and commits to a single mechanism with four parts — an
`Approval` CRD, Argo Workflows managing suspend/resume/timeout/decision-emission, OPA consulted on
creation to **only elevate** the required level (never lower), and a Headlamp approval-queue UI.
Decisions emit a `platform.approval.*` CloudEvent back to the requesting agent plus an audit event.

**This spec is B19-core.** Per `_meta/waves.md`, the B19→A23→B5 cycle is resolved by splitting
B19: **B19-core (W3)** ships the `Approval` CRD, the OPA elevation surface integration, the Argo
Workflow integration, and CloudEvent decision emission — this is what A23 (Kargo, W4) depends on
and it lands first. The **Headlamp approval-queue UI (B19-ui) ships in W4 alongside B5** (B5 owns
the cross-cutting Headlamp plugins, including the Kargo plugin and the approval-queue UI). B19-core
is built and testable without the UI by enqueuing to a queue B5/B19-ui later renders. Initial v1.0
consumers: Coach skill changes (B10) and HolmesGPT remediation (A14/ADR 0012 "upon-approval" state).

## 2. Scope

### 2.1 In scope (B19-core)
- The **`Approval` CRD** (namespaced) with the source-stated fields: `requestingAgent`,
  `actionType`, `actionAttributes`, `defaultLevel`, `evidenceRefs[]`, `decision`, `decidedBy`,
  `decidedAt` (interface-contract §1.5).
- The **Argo Workflow integration** underneath each `Approval`: suspend/resume mechanics, timeout,
  and decision-event emission (ADR 0017; no escalation in v1.0).
- The **OPA elevation integration**: on `Approval` creation, consult OPA's `approval.elevation`
  surface (the Rego is owned by B16); apply the resolved level (default or higher, never lower).
- **CloudEvent decision emission** under `platform.approval.*` (requested / OPA-elevated / decided
  approved-or-rejected / timed-out) routed back to the requesting agent.
- **Audit emission** through the platform audit adapter (ADR 0034) for request created, OPA
  elevation decision, approver decision, timeout fired, and CloudEvent delivery (§6.6 audit points).
- The **enqueue contract** to the approval queue (the data B19-ui/B5 renders): the resolved level +
  filter so the queue surfaces to anyone holding the resolved level. B19-core owns the queue *state*;
  B5/B19-ui owns the *rendering*.
- The **dry-run hook** for `approval.elevation` so the policy simulator (A20, ADR 0038) can preview
  approval impact of an OPA policy change (B19 exposes the elevation evaluation in dry-run; honors
  `simulated`).
- The **Kargo integration contract** (ADR 0040): how Kargo (A23) creates an `Approval` for a Stage
  human gate and reads the decision back — the interface A23 binds to.

### 2.2 Out of scope (and where it lives instead)
- **The Headlamp approval-queue UI / plugin (B19-ui)** → ships in **W4 with B5** (cross-cutting
  Headlamp plugins). B19-core defines the enqueue contract; B5/B19-ui renders it.
- **The `approval.elevation` Rego content** → B16 (B19-core integrates and consults it).
- **Argo Workflows install** → A3 (B19-core authors the workflow template that runs on it).
- **OPA / Gatekeeper install** → A7.
- **CloudEvent per-event-type schemas** under `platform.approval.*` → B12 registry.
- **The audit adapter library / endpoint** → A18 (B19-core links the adapter, doesn't own it).
- **Kargo itself** → A23 (consumes B19-core's `Approval` integration contract).
- **M-of-N approvers, delegation, escalation timers, complex routing** → explicitly out of v1.0
  (ADR 0017 / future-enhancements §4).
- **Requesting-agent SDK callback handling** of the decision event → platform SDK (B6) / agent code.

## 3. Context & Dependencies

**Upstream consumed (HARD):**
- **A3 (Argo Workflows)** — the workflow engine B19-core's `Approval` workflow template runs on
  (suspend/resume/timeout/decision-emit).
- **A7 (OPA / Gatekeeper)** — the OPA engine consulted for `approval.elevation` on creation.
- **A9 (Headlamp install + framework)** — the plugin framework B19-ui/B5 later builds the queue on;
  B19-core depends on A9 only for the enqueue/visibility contract, not UI code.
- **B5 (cross-cutting Headlamp plugins)** — co-lands the approval-queue UI in W4 (downstream of
  B19-core for the UI portion; the CSV carries the full B19 edge set, `B5 --> B19` for the UI).
- **B12 (CloudEvent schema registry)** — owns the `platform.approval.*` per-event-type schemas
  B19-core emits against.

> Per the CSV, B19's `downstream` is **A23** and B19's wave is **W3** (the core is the critical
> path). B5 sits in W4; the `B5 --> B19` edge is the **UI** dependency, satisfied by B19-ui co-landing
> with B5 in W4 (see `_meta/waves.md` cycle resolution).

**Downstream consumers (HARD):**
- **A23 (Kargo, W4)** — composes with the `Approval` CRD as its human gate during environment
  promotion; creates an `Approval` and reads the decision back via B19-core's Argo workflow.

**ADRs honored:**
- **ADR 0017** — one approval mechanism (CRD + Argo + OPA-elevation + Headlamp); OPA may only
  elevate; no escalation/M-of-N/delegation in v1.0; decisions emit `platform.approval.*` + audit.
- **ADR 0018** — OPA may only elevate the required level / deny, never lower or grant beyond RBAC.
- **ADR 0034** — decision audit through the adapter; Postgres+S3 system of record (Postgres-only on
  kind); OpenSearch advisory fanout.
- **ADR 0031** — `platform.approval.*` is the closed-set namespace; new event types mint rather than
  break subscribers.
- **ADR 0038** — `approval.elevation` is dry-runnable for the policy simulator's approval-impact
  preview, honoring `simulated`.
- **ADR 0040** — Kargo composes with `Approval` as its Stage human gate; the two systems compose.
- **ADR 0012** — HolmesGPT "upon-approval" state's mechanic is creating an `Approval` here.
- **ADR 0030** — `Approval` CRD versioning owned by B19.

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
**`Approval`** (namespaced; owner/reconciler = Argo Workflow + B19 — interface-contract §1.5):

| Field (source-stated) | Meaning |
|---|---|
| `requestingAgent` | the Platform Agent (or external system) requesting the action |
| `actionType` | the kind of action proposed |
| `actionAttributes` | all metadata needed to act on the proposal independently if approved |
| `defaultLevel` | requested approval level (e.g. `operator`) — OPA may elevate, never lower |
| `evidenceRefs[]` | links to evidence (traces, audit, RAG references) |
| `decision` | the recorded decision (approved/rejected) |
| `decidedBy` | the approver identity |
| `decidedAt` | decision timestamp |

`[PROPOSED — not in source]`: ADR 0017 explicitly defers "first-class fields vs generic metadata
blob," timeout value/behaviour, whether the Argo template is shared or per-`actionType`, and audit
detail level for approve vs reject. B19-core proposes: first-class fields as above (matching §1.5),
a single shared Argo template parameterized by `actionType`, a `status` carrying the **resolved**
level + workflow ref + timeout deadline, and equal audit detail for approve and reject. All tagged
`[PROPOSED — not in source]`; the level-name vocabulary beyond the example `operator`/`org-admin`
(§7.5) is `[PROPOSED — not in source]`.

### 4.2 APIs / SDK surfaces
- **Enqueue contract** (to B19-ui/B5): the queue exposes pending `Approval`s filtered by **resolved
  level**; B19-core owns the queue state, B5/B19-ui the rendering. The concrete query/field surface
  is `[PROPOSED — not in source]` (§7.5 states only "workqueue filtered by resolved level").
- **Kargo integration contract** (to A23): Kargo creates an `Approval` (Stage gate) and the backing
  Argo Workflow reports the decision back to Kargo (ADR 0017/0040). The exact handshake fields are
  `[PROPOSED — not in source]`.
- **Dry-run surface** (to A20): the `approval.elevation` evaluation is invocable in dry-run, emitting
  the B3 decision shape with `simulated: true` (ADR 0038). The Rego itself is B16's.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
**Emitted under `platform.approval.*`** (interface-contract §2): approval requested, OPA-elevated,
decided (approved/rejected), timed out. Concrete per-event-type **names/schemas live in B12**;
B19-core emits against them and MUST NOT mint a new top-level namespace. The decision event is
routed back to `requestingAgent`. **Consumed:** none required for core operation beyond the approver
decision resuming the Argo Workflow. (Escalation deferred — ADR 0017.)

### 4.4 Data schemas / connection-secret contracts
No dedicated datastore owned by B19-core beyond the `Approval` CRD state + Argo Workflow state.
Audit records flow through the **A18 adapter** to the Postgres+S3 system of record (Postgres-only on
kind) — B19-core does not write audit directly (ADR 0034). Any backing store needed is provisioned
via B4 XRDs and consumed through the connection-secret contract; B19-core does not define one.

## 5. OSS-vs-Custom Decision
**Custom integration (build-new) over OSS Argo Workflows (A3) + OSS OPA (A7).** Upstream: Argo
Workflows (install A3; no version pinned in source — `[PROPOSED]`), OPA (A7). Approach: **build-new**
the `Approval` CRD + reconciliation + Argo workflow template + CloudEvent/audit emission; **wrap**
Argo's suspend/resume and OPA's evaluation. Rationale (ADR 0017): a generic CRD + Argo template +
(later) Headlamp plugin is a small enough increment to build once rather than maintain parallel
per-use-case flows.

## 6. Functional Requirements
- **REQ-B19-01:** B19-core MUST define the namespaced **`Approval` CRD** with the source-stated
  fields (`requestingAgent`, `actionType`, `actionAttributes`, `defaultLevel`, `evidenceRefs[]`,
  `decision`, `decidedBy`, `decidedAt`).
- **REQ-B19-02:** On `Approval` creation, B19-core MUST consult OPA's `approval.elevation` surface
  and apply the **resolved** required level, which may equal or exceed `defaultLevel` and **never**
  be lower (ADR 0017/0018).
- **REQ-B19-03:** B19-core MUST run an **Argo Workflow** beneath each `Approval` managing
  suspend/resume, **timeout**, and decision-event emission (ADR 0017; no escalation in v1.0).
- **REQ-B19-04:** B19-core MUST **enqueue** each pending `Approval` to the approval queue filtered by
  the **resolved level**, exposing the enqueue contract that B5/B19-ui renders (§7.5).
- **REQ-B19-05:** On approve **or** reject, B19-core MUST record `decision`/`decidedBy`/`decidedAt`,
  emit an **audit event** through the A18 adapter, and emit a **`platform.approval.*` CloudEvent**
  routed to `requestingAgent` (ADR 0017/0034/0031).
- **REQ-B19-06:** On **timeout**, B19-core MUST fire the timeout, audit it, and emit the timed-out
  `platform.approval.*` event (ADR 0017; §6.6 audit points).
- **REQ-B19-07:** B19-core MUST expose the `approval.elevation` evaluation in **dry-run** so the
  policy simulator (A20) can preview approval impact, honoring `simulated` with no enforcement-path
  side effect (ADR 0038).
- **REQ-B19-08:** B19-core MUST provide the **Kargo integration contract**: Kargo creates an
  `Approval` for a Stage gate and the backing Argo Workflow reports the decision back to Kargo
  (ADR 0040), without B19-core depending on Kargo.
- **REQ-B19-09:** B19-core MUST NOT implement M-of-N approvers, delegation, escalation timers, or
  complex routing (out of v1.0 — ADR 0017); the model is "request a level, OPA may elevate, anyone
  with that level decides."
- **REQ-B19-10:** The `Approval` CRD MUST follow ADR 0030 versioning (owned by B19).
- **REQ-B19-11:** B19-core MUST be operable and testable **without** the Headlamp UI by exercising
  create→elevate→enqueue→decide(via API/Argo resume)→emit, so the W3/W4 split holds.

## 7. Non-Functional Requirements
- **Security:** OPA-as-restrictor on elevation (REQ-B19-02) — never lowers, never makes an action
  approvable by someone RBAC would not allow to act (ADR 0018). Decisions are fully audited
  (ADR 0034) so approval traceability is uniform regardless of `actionType`.
- **Multi-tenancy (§6.9):** `Approval` is namespaced; the resolved-level filter + RBAC scope who can
  decide; no cross-tenant approval bridging.
- **Observability (§6.5):** every state transition (requested/elevated/decided/timed-out) emits an
  audit event and a `platform.approval.*` CloudEvent; dry-run elevation decisions are observable
  under `platform.policy.*` via the simulator path.
- **Scale:** lean by design (ADR 0017); one shared Argo template parameterized by `actionType`
  `[PROPOSED]`; no per-type machinery.
- **Versioning (ADR 0030):** `Approval` CRD owned by B19; the `platform.approval.*` event schemas are
  owned by B12; breaking changes mint new event types.

## 8. Cross-Cutting Deliverable Checklist
- Helm/manifests — **applicable** (`Approval` CRD, Argo workflow template, controller manifests).
- Per-product docs (10.5) — **applicable** ("how to request an approval from an agent", Kargo gate).
- Runbook (10.7) — **applicable** (stuck/suspended workflow recovery, timeout tuning, replay).
- Alerts — **applicable** (approvals pending past timeout, elevation evaluation failures).
  `[PROPOSED — not in source]`.
- Grafana dashboard (Crossplane XR) — **applicable** — approval-queue depth / decision-latency
  dashboard authored as a `GrafanaDashboard` XR.
- Headlamp plugin — **deferred to B19-ui / B5 (W4)** — the approval-queue UI ships with B5;
  B19-core defines the enqueue contract only.
- OPA/Rego integration — **applicable (integration)** — consults `approval.elevation` (Rego in B16);
  exposes dry-run for the simulator.
- Audit emission (ADR 0034) — **applicable** — emits decision/timeout/elevation audit via the
  A18 adapter.
- Knative trigger flow — **applicable** — emits `platform.approval.*` decision CloudEvent back to the
  requesting agent (schemas in B12).
- HolmesGPT toolset — **N/A** — HolmesGPT is a *consumer* (creates `Approval`s via its "upon-approval"
  state, ADR 0012), not a B19 toolset.
- 3-layer tests — **applicable** (Chainsaw for `Approval`→elevation→Argo→decision; PyTest for
  elevation-integration + emission logic; Playwright **deferred to B19-ui** — no UI in core).
- Tutorials & how-tos — **applicable** ("wire a new approval surface via the `Approval` CRD").

## 9. Acceptance Criteria
- **AC-B19-01:** Creating an `Approval` with all source-stated fields admits and is reconciled.
  (→ REQ-B19-01)
- **AC-B19-02:** An `Approval` whose action OPA marks sensitive is **elevated** above `defaultLevel`;
  no test case ever resolves a level **below** `defaultLevel`. (→ REQ-B19-02)
- **AC-B19-03:** Each `Approval` launches a suspended Argo Workflow that resumes on decision and
  fires on timeout. (→ REQ-B19-03)
- **AC-B19-04:** A pending `Approval` appears in the queue filtered to the resolved level; the
  enqueue contract returns it for a holder of that level and not for others. (→ REQ-B19-04)
- **AC-B19-05:** Approve and reject each record `decision`/`decidedBy`/`decidedAt`, emit an audit
  event via the adapter, and emit a `platform.approval.*` CloudEvent to `requestingAgent`.
  (→ REQ-B19-05)
- **AC-B19-06:** A timed-out `Approval` fires the timeout, audits it, and emits the timed-out
  `platform.approval.*` event. (→ REQ-B19-06)
- **AC-B19-07:** The policy simulator dry-runs `approval.elevation` against a sample `Approval` and
  returns `simulated: true` with no state change. (→ REQ-B19-07)
- **AC-B19-08:** A simulated Kargo Stage gate creates an `Approval` and receives the decision back
  through the Argo Workflow (contract test with an A23 fake). (→ REQ-B19-08)
- **AC-B19-09:** No M-of-N / delegation / escalation / routing code path exists; a single resolved
  level governs who may decide. (→ REQ-B19-09)
- **AC-B19-10:** An `Approval` CRD XR schema change ships a new `vN` with conversion and ≥1-release
  deprecation. (→ REQ-B19-10)
- **AC-B19-11:** The full create→elevate→enqueue→decide(via Argo resume API)→emit cycle passes with
  **no Headlamp UI present**, proving the W3/W4 split. (→ REQ-B19-11)

## 10. Risks & Open Questions
- **R1 (high):** The B19→A23→B5 cycle. Mitigation (per `_meta/waves.md`): B19-core (W3) ships
  CRD+OPA+Argo+CloudEvent with no UI dependency; B19-ui co-lands with B5 (W4). REQ-B19-11 guards the
  split with a UI-less acceptance test. Blast radius: high if the split leaks (UI coupling sneaks
  into core).
- **R2 (med):** `Approval` first-class-fields-vs-metadata-blob, timeout behaviour, shared-vs-per-type
  Argo template, and approve/reject audit detail are **deferred by ADR 0017** (`[PROPOSED]` choices
  made here). If a later approval surface needs a per-type template, the shared-template choice must
  flex.
- **R3 (med):** Approval-level vocabulary beyond `operator`/`org-admin` is `[PROPOSED]`; must align
  with Keycloak `platform_roles`/`tenant_roles` claims so the resolved-level filter is enforceable.
- **R4 (med):** The Kargo integration handshake fields are `[PROPOSED]`; A23 binds to them — freeze
  the contract before A23 lands (B19-core is W3, A23 W4, so order is favourable).
- **R5 (low):** `platform.approval.*` per-event schemas are owned by B12; B19-core needs them frozen
  to emit — fixtures used until B12 lands.
- **OQ1:** Does the enqueue queue live in `Approval`/Argo state alone, or a separate index B19-ui
  reads? `[PROPOSED]` derive the queue from `Approval` CRD list + resolved level in status; no
  separate store.
- **OQ2:** Does Kargo create the `Approval` directly, or via an adapter B19-core ships? `[PROPOSED]`
  Kargo creates it directly per ADR 0040 ("Kargo creates an `Approval` CRD instance").

## 11. References
- architecture-overview.md §7.5 (lines 1294–1362: approval primitives, RBAC/OPA elevation semantics,
  flow + sequence diagrams, "keep it thin" v1.0 scope), §6.6 (line 527: approval-workflow audit
  emission points; lines 540 approval-system OPA decision point).
- ADR 0017 (generalized approval — primary; deferred sub-decisions; Kargo composition), 0018
  (RBAC-floor / OPA-restrictor — elevation only), 0034 (audit pipeline), 0031 (`platform.approval.*`
  taxonomy), 0038 (policy simulator dry-run of elevation), 0040 (Kargo human gate), 0012 (HolmesGPT
  upon-approval consumer), 0030 (versioning).
- interface-contract.md §1.5 (`Approval` fields), §2 (`platform.approval.*`), §5 (audit adapter).
- _meta/waves.md (B19→A23→B5 cycle resolution; B19-core W3 / B19-ui W4).
- Related pieces: A3 (Argo), A7 (OPA), A9 (Headlamp framework), B5/B19-ui (queue UI, W4), B12 (event
  schemas), A23 (Kargo), B16 (`approval.elevation` Rego), A20 (simulator), A18 (audit adapter),
  B10/A14 (initial consumers).
