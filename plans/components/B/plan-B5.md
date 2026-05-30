# PLAN B5 — Cross-cutting Headlamp plugins

> spec: SPEC-B5 · kind: COMPONENT · tier: T1
> wave: W4 · estimate: L
> upstream-pieces: [A23] · downstream-pieces: [B19]

## 1. Implementation Strategy

Build four cross-cutting Headlamp plugins as custom TypeScript on the A9 framework and A22 editor widgets: capability inspector (read views over the eight capability CRDs + LiteLLM registry/health), approval-queue UI (the B19-ui half of the split B19), virtual-key admin (self-serve OPA-gated `VirtualKey` issuance + force-register override), and the Kargo plugin (Stage/Warehouse/promotion status + triggering). Every plugin gates on §6.9 claims in plugin code (§9), routes actions through OPA (RBAC-floor/OPA-restrictor), and audits via the A18 adapter. Per the waves.md cycle resolution, B5/B19-ui co-land with the Kargo plugin in W4 after B19-core (W3) and A23 (W4); build against mock B19-core/Kargo contracts first, integrate when they land. B5 must not re-implement B19-core or the A22 framework — it consumes both.

## 2. Ordered Task List

- **TASK-01:** Plugin scaffold on A9 framework + §6.9 claim-gating helper — produces: plugin base + gating lib — depends-on: [].
- **TASK-02:** Capability inspector (read views + LiteLLM registry/health) — produces: inspector plugin — depends-on: [TASK-01].
- **TASK-03:** Virtual-key admin (issuance flow, OPA gate, secure delivery, force-register override) — produces: vkey plugin — depends-on: [TASK-01].
- **TASK-04:** Approval-queue UI / B19-ui (queue by resolved level, evidence display, approve/reject → resume Argo Workflow) — produces: approval plugin — depends-on: [TASK-01].
- **TASK-05:** Kargo plugin (Stage/Warehouse/promotion viz + trigger + per-Stage deep-links) — produces: Kargo plugin — depends-on: [TASK-01].
- **TASK-06:** A22 editor-widget reuse for edit-capable views + read-only handling for non-A22 CRDs — produces: editor integration — depends-on: [TASK-02, TASK-03].
- **TASK-07:** Audit emission of plugin actions via A18 adapter — produces: action audit — depends-on: [TASK-03, TASK-04, TASK-05].
- **TASK-08:** Integrate against real B19-core + A23 (replace mocks) — produces: end-to-end wiring — depends-on: [TASK-04, TASK-05].
- **TASK-09:** Helm/packaging into Headlamp, manifests in envs/ — produces: deploy — depends-on: [TASK-02..07].
- **TASK-10:** 3-layer tests, dashboard XR, runbook, how-tos — produces: cross-cutting deliverables — depends-on: [TASK-09].

## 3. Dependency Map

### 3.1 Upstream that must ship first (HARD)
- A23 (Kargo) — API/CRDs the Kargo plugin drives (piece-index A23→B5).
- A9 (Headlamp framework) + A22 (editor framework) — plugin host + editor widgets (implicit, required).
- B19-core (W3) — `Approval` CRD shape + resolved-level + Argo-resume contract for the queue UI.
- B13 — reconciled capability-registry state the inspector reads. A18 — audit adapter.

### 3.2 Downstream blocked on this
- B19 — the approval system needs B5's approval-queue UI (B19-ui) to be usable (piece-index B5→B19; the split makes this co-land, not block).

### 3.3 Continuous (non-blocking) inputs
- A20 (policy simulator, deep-linked for preview), B14 test framework, B22 threat model.

## 4. Parallelizable Subtasks
- After TASK-01: TASK-02, TASK-03, TASK-04, TASK-05 (the four plugins) run concurrently.
- TASK-06 fans from inspector/vkey; TASK-07 fans across action-emitting plugins.
- TASK-08 (real integration) gates on B19-core/A23 landing; until then plugins develop against mocks in parallel.
- Docs/dashboard/tests (TASK-10) parallel once plugins stabilize.

## 5. Test Strategy
- **Playwright (primary):** AC-B5-01..09, -11 (inspector listing, approval flow, vkey issuance, Kargo viz/trigger, claim-gating, OPA-deny block, editor reuse, read-only CRDs, force-register).
- **Chainsaw:** AC-B5-02, -07, -11 (create `Approval` → appears in queue → approve resumes workflow; action audit events on broker; force-register reflected in cluster).
- **PyTest:** AC-B5-06, -10, -12 (resolved-level filtering helper, no-B19-core-code assertion, version-skew handling).
- **Fakes:** mock B19-core (Approval CRD + stub Argo-resume), mock Kargo API, fake LiteLLM registry/health feed, fake OPA gate (allow/deny + `simulated:true`), fake audit endpoint. Swap for real B19-core/A23/B13/A18 at TASK-08.

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/4` (contains A23 + B19-core + W0–W3 specs).
### 6.2 PR — `piece/B5-headlamp-cross-cutting-plugins` → base `wave/4`; carries spec-B5 + plan-B5. B19-ui co-lands here (not a separate B19 PR).
### 6.3 Merge order — co-land with A23's Kargo wiring; within W4 independent of B15/B18/B21; wave/4 rolls up to main.

## 7. Effort Estimate
- TASK-01 M, 02 M, 03 L, 04 L, 05 M, 06 M, 07 S, 08 M, 09 S, 10 M. Rollup **L**.
- Critical path: TASK-01 → 04 (approval-queue UI) → 08 (B19-core integration) → 10, with TASK-05 (Kargo) on a parallel near-critical leg gated by A23.

## 8. Rollback / Reversibility
Plugins are a versioned bundle loaded into Headlamp; rollback = remove/pin the prior bundle. No in-cluster reconcile state owned. Reverting the approval-queue UI breaks the human-decision surface of the approval system (B19) — approvals can still be created/decided via raw kubectl but lose the workqueue UX; reverting the Kargo plugin removes Headlamp-driven promotion (Kargo UI deep-links still work). No data migration. Downstream B19 usability degrades to CLI/raw until restored.
