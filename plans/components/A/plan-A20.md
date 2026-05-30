# PLAN A20 — Policy simulator service

> spec: SPEC-A20 · kind: COMPONENT · tier: T1
> wave: W1 · estimate: L
> upstream-pieces: [A7, A1, A6] · downstream-pieces: [A22]

## 1. Implementation Strategy

Build A20 as a thin stateless aggregator that fans a synthetic request out to each enforcement layer's structured dry-run and composes the per-layer verdicts (with RBAC as its own attributed layer) into one trace bound to B3's decision-document shape. Expose exactly one aggregator API consumed by two surfaces — the Headlamp panel and the HolmesGPT skill — so a HolmesGPT explanation is reproducible in Headlamp. The hard reality is that each layer's dry-run is a per-component ADR 0038 deliverable A20 does not own; so build aggregator-and-contract-first against fakes for every layer, surface unavailable layers honestly (never fabricate a verdict), and swap fakes for live dry-run paths as B2/A6/B19/A7/B9 land them. The two invariants under test throughout: every verdict carries `simulated: true` with zero enforcement-path side effect, and runs are audited under `platform.policy.*` via A18 without entering enforcement.

## 2. Ordered Task List

- **TASK-01:** Aggregator core + the per-layer fan-out/compose contract; bind the decision-trace shape to B3's decision document (`simulated` + cited reasons) — produces: aggregator service skeleton — depends-on: [] (B3 contract).
- **TASK-02:** Per-layer dry-run clients: RBAC (separate layer), Gatekeeper audit-mode, LiteLLM OPA callback, Envoy egress ext-authz, approval elevation (no `Approval` created), `agent-platform` CLI promotion gate — produces: 6 layer adapters — depends-on: [TASK-01]. (Fakes until each lands.)
- **TASK-03:** Unavailable-layer handling (surface as unavailable, never fabricate) + statelessness/idempotence guarantees — produces: degraded-mode behavior — depends-on: [TASK-02].
- **TASK-04:** Aggregator HTTP API (`/v1/...`) — the single API both consumers call — produces: versioned API — depends-on: [TASK-01].
- **TASK-05:** Audit emission of runs under `platform.policy.*` via the A18 adapter; assert no enforcement-path effect — produces: audit wiring — depends-on: [TASK-04]. (Mock A18 until landed.)
- **TASK-06:** OPA gating of the simulator itself (who may simulate which subject's claims) + admission Rego for the A20 service — produces: A20 access Rego — depends-on: [TASK-04].
- **TASK-07:** Headlamp simulator panel (compose/paste request → layer-by-layer trace) — produces: panel plugin — depends-on: [TASK-04] (A9/A22 framework).
- **TASK-08:** HolmesGPT skill wrapping the aggregator API (`Skill` artifact + CapabilitySet reference) — produces: HolmesGPT skill — depends-on: [TASK-04] (B13 `Skill` CRD).
- **TASK-09:** Cross-cutting — `GrafanaDashboard` XR (sim volume, per-layer availability, deny-reason distribution), alerts, runbook, per-product docs — depends-on: [TASK-05, TASK-06].
- **TASK-10:** 3-layer tests mapping all ACs — depends-on: [TASK-07, TASK-08, TASK-05].
- **TASK-11:** Tutorials & how-tos ("simulate before editing a policy"; "ask HolmesGPT why a request was denied") — depends-on: [TASK-09].

## 3. Dependency Map

### 3.1 Upstream that must ship first (HARD)
- **A7 (OPA/Gatekeeper)** — OPA engine + Gatekeeper audit-mode dry-run.
- **A1 (LiteLLM)** — LiteLLM OPA-callback dry-run path.
- **A6 (agent-sandbox + Envoy egress)** — Envoy egress ext-authz dry-run path.

### 3.2 Downstream blocked on this
- **A22** — every CRD editor integrates the simulator for preview-before-commit (CSV downstream).

### 3.3 Continuous (non-blocking) inputs
- **B3** — canonical OPA decision contract + `simulated` flag the trace binds to (`[PROPOSED]` build-order; not in CSV upstream). **B19-core** approval elevation dry-run. **B9** CLI promotion-gate dry-run. **A18** audit adapter. **A14** HolmesGPT (hosts the skill). **B13** `Skill` CRD. **B14** test framework. **B22** threat model (simulator targets OPA-bypass/capability-escape patterns).

## 4. Parallelizable Subtasks
- After TASK-01: the six per-layer dry-run clients (TASK-02) are an independent fan-out, one per layer, each developed against its own fake.
- After TASK-04: TASK-05 (audit), TASK-06 (gating), TASK-07 (panel), TASK-08 (skill) are independent.
- TASK-09 (cross-cutting) + TASK-11 (docs) parallelize once panel + skill exist.

## 5. Test Strategy
- **PyTest (logic):** AC-A20-01 (per-layer verdicts + cited rule), -02 (all six layers present), -03 (RBAC-attributed, not OPA), -04 (approval level resolved, no `Approval` created), -05 (`simulated:true` + no state diff), -06 (panel/skill identical traces), -09 (unavailable-layer surfacing), -10 (idempotent/stateless), -11 (point-query-only), -12 (cross-tenant simulate denied).
- **Playwright (UI/e2e):** AC-A20-07 (Headlamp panel renders layer-by-layer trace).
- **Chainsaw (operator/CRD):** AC-A20-08 (run emits `platform.policy.*` audit, no enforcement), admission of the A20 service; cross-layer dry-run wiring against fakes for not-yet-landed layers.
- **Fakes/fixtures:** a fake dry-run server per layer (RBAC/Gatekeeper/LiteLLM/Envoy/approval/CLI), each replaced by the live path as B2/A6/B19/A7/B9 land; mock A18 adapter; mock B13 `Skill`. No-side-effect tests assert state diffs before/after each simulation.

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/1` (contains A7, A1, A6 specs A20 builds on).
### 6.2 PR — `piece/A20-policy-simulator` → base `wave/1`; carries SPEC-A20 + PLAN-A20.
### 6.3 Merge order — independent of W1 siblings; wave/1 rolls up to main. A22 (wave/2) bases on the merged A20 aggregator API.

## 7. Effort Estimate
- Aggregator + per-layer clients + degraded mode (TASK-01..04): M (the per-layer fan-out against shifting dry-run contracts is the bulk).
- Audit + gating + panel + skill (TASK-05..08): M (panel + skill are two thin consumers of one API).
- Cross-cutting + tests + docs (TASK-09..11): M.
- **Rollup: L** (matches CSV). **Critical path:** TASK-01 (bind to B3 contract) → 02 (per-layer dry-runs) → 04 (one API) → 07/08 (two consumers) → 10.

## 8. Rollback / Reversibility
Back out by unregistering the Headlamp panel, withdrawing the HolmesGPT skill (remove from CapabilitySet), and deleting the aggregator Deployment. A20 is **stateless and non-enforcing**, so rollback is fully non-destructive — no policy, no enforcement decision, and no live request is affected (the simulator never touched the enforcement path by design). **Downstream break:** A22's editors lose preview-before-commit (they fall back to submit-without-simulation or a degraded banner per A22's mock-out plan), and HolmesGPT loses the "why was this denied" skill — diagnostic convenience only, no security regression since enforcement was never in A20's hands.
