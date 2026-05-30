# PLAN A22 — Headlamp graphical-editor framework for platform CRDs

> spec: SPEC-A22 · kind: COMPONENT · tier: T0
> wave: W2 · estimate: L
> upstream-pieces: [A9, A20] · downstream-pieces: [A21]

## 1. Implementation Strategy

Build the editor framework as a Headlamp plugin layer on top of A9's base install and shared widgets, then ship the eleven initial-set editors on that framework. Develop framework-first so A21 (and per-component teams) have a stable widget/form-generation library to bind to; mock the A20 simulator aggregator and the Git/PR helper until the real surfaces land, then swap mocks for the live contracts (per §10 mock-out commitment). Critical path is: framework form-generation + diff engine → simulator-panel integration → overlay-resolution view (the highest-risk editor capability) → the full initial editor set → claim-scoped gating + audit emission. Everything is PR-only by construction; there is never a cluster-write code path.

## 2. Ordered Task List

- **TASK-01:** Pin Headlamp version; scaffold the A22 plugin package against the A9 framework + auth handoff — produces: plugin skeleton + Keycloak-claim access — depends-on: [] (A9 available).
- **TASK-02:** Schema-driven form generator: read served OpenAPI schema → render fields (enums→dropdowns, regex/FQDN validation, list add/patch controls) — produces: form-generation widget library — depends-on: [TASK-01].
- **TASK-03:** Reference-picker widget, claim-scoped multi-select over in-scope resources of a referenced kind — produces: reference-picker widget — depends-on: [TASK-02].
- **TASK-04:** Diff-against-Git engine: read desired state from ArgoCD Git source, render YAML diff, surface list removals explicitly — produces: diff viewer — depends-on: [TASK-02].
- **TASK-05:** Git/PR submission flow (GitHub, ADR 0033); guarantee no cluster-write path — produces: PR submission module — depends-on: [TASK-04]. (Mock Git host until wired.)
- **TASK-06:** Simulator-panel integration against A20 aggregator API; degraded-simulator banner + submit-with-warning — produces: simulator panel — depends-on: [TASK-02]. (Mock A20 until it lands.)
- **TASK-07:** Overlay-resolution view: consume ADR 0032 resolver output, render effective resolved capability set for `CapabilitySet`/`Agent`/templates — produces: effective-set view — depends-on: [TASK-02, TASK-03]. (Mock resolver until B13/ADR 0032 surface exists.)
- **TASK-08:** Core capability-CRD editors — `MCPServer`, `A2APeer`, `RAGStore`, `EgressTarget`, `Skill` — produces: 5 editors — depends-on: [TASK-02, TASK-04, TASK-06].
- **TASK-09:** Overlay-aware editors — `CapabilitySet`, `Agent`, Agent templates (with `sdk` enum `langgraph`/`deep-agents`) — produces: 3 editors — depends-on: [TASK-07].
- **TASK-10:** XRD/config editors — `TenantOnboarding` (baseline; A21 extends), `AuditLog`, `LogLevel`, `GrafanaDashboard` — produces: 4 editors — depends-on: [TASK-08].
- **TASK-11:** OPA-bundle editor as a Git-tracked policy artifact (validation + diff + simulator + PR) — produces: bundle editor — depends-on: [TASK-05, TASK-06].
- **TASK-12:** Claim-based plugin gating (REQ-A22-14) + audit emission for PR-open / simulation-run (REQ-A22-15) — produces: gating + audit wiring — depends-on: [TASK-05, TASK-06].
- **TASK-13:** Cross-cutting deliverables — alerts, `GrafanaDashboard` XR, runbook, HolmesGPT toolset, per-product docs — depends-on: [TASK-10, TASK-11, TASK-12].
- **TASK-14:** 3-layer tests (Playwright e2e, PyTest helpers, Chainsaw apply-output) mapping all ACs — depends-on: [TASK-09, TASK-10, TASK-11, TASK-12].
- **TASK-15:** Tutorials & how-tos ("author a CRD via the graphical editor"; extend "write a Headlamp plugin") — depends-on: [TASK-13].

## 3. Dependency Map

### 3.1 Upstream that must ship first (HARD)
- **A9** — Headlamp base install, branding, shared libraries, auth handoff (Keycloak claims). Consumed: plugin-loading mechanism + claim access + shared widgets.
- **A20** — policy-simulator aggregator API. Consumed: per-layer decision trace for the simulator panel. (Mockable until landed.)

### 3.2 Downstream blocked on this
- **A21** — builds its `TenantOnboarding` editor on this framework.
- Per-component editor teams (any A/B component shipping a CRD editor).

### 3.3 Continuous (non-blocking) inputs
- **B14** test framework (Chainsaw/Playwright/PyTest harness). **B22** security threat model (editor authoring is a privileged surface). **B13/ADR 0032** overlay resolver output (mock until exposed). **B3/B16** OPA Rego for who-may-edit gating.

## 4. Parallelizable Subtasks

- After TASK-02: TASK-03, TASK-04, TASK-06 are independent fan-out (picker / diff / simulator).
- After the framework core (TASK-02..07): TASK-08, TASK-09, TASK-10, TASK-11 are largely independent per-editor groups (one per editor cluster), parallelizable across editor authors.
- TASK-13 (cross-cutting) and TASK-15 (docs) parallelize once their editors exist.

## 5. Test Strategy

- **Playwright (UI/e2e):** AC-A22-01..07, 09, 11, 13, 14, 16 — form rendering, picker scoping, diff-against-Git, list-patch, schema block, simulator panel + degraded banner, overlay effective set, all editors loadable, gating, `sdk` enum.
- **Chainsaw (operator/CRD):** AC-A22-08 (no cluster write — API-server audit assertion), AC-A22-12 (rebuild against bumped CRD version applies cleanly), AC-A22-10 (bundle PR not direct-write).
- **PyTest (logic):** AC-A22-15 (audit/policy event emission), helper logic for diff computation and claim-scoping filters; AC-A22-13 cross-component library-consumption check.
- **Fakes/fixtures:** mock A20 aggregator (OQ-A22-3), mock ADR 0032 resolver (OQ-A22-4), Git/cluster divergence fixture (AC-A22-03), mock GitHub PR API. Replace each with the real surface when it lands and re-run (§10 mock-out re-run).

## 6. PR / Branch Mapping

### 6.1 Stack position — base branch = `wave/2` (contains A9, A20 specs A22 builds on).
### 6.2 PR — `piece/A22-headlamp-editor-framework` → base `wave/2`; carries SPEC-A22 + PLAN-A22.
### 6.3 Merge order — independent of W2 siblings; wave/2 rolls up to main. A21 (wave/3) bases on the merged A22.

## 7. Effort Estimate

- Framework core (TASK-01..07): L (form-gen + diff + simulator + overlay view are the bulk; overlay view is the single hardest task).
- Editor set (TASK-08..11): M (parallelizable per editor).
- Cross-cutting + tests + docs (TASK-12..15): M.
- **Rollup: L** (matches CSV). **Critical path:** TASK-01 → 02 → 07 (overlay) → 09 → 14.

## 8. Rollback / Reversibility

Back out by unregistering the A22 plugins from Headlamp; the platform reverts to raw-YAML / read-only views (the pre-editor baseline). No cluster state depends on the plugin (it only opens PRs), so rollback is non-destructive and leaves any already-merged PRs intact. **Downstream break:** A21's `TenantOnboarding` editor and any per-component editor built on the framework stop loading — those components must vendor a framework version or revert their editors. Open PRs already authored remain valid (they are plain Git PRs).
