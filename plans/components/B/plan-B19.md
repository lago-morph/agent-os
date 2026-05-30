# PLAN B19 — Generalized approval system (B19-core)

> spec: SPEC-B19 · kind: COMPONENT · tier: T0
> wave: W3 (B19-core) · estimate: L
> upstream-pieces: [A3, A7, A9, B5, B12] · downstream-pieces: [A23]

## 1. Implementation Strategy
Build **B19-core** — the `Approval` CRD, the OPA elevation integration, the Argo Workflow
integration, and CloudEvent/audit decision emission — as a UI-less, fully testable unit so the
B19→A23→B5 cycle stays acyclic (`_meta/waves.md`): B19-core (W3) lands first; the Headlamp
approval-queue UI (**B19-ui**) co-lands with **B5 (W4)**. The build order is CRD first, then the Argo
workflow template (suspend/resume/timeout/decision-emit), then wire OPA `approval.elevation` on
creation (Rego owned by B16), then CloudEvent + audit emission, then the enqueue contract that
B5/B19-ui later renders, then the Kargo integration contract and the dry-run surface for the policy
simulator (A20). A UI-less end-to-end acceptance test (REQ-B19-11) is the guard that the split holds.

## 2. Ordered Task List
- **TASK-01:** Define the `Approval` CRD (namespaced) with source-stated fields + a `status` carrying
  resolved level / workflow ref / timeout deadline `[PROPOSED]` — produces: `Approval` CRD — depends-on: []
- **TASK-02:** Argo Workflow template beneath each `Approval` (suspend/resume, timeout, decision-emit);
  single shared template parameterized by `actionType` `[PROPOSED]` — produces: Argo template +
  controller wiring — depends-on: [TASK-01]
- **TASK-03:** OPA `approval.elevation` integration on creation — consult OPA, apply resolved level
  (elevate-only, never lower) — produces: elevation integration — depends-on: [TASK-01]
- **TASK-04:** Enqueue contract — derive the queue from `Approval` list + resolved level in status,
  filtered for holders of that level (the surface B5/B19-ui renders) — produces: enqueue contract —
  depends-on: [TASK-02, TASK-03]
- **TASK-05:** Decision path — record `decision`/`decidedBy`/`decidedAt` on approve/reject, audit via
  the A18 adapter, emit `platform.approval.*` CloudEvent to `requestingAgent` — produces: decision +
  emission — depends-on: [TASK-02]
- **TASK-06:** Timeout path — fire/audit/emit timed-out event (no escalation, ADR 0017) — produces:
  timeout handling — depends-on: [TASK-02]
- **TASK-07:** Dry-run surface for `approval.elevation` (A20 simulator preview, honors `simulated`) —
  produces: dry-run hook — depends-on: [TASK-03]
- **TASK-08:** Kargo integration contract — Kargo creates an `Approval` Stage gate; the Argo Workflow
  reports the decision back (ADR 0040) — produces: Kargo contract — depends-on: [TASK-05]
- **TASK-09:** UI-less end-to-end test (create→elevate→enqueue→decide-via-Argo-resume→emit) proving the
  W3/W4 split (REQ-B19-11) — produces: e2e test — depends-on: [TASK-04, TASK-05, TASK-06]
- **TASK-10:** CRD versioning (ADR 0030), runbook (stuck/suspended-workflow recovery, timeout tuning),
  operability `GrafanaDashboard` claim (queue depth / decision latency), alerts, how-tos — produces:
  versioning + docs + dashboard — depends-on: [TASK-09]

## 3. Dependency Map
### 3.1 Upstream pieces that must ship first (HARD)
- **A3 (Argo Workflows)** — workflow engine for suspend/resume/timeout. **A7 (OPA)** — engine for
  `approval.elevation`. **A9 (Headlamp framework)** — only the enqueue/visibility contract for core;
  UI is B19-ui/B5. **B12 (CloudEvent schema registry)** — `platform.approval.*` schemas B19-core emits
  against. **B16** (not a CSV upstream of B19 but a content dependency) — supplies the
  `approval.elevation` Rego B19-core consults.
- **B5** — CSV upstream for the **UI** portion only; B19-ui co-lands with B5 in W4. B19-core (W3) does
  not block on B5.
### 3.2 Downstream pieces blocked on this
- **A23 (Kargo, W4)** — uses the `Approval` CRD + Argo workflow as its Stage human gate.
### 3.3 Continuous (non-blocking) inputs
- **B22 (threat model)** — approval-bypass / privilege-escalation standards; security review before
  complete (ADR 0027). **B14 (test framework)** — Chainsaw/PyTest harness.

## 4. Parallelizable Subtasks
- After TASK-02, **TASK-03 (elevation)**, **TASK-05 (decision/emit)**, and **TASK-06 (timeout)** fan
  out concurrently.
- **TASK-07 (dry-run)** parallels TASK-04/05 once TASK-03 lands.
- **TASK-08 (Kargo contract)** parallels TASK-06/07 once TASK-05 lands.

## 5. Test Strategy
- **Chainsaw (operator/CRD):** AC-B19-01 (`Approval` admits), AC-B19-03 (Argo suspend/resume/timeout),
  AC-B19-04 (enqueue filter), AC-B19-10 (CRD conversion).
- **PyTest (logic):** AC-B19-02 (elevate-only, never lower — across cases), AC-B19-05/06 (decision +
  timeout emission/audit), AC-B19-07 (dry-run no-side-effect), AC-B19-08 (Kargo contract with an A23
  fake), AC-B19-09 (no M-of-N/escalation path), **AC-B19-11 (UI-less e2e — the split guard)**.
- **Playwright:** **deferred to B19-ui (W4)** — no UI in core.
- **Fixtures/fakes:** B16 `approval.elevation` Rego stubbed until B16 lands; B12 `platform.approval.*`
  schemas faked; A18 audit adapter faked; A23 Kargo gate faked for the contract test; A20 simulator
  stub for dry-run.

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/3` (contains W3 siblings + the W1/W2 specs it binds to).
### 6.2 PR — `piece/B19-core-approval-system` → base `wave/3`; carries SPEC-B19 + PLAN-B19 + CRD + Argo
template + controller. **B19-ui** is a separate W4 PR co-landing with B5 (`piece/B19-ui-approval-queue`
→ `wave/4`).
### 6.3 Merge order — B19-core merges in W3 before A23 (W4). B19-ui + B5 co-land in W4. The UI-less e2e
(AC-B19-11) must pass before the W3 rollup to main.

## 7. Effort Estimate
- TASK-01 S, TASK-02 L, TASK-03 M, TASK-04 M, TASK-05 M, TASK-06 S, TASK-07 S, TASK-08 M, TASK-09 M,
  TASK-10 M.
- **Rollup: L** (matches CSV; UI deferred to B19-ui/B5 keeps core at L). **Critical path:** TASK-01 →
  TASK-02 → TASK-05 → TASK-08 → TASK-09 → TASK-10.

## 8. Rollback / Reversibility
Back out by reverting the `Approval` CRD + controller + Argo template. **Downstream breakage if
reverted:** A23 (Kargo) loses its human gate; Coach (B10) and HolmesGPT (A14) lose their approval
path ("upon-approval" state, ADR 0012). In-flight `Approval`s suspend indefinitely on revert — drain
or force-decide before removing. The W3/W4 split means B19-ui can be reverted independently of
B19-core without breaking A23. CRD shape changes are breaking (ADR 0030) — use the conversion path.
