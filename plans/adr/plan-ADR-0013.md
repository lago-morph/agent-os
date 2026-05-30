# PLAN ADR-0013 — Capability CRD model and CapabilitySet layering [PROPOSED]

> spec: SPEC-ADR-0013 · kind: ADR · tier: T0
> wave: authoring-parallel · estimate: M
> upstream-pieces: [B13] · downstream-pieces: [A1, A6, A7, A9, B6, B16, B17, A22]

## 1. Implementation Strategy
Enforcement/verification map. ADR 0013 is enforced primarily by **B13** (the kopf operator that reconciles the six capability CRDs + `CapabilitySet` into LiteLLM and adjacent state, emits `platform.capability.changed`, and owns CRD versioning), with the "single source of truth" property enforced across **A1** (LiteLLM registry), **A6** (Envoy allowlists), **A7** (Gatekeeper admission + OPA gating against `capability_set_refs`), **A9/A22** (Headlamp resolved view + graphical editor), and **B6** (SDK capability-change callback + `refresh_capabilities()`). Conformance is tested by proving all CRDs reconcile, that the same CapabilitySet drives every enforcement/observability surface, that hard enforcement survives suppressed notification, namespace-private defaults hold, and versioning is conversion-webhook-gated. Overlay-resolution conformance is deferred to ADR 0032.

## 2. Ordered Task List
- TASK-01: Map each REQ to its enforcing component piece ID — produces: enforcement matrix — depends-on: []
- TASK-02: Verify B13 SPEC reconciles all six capability CRDs + `CapabilitySet` into LiteLLM/adjacent state — produces: B13 conformance checklist — depends-on: [TASK-01]
- TASK-03: Verify single-source-of-truth fan-out (LiteLLM/Envoy/OPA/RBAC/Headlamp/audit/observability) — produces: SoT trace matrix — depends-on: [TASK-01]
- TASK-04: Verify governance hooks (Gatekeeper admit, OPA gate on `capability_set_refs`, RBAC create/bind) — produces: governance conformance note — depends-on: [TASK-01]
- TASK-05: Define `platform.capability.changed` emission + SDK callback + `refresh_capabilities()` conformance — produces: PyTest+Chainsaw set — depends-on: [TASK-02]
- TASK-06: Define hard-enforcement-survives-suppressed-notification conformance — produces: PyTest test — depends-on: [TASK-02]
- TASK-07: Verify Headlamp resolved-per-Agent-view obligation — produces: A22 trace — depends-on: [TASK-03]
- TASK-08: Verify namespace-private default + OPA-checked publication for cross-namespace refs — produces: Chainsaw test — depends-on: [TASK-04]
- TASK-09: Define CRD-versioning conformance (breaking change requires `vN` + webhook) — produces: CI gate — depends-on: [TASK-02]

## 3. Dependency Map
### 3.1 Upstream pieces that must ship first (HARD)
- B13 — the kopf operator reconciling every capability CRD and emitting the change event.
### 3.2 Downstream pieces blocked on this
- A1 (registry), A6 (Envoy), A7 (gating), A9/A22 (resolved view/editor), B6 (SDK callback), B16 (OPA content), B17 (profiles as CapabilitySets).
### 3.3 Continuous (non-blocking) inputs
- ADR 0032 overlay semantics (resolves deferred sub-semantics); B14 test framework; B22 threat model.

## 4. Parallelizable Subtasks
TASK-03, TASK-04 fan out once TASK-01 lands. TASK-05/06/09 fan out once TASK-02 lands; TASK-07 after TASK-03; TASK-08 after TASK-04.

## 5. Test Strategy
- AC-01 → Chainsaw: `api-resources` shows all CRDs reconciled by B13; gateway-only config is inert.
- AC-02/AC-03 → Chainsaw: layer a CapabilitySet; PyTest: assert one Agent's LiteLLM/Envoy/OPA/Headlamp/audit surfaces all derive from it.
- AC-05 → PyTest: Gatekeeper denies malformed CRD; OPA denies registration with absent `capability_set_refs`; RBAC denies unauthorized bind.
- AC-06 → Chainsaw: cross-namespace reference blocked until OPA-checked publication.
- AC-07/AC-08 → PyTest: edit set → exactly one change event; non-subscriber refreshes within one turn.
- AC-09 → PyTest: with notification suppressed, removed capability still denied at LiteLLM/OPA/Envoy.
- AC-10 → Playwright: Headlamp renders resolved per-Agent view matching applied state.
- AC-11 → CI gate: breaking CRD change without `vN`+webhook fails.
- Fixtures/fakes: stub LiteLLM admin API + Envoy config + OPA decision for not-yet-landed A1/A6/A7.

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/authoring`
### 6.2 PR — `piece/ADR-0013-capability-crd-capabilityset` → base `wave/authoring`; carries spec + plan.
### 6.3 Merge order — independent of sibling ADR specs; rolls up to main. Coordinate with ADR 0032 spec for overlay sub-semantics.

## 7. Effort Estimate
M overall (T0, broad fan-out). Per-task: TASK-01 S, 02 M, 03 M, 04 S, 05 M, 06 S, 07 S, 08 S, 09 S. Critical path: TASK-01 → 02 → 03 → 07.

## 8. Rollback / Reversibility
Decision record; back out by reverting spec+plan. If reverted, the platform loses its single-source-of-truth contract for capabilities and the change-notification model; B13/A1/A6/A7/A22 lose their conformance anchor. No runtime artifact is deleted; overlay sub-semantics remain owned by ADR 0032 regardless.
