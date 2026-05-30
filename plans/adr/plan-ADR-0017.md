# PLAN ADR-0017 — Generalized approval system [PROPOSED]

> spec: SPEC-ADR-0017 · kind: ADR · tier: T0
> wave: authoring-parallel · estimate: M
> upstream-pieces: [B19, A3, A7] · downstream-pieces: [A14, B10, A9, A23, B16, A18, B12]

## 1. Implementation Strategy
Enforcement/verification map. ADR 0017 is enforced primarily by **B19** (the `Approval` CRD reconciler + Headlamp queue plugin), over **A3** (Argo Workflows providing suspend/resume/timeout/decision-event) and **A7** (OPA evaluating `approval.elevation`, elevate-only). The invariant — all approvals flow through one mechanism — is enforced by reuse pressure verified against the three concrete consumers: **A14** (HolmesGPT upon-approval), **B10** (Coach skill changes), and **A23** (Kargo human gate). **B16** owns the `approval.elevation` policy surface, **A18** the decision audit, **B12** the `platform.approval.*` schemas. Conformance is tested by proving a new approval need is met by writing an `Approval` (no parallel machinery), OPA can elevate-but-not-lower, the queue is filtered to the resolved level, and decisions emit both a CloudEvent and an audit record.

## 2. Ordered Task List
- TASK-01: Map each REQ to its enforcing component piece ID — produces: enforcement matrix — depends-on: []
- TASK-02: Verify B19 SPEC carries CRD + Argo-backing + Headlamp-queue + OPA-elevate obligations — produces: B19 conformance checklist — depends-on: [TASK-01]
- TASK-03: Verify OPA elevate-only contract (never lower; never grant beyond RBAC) with ADR 0018 — produces: elevation conformance note — depends-on: [TASK-01]
- TASK-04: Verify the three consumers reuse the mechanism (A14, B10, A23) — produces: reuse audit — depends-on: [TASK-01]
- TASK-05: Define Argo suspend/resume/timeout/decision-event conformance — produces: Chainsaw set — depends-on: [TASK-02]
- TASK-06: Define decision CloudEvent (`platform.approval.*`) + audit-emission conformance — produces: PyTest set — depends-on: [TASK-02]
- TASK-07: Verify Headlamp queue filtered to the resolved level — produces: A9 trace — depends-on: [TASK-02]
- TASK-08: Verify v1.0 scope exclusion (no M-of-N / delegation / escalation-timer / routing) — produces: scope lint — depends-on: [TASK-01]

## 3. Dependency Map
### 3.1 Upstream pieces that must ship first (HARD)
- B19 (approval system), A3 (Argo Workflows), A7 (OPA).
### 3.2 Downstream pieces blocked on this
- A14 (upon-approval), B10 (Coach), A23 (Kargo gate), B16 (`approval.elevation`).
### 3.3 Continuous (non-blocking) inputs
- ADR 0038 (policy simulator preview), ADR 0034 (audit), B12 schema registry, B22 threat model.

## 4. Parallelizable Subtasks
TASK-03, TASK-04, TASK-08 fan out once TASK-01 lands. TASK-05/06/07 fan out once TASK-02 lands.

## 5. Test Strategy
- AC-01 → Chainsaw: a new approval need satisfied by writing an `Approval`, no new controller/UI.
- AC-02 → PyTest: `Approval` carries requester + action attributes + default level + evidence refs.
- AC-03 → Chainsaw: `Approval` suspends, times out, emits decision event on resume.
- AC-04 → PyTest: OPA raises level; a lower/grant-beyond-RBAC attempt rejected.
- AC-05 → Playwright: queue shows `Approval` only to resolved-level holders; any such identity decides.
- AC-06 → PyTest: decision emits `platform.approval.*` event + one audit record.
- AC-07 → Chainsaw: HolmesGPT upon-approval + Coach skill-change flows each create `Approval`, no parallel path.
- AC-08 → Chainsaw: Kargo Stage gate creates `Approval`; Argo Workflow reports decision back to Kargo.
- AC-09 → scope lint: no M-of-N/delegation/escalation-timer/routing code ships.
- Fixtures/fakes: stub Argo suspend/resume engine, OPA elevation endpoint, audit endpoint for not-yet-landed A3/A7/A18.

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/authoring`
### 6.2 PR — `piece/ADR-0017-generalized-approval-system` → base `wave/authoring`; carries spec + plan.
### 6.3 Merge order — independent of sibling ADR specs; rolls up to main. Coordinate with ADR 0040 (Kargo) spec.

## 7. Effort Estimate
M overall (T0, invariant + three consumers). Per-task: TASK-01 S, 02 M, 03 S, 04 M, 05 M, 06 S, 07 S, 08 S. Critical path: TASK-01 → 02 → 05.

## 8. Rollback / Reversibility
Decision record; back out by reverting spec+plan. If reverted, the "one approval mechanism" invariant lapses and A14/B10/A23 could diverge into parallel flows; B16 loses its `approval.elevation` policy surface anchor. No runtime artifact is deleted. The deferred CRD-schema/timeout sub-decisions remain design-time (architecture-backlog §1.5) regardless.
