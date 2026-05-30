# PLAN ADR-0025 — Memory access modes per memory store [PROPOSED]

> spec: SPEC-ADR-0025 · kind: ADR · tier: T1
> wave: authoring-parallel · estimate: M
> upstream-pieces: [A10, A7, B4] · downstream-pieces: [B11, A18, A5]

## 1. Implementation Strategy
Enforcement/verification map. ADR 0025 is enforced by: **B4** (the `MemoryStore` XR carries the required `accessMode` enum), **A7** (a Gatekeeper admission policy rejecting missing/invalid `accessMode` and serving as the RBAC/OPA-mode decision point), **A10/B11** (the Letta-backed service and adapter honoring the declared mode identically), **A5** (ARK reconciling `Memory.memoryStoreRef` so the referenced store's mode governs), and **A18** (audit attributing reads/writes to agent identity + mode). Conformance is tested by admission accept/reject, structural enforcement of private/namespace-shared without OPA round-trips, RBAC/OPA-mode obeying RBAC-as-floor, backend-parity of the mode semantics, and mode-attributed audit records.

## 2. Ordered Task List
- TASK-01: Map each REQ to its enforcing piece (B4 field, A7 admission, A10/B11 backends, A5 ref-resolution, A18 audit) — produces: enforcement matrix — depends-on: []
- TASK-02: Verify `MemoryStore` XR requires exactly one of three `accessMode` values — produces: B4 schema check — depends-on: [TASK-01]
- TASK-03: Verify Gatekeeper admission rejects missing/invalid `accessMode` — produces: A7 admission conformance — depends-on: [TASK-02]
- TASK-04: Define structural-enforcement tests (private = writer identity; namespace-shared = membership; no OPA call) — produces: Chainsaw+PyTest set — depends-on: [TASK-02]
- TASK-05: Define RBAC/OPA-mode test (RBAC floor; OPA restricts, never grants beyond) — produces: PyTest+Chainsaw — depends-on: [TASK-03]
- TASK-06: Define backend-parity test (same mode suite on Letta-backed store and Crossplane XR) — produces: PyTest set — depends-on: [TASK-04]
- TASK-07: Verify `Agent`→store-by-name resolution applies the referenced mode; verify mode-attributed audit — produces: A5/A18 conformance — depends-on: [TASK-04]

## 3. Dependency Map
### 3.1 Upstream pieces that must ship first (HARD)
- B4 (`MemoryStore` XR + `accessMode` field), A7 (Gatekeeper admission + RBAC/OPA decision), A10 (Letta backend).
### 3.2 Downstream pieces blocked on this
- B11 (memory adapter surfacing the mode), A18 (mode-attributed audit), A5 (ARK ref-resolution).
### 3.3 Continuous (non-blocking) inputs
- ADR 0018 (RBAC-floor/OPA-restrictor), ADR 0016 (namespace tenancy); B22 threat model (unintended-access-excess detection).

## 4. Parallelizable Subtasks
TASK-04 and TASK-05 fan out once TASK-02/03 land. TASK-06 and TASK-07 follow TASK-04 independently.

## 5. Test Strategy
- AC-01 → Chainsaw: admit valid, reject missing/invalid `accessMode`.
- AC-02 → PyTest: assert no override path mutates `accessMode` post-declaration.
- AC-03 → Chainsaw+PyTest: private denies cross-agent read; namespace-shared permits same-ns; assert zero OPA calls.
- AC-04 → PyTest+Chainsaw: RBAC/OPA store grants only where RBAC permits and OPA doesn't deny; OPA can't grant beyond RBAC.
- AC-05 → PyTest: same mode suite green on Letta-backed store and Crossplane XR.
- AC-06 → Chainsaw: `Agent` referencing store by name allowed/denied per mode.
- AC-07 → PyTest: read and write each emit audit with agent identity + declared mode.
- Fixtures/fakes: stub Letta + OPA decision endpoint + audit endpoint for not-yet-landed A10/A7/A18.

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/authoring`
### 6.2 PR — `piece/ADR-0025-memory-access-modes` → base `wave/authoring`; carries spec + plan.
### 6.3 Merge order — independent of sibling ADR specs; rolls up to main.

## 7. Effort Estimate
M overall. Per-task: TASK-01 S, 02 S, 03 M, 04 M, 05 M, 06 M, 07 S. Critical path: TASK-01 → 02 → 03 → 05.

## 8. Rollback / Reversibility
Decision record; back out by reverting spec+plan. If reverted, B4/A7/A10/B11/A18 lose the three-mode conformance contract (admission validity, structural enforcement, backend parity, mode-attributed audit); no runtime artifact is deleted.
