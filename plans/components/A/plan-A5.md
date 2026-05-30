# PLAN A5 — ARK agent operator

> spec: SPEC-A5 · kind: COMPONENT · tier: T0
> wave: W1 · estimate: L
> upstream-pieces: [A6] · downstream-pieces: [B16, B7, B9, B10, B17, B18, B20]

## 1. Implementation Strategy

Install ARK unmodified at a pinned, tested upstream version via Helm (config + wrap, not fork — ADR 0001/§9), then layer the platform's standard Workstream A deliverables around it: Gatekeeper admission for the seven CRDs, audit-adapter linkage, OTel/trace config, the cross-tenant Headlamp plugin, the HolmesGPT ARK toolset, and the trigger-flow wiring for `AgentRun` creation. Build bootstrap-tolerantly so ARK is useful in W1 before A18 audit endpoint and the full OPA decision points are wired, then rewire as those land. The critical path is: pin ARK → CRDs admit/reconcile → Sandbox handoff to A6 → AgentRun execution → audit/OTel/OPA wiring → plugin + toolset + tests.

## 2. Ordered Task List

- TASK-01: Select + pin upstream ARK version; author Helm values/manifests in Git — produces: ArgoCD-syncable ARK install — depends-on: []
- TASK-02: Verify registration of the seven CRDs (`Agent`, `AgentRun`, `Team`, `Tool`, `Memory`, `Evaluation`, `Query`), all namespaced — produces: CRD inventory conformance — depends-on: [TASK-01]
- TASK-03: Wire `Agent` reconcile → `sandboxTemplateRef` → A6 Sandbox request (fake/stub A6 until ready) — produces: agent-pod placement path — depends-on: [TASK-02]
- TASK-04: Validate `Agent` binding resolution (`capabilitySetRefs[]`, `memoryRefs[]`→`Memory`→`MemoryStore`, `modelRef`, `sdk`, `triggers`, `exposes`) — produces: binding-resolution behavior — depends-on: [TASK-03]
- TASK-05: Implement `AgentRun` reconcile from B8-adapter and Argo-step creators (fakes for B8/A3) — produces: run execution + state surfacing — depends-on: [TASK-04]
- TASK-06: Link audit adapter library; emit `platform.audit.*` for all seven CRD lifecycle actions; add buffering fallback when endpoint absent — produces: audit emission — depends-on: [TASK-02]
- TASK-07: Configure OTel emission; propagate `AgentRun.traceId` for Tempo↔Langfuse correlation — produces: traces/metrics — depends-on: [TASK-05]
- TASK-08: Author Gatekeeper admission policies for the seven CRDs (incl. `sdk` value-set check); contribute targets to B16 — produces: admission gating — depends-on: [TASK-02]
- TASK-09: Design + document the `platform.lifecycle.*` emit set and `AgentRun`-creation Trigger contract (AlertManager→HolmesGPT via B8) — produces: trigger-flow doc — depends-on: [TASK-05]
- TASK-10: Build ARK Headlamp plugin (cross-tenant agent visibility) on A9/A22 framework — produces: Headlamp plugin — depends-on: [TASK-02]
- TASK-11: Build ARK HolmesGPT toolset (agent/run lifecycle + ARK-state inspection) — produces: toolset — depends-on: [TASK-05]
- TASK-12: Define CRD versioning lifecycle (conversion-webhook + deprecation-window procedure) per ADR 0030 — produces: versioning runbook + webhook scaffold — depends-on: [TASK-02]
- TASK-13: Author 3-layer tests, docs, runbook, alerts, `GrafanaDashboard` XR — produces: full deliverable set — depends-on: [TASK-06, TASK-07, TASK-08, TASK-10, TASK-11]

## 3. Dependency Map

### 3.1 Upstream pieces that must ship first (HARD)
- **A6** — Sandbox/SandboxTemplate reconciliation + warm pool; A5 hands placement to A6 via `sandboxTemplateRef`. Stubbed until A6 lands.

### 3.2 Downstream pieces blocked on this
- B16 (OPA content for ARK CRDs), B7 (SDK keyed to `Agent.sdk`), B9 (CLI), B10 (Coach), B17/B18 (profiles/compositions as `Agent` CRDs), B20 (PV mapping).

### 3.3 Continuous (non-blocking) inputs
- B14 test framework, B22 threat-model standards (fan-out edges). A18 audit adapter, A13/A2 observability — consumed as they land; bootstrap-tolerant.

## 4. Parallelizable Subtasks

- After TASK-02: TASK-06 (audit), TASK-08 (OPA admission), TASK-10 (Headlamp plugin), TASK-12 (versioning) run concurrently.
- After TASK-05: TASK-07 (OTel), TASK-09 (trigger flow), TASK-11 (toolset) run concurrently.
- TASK-13 fans in from the above groups.

## 5. Test Strategy

| AC | Layer | Notes / fixtures |
|---|---|---|
| AC-A5-01/02/03/04/05/06/10/13 | Chainsaw | CRD apply/reconcile/admission/versioning; fake A6 Sandbox controller + fake B8/A3 `AgentRun` creators |
| AC-A5-11 | Playwright | Headlamp plugin cross-tenant visibility (admin vs non-admin) |
| AC-A5-07/08/09/12/14/15 | PyTest | audit-adapter emission (fake endpoint), CloudEvent namespace/version assertions, OTel trace join, HolmesGPT toolset logic, trigger-flow doc lint, dashboard render |

Fakes needed for not-yet-landed upstreams: A6 Sandbox controller stub, B8 adapter + Argo step `AgentRun` creators, A18 audit endpoint sink, A7/B16 OPA bundle.

## 6. PR / Branch Mapping

### 6.1 Stack position — base branch = `wave/1` (contains A6 spec from W0 rolled up)
### 6.2 PR — `piece/A5-ark-agent-operator` → base `wave/1`; carries SPEC-A5 + PLAN-A5
### 6.3 Merge order — independent of W1 siblings (A10, A18, B13, B4); `wave/1` rolls up to main after the band's specs land.

## 7. Effort Estimate

- S: TASK-01, TASK-02, TASK-09, TASK-12. M: TASK-03, TASK-04, TASK-06, TASK-07, TASK-08, TASK-10, TASK-11. L: TASK-05, TASK-13.
- Rollup: **L** (matches CSV estimate).
- Critical path: TASK-01 → 02 → 03 → 04 → 05 → 13.

## 8. Rollback / Reversibility

Back out by reverting the ArgoCD-synced ARK release; the seven CRDs and their resources are removed (or pinned to prior version via conversion webhook for a soft rollback). Downstream that breaks: all `Agent`/`AgentRun`-driven execution (B7/B9/B10/B17/B18), and any agent placement (loss of A6 handoff). Because ARK holds no system-of-record state (agent state in Postgres via A10/A18), rollback loses no durable data; CRDs re-sync from Git on reinstall.
