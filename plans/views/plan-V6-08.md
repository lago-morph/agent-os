# PLAN V6-08 — Capability registries and approved primitives [PROPOSED]

> spec: SPEC-V6-08 · kind: VIEW · tier: T1
> wave: authoring-parallel · estimate: S
> upstream-pieces: [] · downstream-pieces: [B13, A12, A17, B17]

## 1. Implementation Strategy
Realization map. This view is not built directly; it is realized by the capability-registry
components in wave order. The critical-path realizer is **B13** (W1) — the kopf operator that turns
all capability CRDs into LiteLLM/Envoy state and emits `platform.capability.changed`. Once B13
lands, **A17** (W2) populates the approved MCP-service set, **B17** (W2) ships reusable
CapabilitySet profiles that exercise the overlay contract, and **A12** (consumer wave) extends the
registry via OpenAPI→MCP synthesis. The enforcement-triad and notification invariants (SPEC §6) are
verified end-to-end against the running registry rather than per-component.

## 2. Ordered Task List
- **TASK-01:** Bind SPEC §6 invariants to realizing piece IDs — produces: realization trace
  (REQ-V6-08-01/02/04/05/06/07 → B13; REQ-03 → B13+ADR 0032; REQ-08 → B13+OPA/V6-09) — depends-on: [].
- **TASK-02:** Confirm overlay-resolution ownership boundary with ADR 0032 (mechanics vs grant-scope)
  — produces: deferral note — depends-on: [TASK-01].
- **TASK-03:** Define the end-to-end verification path for the four spanning invariants — produces:
  §5 AC→layer map — depends-on: [TASK-01].
- **TASK-04:** Cross-link to V6-09 (cross-tenant publish) and V6-12 (CRD inventory) — produces:
  reference set — depends-on: [TASK-01].

## 3. Dependency Map
### 3.1 Upstream pieces that must ship first (HARD)
None — VIEW is authoring-parallel; it imposes constraints, consumes no build outputs.
### 3.2 Downstream pieces blocked on this — ids
B13, A12, A17, B17 bind to this contract (constraint dependency, not a build block).
### 3.3 Continuous (non-blocking) inputs
B14 test framework (Chainsaw/PyTest harness for CRD + event assertions); B22 threat model
(enforcement-triad assumptions); B12 registry (event schema, deferred).

## 4. Parallelizable Subtasks
TASK-02, TASK-03, TASK-04 run concurrently after TASK-01.

## 5. Test Strategy
Maps SPEC §9 ACs to layers; realized by B13's deliverables, verified end-to-end:
- AC-V6-08-01/02/06 → **Chainsaw**: apply capability CRDs + ordered CapabilitySets + Agent
  overrides; assert resolved state and reachability/unreachability.
- AC-V6-08-03 → **PyTest/Chainsaw**: mutate a capability CRD; assert exactly one
  `platform.capability.changed` event on the broker.
- AC-V6-08-04 → **PyTest**: drop a capability, suppress notification, attempt use; assert perimeter
  denial (LiteLLM/OPA/Envoy fakes until those land).
- AC-V6-08-05 → **Playwright**: Headlamp resolved per-Agent capability view renders.
- AC-V6-08-06 → **Chainsaw**: cross-namespace reference without OPA publication is rejected.
Fixtures: fake LiteLLM admin API + fake Envoy egress allowlist until A1/A6 land; B12 event-schema stub.

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/auth` (views authoring band)
### 6.2 PR — `view/V6-08-capability-registries` → base `wave/auth`; carries spec + plan
### 6.3 Merge order — independent of sibling views; rolls up to main with the auth band

## 7. Effort Estimate
S overall. TASK-01 S, TASK-02 S, TASK-03 S, TASK-04 S. Critical path: TASK-01 → TASK-03.

## 8. Rollback / Reversibility
Pure documentation/contract; reverting removes the spanning contract B13/A12/A17/B17 bind to. No
runtime blast radius; downstream component specs would lose their integration anchor for §6.8.
