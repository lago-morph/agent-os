# PLAN ADR-0006 — Python kopf operator for LiteLLM reconciliation [PROPOSED]

> spec: SPEC-ADR-0006 · kind: ADR · tier: T0
> wave: authoring-parallel · estimate: S
> upstream-pieces: [A1] · downstream-pieces: [A17;B16;B17]

## 1. Implementation Strategy
Enforcement/verification map. ADR 0006 is enforced by component B13 (the kopf operator that reconciles the eight LiteLLM-facing CRDs and is subcharted with LiteLLM), with the controller-framework invariant policed across the whole repo and the Crossplane boundary held by B4. Conformance is proven by: (a) per-CRD reconcile tests against the LiteLLM admin API; (b) a controller-framework scan (Crossplane-or-kopf only); (c) a subchart/version-pin check; (d) a `/v1/...` API-versioning gate. No new product build belongs to this ADR — it is a constraint map over B13/A1/B4.

## 2. Ordered Task List
- TASK-01: Map each REQ to the enforcing component piece — produces: enforcement matrix — depends-on: []
- TASK-02: Specify per-CRD reconcile→LiteLLM-admin-API conformance for all eight CRDs — produces: Chainsaw suite (B13) — depends-on: [TASK-01]
- TASK-03: Specify the controller-framework invariant scan (no bespoke frameworks; B13 owns no XR) — produces: CI scan (B13/B4) — depends-on: [TASK-01]
- TASK-04: Specify the subchart packaging + gateway↔operator version-pin check — produces: CI/Helm check (B13/A1) — depends-on: [TASK-01]
- TASK-05: Specify `/v1/...` admin-API versioning gate + capability/audit event emission check — produces: CI gate + e2e (B13/B12/A18) — depends-on: [TASK-01]

## 3. Dependency Map
### 3.1 Upstream that must ship first (HARD)
- A1 — LiteLLM admin API the operator reconciles into.
### 3.2 Downstream blocked on this
- A17, B16, B17.
### 3.3 Continuous (non-blocking) inputs
- B4 Crossplane boundary; A7 OPA gating; A18 audit; B12 capability events; B14 test framework; B15 CI.

## 4. Parallelizable Subtasks
TASK-02, TASK-03, TASK-04, TASK-05 fan out independently after TASK-01.

## 5. Test Strategy
- AC-01/02/07 → Chainsaw (per-CRD reconcile; no-XR-ownership; no Crossplane secret use).
- AC-03 → PyTest/CI scan (controller-framework inventory).
- AC-04 → Helm/CI (single-release subchart + matching pinned versions).
- AC-05/06 → PyTest + e2e (`/v1/...` versioning gate; capability + audit event emission).
Fixtures: fake LiteLLM admin API until A1 lands; stub OPA data sink until A7.

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/authoring`
### 6.2 PR — `piece/ADR-0006-kopf-litellm` → base `wave/authoring`; carries spec + plan.
### 6.3 Merge order — independent of sibling ADR PRs; rolls up to main.

## 7. Effort Estimate
TASK-01..05 each S. Rollup: S. Critical path: TASK-01 → TASK-02.

## 8. Rollback / Reversibility
Backing out means re-opening the LiteLLM-reconciler choice (Go Crossplane provider) and re-homing eight CRDs. Downstream breakage: A17/B16/B17 and the entire declarative capability surface. Low reversibility once capabilities are GitOps-managed through B13; the controller-framework invariant would also be violated by any bespoke replacement.
