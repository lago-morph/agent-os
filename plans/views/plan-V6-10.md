# PLAN V6-10 — Platform self-management with HolmesGPT [PROPOSED]

> spec: SPEC-V6-10 · kind: VIEW · tier: T1
> wave: authoring-parallel · estimate: S
> upstream-pieces: [] · downstream-pieces: [A14, B10, A16, B19]

## 1. Implementation Strategy
Realization map. This view is not built directly; it is realized in wave order. The earliest and
critical-path realizer is **A14** (W0) — HolmesGPT lands very early on the raw cluster (read-only SA
+ read-only AWS IAM, no secret-store access) and is progressively rewired through LiteLLM, the audit
adapter, and OPA decision points as those layers land. **A16** (W2, the Interactive Access Agent)
realizes the shared-Knowledge-Base invariant; **B10** (W3, the Coach Component) realizes the
self-improvement path; **B19** (W3, approval system) is the `upon-approval` hand-off target. The
read-only-first posture, the three-state OPA model, and the toolset-contribution convention (SPEC §6)
are verified end-to-end across the phased trajectory rather than at one point in time.

## 2. Ordered Task List
- **TASK-01:** Bind SPEC §6 invariants to realizing piece IDs — produces: realization trace
  (REQ-01/02/03/07/09/10 → A14; REQ-04/05 → A14+OPA(B16)+B19; REQ-06 → A14+A16+B13 RAGStore;
  REQ-08 → all toolset-contributing components, non-blocking) — depends-on: [].
- **TASK-02:** Confirm the `upon-approval` hand-off boundary with V6-? / ADR 0017 (this view
  references B19, does not build it) — produces: deferral note — depends-on: [TASK-01].
- **TASK-03:** Define the phased-trajectory verification path (phase-1 read-only diagnostics →
  rewired-through-layers) and the three-state decision test — produces: §5 AC→layer map —
  depends-on: [TASK-01].
- **TASK-04:** Cross-link to V6-04 (Knowledge Base primitive) and V6-06 (security/policy) — produces:
  reference set — depends-on: [TASK-01].

## 3. Dependency Map
### 3.1 Upstream pieces that must ship first (HARD)
None — VIEW is authoring-parallel; it imposes constraints, consumes no build outputs.
### 3.2 Downstream pieces blocked on this — ids
A14, B10, A16 bind to this contract; B19 is referenced for the approval hand-off (constraint
dependency, not a build block).
### 3.3 Continuous (non-blocking) inputs
Toolset-contribution edges from every component into A14 are non-blocking (waves.md); the dotted
`A13 -.-> A14`, `A2 -.-> A14` edges set no wave depth. B14 test framework; B22 threat model
(self-referential-risk assumptions); B12 registry (event schemas, deferred).

## 4. Parallelizable Subtasks
TASK-02, TASK-03, TASK-04 run concurrently after TASK-01.

## 5. Test Strategy
Maps SPEC §9 ACs to layers; realized by A14/B10/A16 + policy/approval owners, verified end-to-end:
- AC-V6-10-01 → **Chainsaw/PyTest**: HolmesGPT exists as `Agent` CRD; after layers land, LLM traffic
  visible via LiteLLM, audit via adapter, tool calls hit OPA; assert no out-of-band admin path.
- AC-V6-10-02 → **PyTest**: a proposed write resolves to allowed/upon-approval/denied; denied is
  audited.
- AC-V6-10-03 → **Chainsaw**: `upon-approval` creates an `Approval` and does not execute until a
  decision is recorded (B19 fake until it lands).
- AC-V6-10-04 → **Chainsaw**: HolmesGPT + Interactive Access Agent resolve the same
  `platform-knowledge-base` RAGStore; removing it from HolmesGPT's CapabilitySet drops access without
  rebuild.
- AC-V6-10-05 → **PyTest**: peer A2A hand-off to HolmesGPT through LiteLLM is audited.
- AC-V6-10-06 → **Chainsaw**: phase-1 read-only SA + read-only IAM + no secret binding allows
  diagnostics, denies writes/secret reads.
Fixtures: fake LiteLLM + OPA decision endpoint + audit adapter until A1/A7/A18 land; fake `Approval`
reconciler until B19; read-only IAM fixture.

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/auth` (views authoring band)
### 6.2 PR — `view/V6-10-holmesgpt-self-management` → base `wave/auth`; carries spec + plan
### 6.3 Merge order — independent of sibling views; rolls up to main with the auth band

## 7. Effort Estimate
S overall. TASK-01 S, TASK-02 S, TASK-03 S, TASK-04 S. Critical path: TASK-01 → TASK-03.

## 8. Rollback / Reversibility
Pure documentation/contract; reverting removes the self-management contract A14/B10/A16 bind to. No
runtime blast radius; downstream specs would lose their §6.10 anchor (three-state model, KB-as-shared
primitive, phased trajectory).
