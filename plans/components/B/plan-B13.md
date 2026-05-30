# PLAN B13 — Custom Python kopf operator for LiteLLM

> spec: SPEC-B13 · kind: COMPONENT · tier: T0
> wave: W1 · estimate: L
> upstream-pieces: [A1] · downstream-pieces: [A17, B16, B17]

## 1. Implementation Strategy
B13 is a Python kopf operator packaged as a subchart of the LiteLLM Helm chart that reconciles eight
namespaced CRDs into LiteLLM's admin API, OPA data, and the Envoy egress allowlist. The approach is
CRD-contract-first: define the eight CRDs with their Canon field sets and freeze the `status`
subresource (Headlamp and ops bind to it), then build reconcilers per CRD, then the CapabilitySet
layering resolver implementing exactly the §6.8 committed model (declared-order, field-level
add/replace, lists restated, overrides last) with sub-semantics gated behind ADR 0032. Capability
mutations emit `platform.capability.changed`; reconcile/audit events flow through the A18 adapter.
Secrets resolve via ESO. The operator exposes a `/v1/...` admin API and owns its CRD versioning
lifecycle. The kopf/Crossplane split is respected — B13 touches no cloud-shaped XR. Because it ships
as a LiteLLM subchart, one ArgoCD release deploys gateway + operator with automatic version pinning.

## 2. Ordered Task List
- **TASK-01:** Define the eight CRDs (`MCPServer`, `A2APeer`, `RAGStore`, `EgressTarget`, `Skill`,
  `CapabilitySet`, `VirtualKey`, `BudgetPolicy`) with Canon fields + a frozen `status` subresource —
  produces: CRD manifests — depends-on: [].
- **TASK-02:** Scaffold the kopf operator + package it as a **LiteLLM Helm subchart** (one ArgoCD
  release, version pin) — produces: operator skeleton + subchart — depends-on: [TASK-01].
- **TASK-03:** Reconcilers for the **capability CRDs** (`MCPServer`/`A2APeer`/`RAGStore`/`Skill`) into
  LiteLLM admin API, with **ESO/Secret** credential resolution — produces: capability reconcilers —
  depends-on: [TASK-02].
- **TASK-04:** Reconcile **`EgressTarget`** into the **Envoy egress allowlist** (ADR 0003) — produces:
  egress reconciler — depends-on: [TASK-02].
- **TASK-05:** Implement the **CapabilitySet layering resolver** (§6.8 committed model; sub-semantics
  gated behind ADR 0032) + the **resolved per-Agent view** output — produces: layering resolver —
  depends-on: [TASK-03].
- **TASK-06:** Reconcile **`VirtualKey`** (bind capabilitySet/identity/budget/env/models/ttl) and
  **`BudgetPolicy`** (→ LiteLLM enforcement + OPA data) — produces: key/budget reconcilers —
  depends-on: [TASK-03, TASK-05].
- **TASK-07:** Emit **`platform.capability.changed`** on capability CRD mutations + **`platform.audit.*`**
  reconcile events via the **A18 adapter** — produces: event emission — depends-on: [TASK-03, TASK-06].
- **TASK-08:** **`/v1/...` admin API** + CRD versioning lifecycle (conversion-webhook seam, deprecation
  window) — produces: admin API + versioning — depends-on: [TASK-06].
- **TASK-09:** Cross-cutting deliverables: HolmesGPT toolset, `GrafanaDashboard` XR, alerts, runbook,
  per-product docs, how-tos — produces: deliverable set — depends-on: [TASK-07, TASK-08].

## 3. Dependency Map
### 3.1 Upstream pieces that must ship first (HARD)
- **A1 (LiteLLM)** — the admin API B13 reconciles into; B13 ships as its Helm subchart.
- (Interface bindings) **A6** Envoy egress (allowlist target), **A7** OPA (data target), **A18** audit
  adapter, **B12** `platform.capability.changed` schema.

### 3.2 Downstream pieces blocked on this
- **A17** (MCP services as `MCPServer` CRDs), **B16** (OPA content over B13's data + CRDs), **B17**
  (profiles as `CapabilitySet`s). Indirectly B6/B7 (consume `platform.capability.changed`).

### 3.3 Continuous (non-blocking) inputs
- **B3** OPA framework (`data`-namespacing convention to align OPA-data paths), **B14** test framework,
  **B22** threat model (capability-escape controls).

## 4. Parallelizable Subtasks
After TASK-02: TASK-03 and TASK-04 run concurrently. TASK-05 follows TASK-03; TASK-06 follows
TASK-03+TASK-05. TASK-07 and TASK-08 run concurrently after TASK-06. TASK-09 is the serial tail.

## 5. Test Strategy
- **Chainsaw:** apply each CRD → expected LiteLLM/OPA/Envoy state (AC-B13-01); the two-set+overrides
  layering matches §6.8 pseudocode (AC-B13-02); capability mutation emits `platform.capability.changed`
  (AC-B13-03); `VirtualKey`/`BudgetPolicy`/`EgressTarget` reconcile (AC-B13-04..06); non-allowlisted
  FQDN blocked (AC-B13-06); new-`vN`-group conversion webhook (AC-B13-09); subchart single-release
  install (AC-B13-11); cross-namespace default-deny (AC-B13-13).
- **PyTest:** layering resolution logic, ESO secret-wiring (no plaintext, AC-B13-07), OPA-data write,
  audit-via-adapter + no-direct-write (AC-B13-10), Crossplane-XR exclusion (AC-B13-12), resolved-view
  builder (AC-B13-14), `/v1` admin-API routing + deprecation (AC-B13-08).
- **Playwright:** N/A here — virtual-key self-serve UI is Headlamp/B5.
- **Fixtures/fakes:** fake LiteLLM admin API, fake OPA data store, fake Envoy allowlist sink, ESO/Secret
  fixtures, a **B12 `platform.capability.changed` fixture**, fake audit endpoint (A18).

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/1` (contains A1 spec)
### 6.2 PR — `piece/B13-kopf-litellm-operator` → base `wave/1`; carries spec-B13 + plan-B13
### 6.3 Merge order — independent of W1 siblings (B1/B2/B3/B4); wave/1 rolls up to main; A17/B16/B17
(W2) branch after B13 merges.

## 7. Effort Estimate
TASK-01 M · TASK-02 M · TASK-03 L · TASK-04 S · TASK-05 L · TASK-06 M · TASK-07 M · TASK-08 M ·
TASK-09 M. Rollup: **L**. Critical path: TASK-01 → TASK-02 → TASK-03 → TASK-05 → TASK-06 → TASK-07/08
→ TASK-09.

## 8. Rollback / Reversibility
Back out by removing the operator subchart from the LiteLLM release (ArgoCD) and pruning the eight
CRDs; LiteLLM keeps its last applied registry state but stops being CRD-reconciled. Downstream A17/
B16/B17 lose their reconciliation target. CRD removal must precede operator removal to avoid orphaned
finalizers. Because the kopf/Crossplane split is respected, reverting B13 does not touch any
Crossplane/B4 XR. Capability-change consumers (B6/B7) degrade gracefully — enforcement stays at
LiteLLM/OPA/Envoy regardless.
