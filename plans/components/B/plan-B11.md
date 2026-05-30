# PLAN B11 — Memory backend adapter

> spec: SPEC-B11 · kind: COMPONENT · tier: T1
> wave: W1 · estimate: M
> upstream-pieces: [A10] · downstream-pieces: []

## 1. Implementation Strategy
Build a **thin adapter** that is the single seam between Letta (A10) and the platform's memory
contract (ARK's `Memory` CRD + the SDK `memory.*` surface from B6). Localize every Letta-specific
assumption so a future backend swap is a B11 re-implementation, not a platform change (ADR 0005).
Enforce the per-store **access modes** at the backend boundary (private/namespace-shared
structurally; RBAC/OPA-controlled via RBAC-floor + OPA-restrictor). Consume the **B4-composed
`MemoryStore` + uniform connection secret** (ADR 0041) so the adapter never branches on substrate.
Because the `memory.*` signatures are B6-owned and not yet fixed, build against a **fake backend +
fake SDK seam** first and converge with B6.

## 2. Ordered Task List
- **TASK-01:** Define the adapter↔platform boundary contract with B6 (which `memory.*` ops map to
  which Letta calls); freeze the no-Letta-type-leak invariant. — produces: contract note — depends-on: []
- **TASK-02:** Implement the Letta-facing mapping layer against a pinned Letta version. — produces:
  adapter core — depends-on: [TASK-01]
- **TASK-03:** Implement access-mode enforcement: private (writer-only), namespace-shared (same-ns),
  RBAC/OPA-controlled (floor + OPA restrict). — produces: enforcement layer — depends-on: [TASK-02]
- **TASK-04:** Wire persistence: Letta state → Postgres (system of record); OpenSearch retrieval
  index kept rebuildable from Postgres/object storage (ADR 0014). — produces: persistence wiring —
  depends-on: [TASK-02]
- **TASK-05:** Consume the B4 `MemoryStore` connection secret (uniform shape) with no substrate
  branch; namespace-scoped Letta instance. — produces: substrate-agnostic wiring — depends-on: [TASK-02]
- **TASK-06:** Audit emission for reads/writes (agent identity + store mode) + `platform.lifecycle.*`
  `MemoryStore` events + OTel spans. — produces: observability wiring — depends-on: [TASK-03]
- **TASK-07:** Compatibility matrix entry (Letta version pin) + docs/runbook/dashboard XR. —
  produces: docs + pin — depends-on: [TASK-04, TASK-05]
- **TASK-08:** 3-layer tests incl. dual-substrate connection-secret test + OpenSearch-rebuild test. —
  produces: tests — depends-on: [TASK-03, TASK-04, TASK-05, TASK-06]

## 3. Dependency Map
### 3.1 Upstream pieces that must ship first (HARD)
- **A10 (Letta)** — the memory service B11 wraps; pinned tested version.
- **B4** — the `MemoryStore` XR + uniform connection secret (B4's downstream lists B11; treated HARD
  even though not in B11's CSV `upstream` column — see SPEC R4).
- **A5 (ARK)** — the `Memory` CRD (`memoryStoreRef`) B11 backs.
- (Implicit co-design) **B6** — the `memory.*` surface; signatures not yet source-fixed (R1).

### 3.2 Downstream pieces blocked on this
- None (`piece-index.csv` lists no downstream). At runtime every memory-using agent reaches B11 via
  the SDK/`Memory` CRD, but no build piece depends on it.

### 3.3 Continuous (non-blocking) inputs
- **B14** (three-layer tests), **B22** (threat-model standards — access-excess detection focus),
  **B16** (OPA policy *content* for the RBAC/OPA-controlled mode; B11 only invokes the decision point),
  **B12** (CloudEvent schemas for `platform.lifecycle.*`).

## 4. Parallelizable Subtasks
- After TASK-02: TASK-03 (enforcement), TASK-04 (persistence), TASK-05 (substrate wiring) run
  concurrently.
- TASK-06 follows TASK-03; TASK-07 follows TASK-04/05.

## 5. Test Strategy
- **Chainsaw:** apply `MemoryStore` per access mode → assert enforcement + `platform.lifecycle.*`
  + audit events (AC-B11-02,03,04,08,09).
- **PyTest:** adapter mapping; access-mode logic; dual-substrate connection-secret (no branch);
  backend-swap behind a fake backend; the no-direct-Letta-call architecture lint (AC-B11-01,06,10,11).
- **Playwright:** N/A — no UI.
- **Rebuild test:** delete + rebuild OpenSearch index, assert retrieval reproduced from Postgres/
  object storage (AC-B11-05).
- **Fakes:** fake `MemoryStore` + connection secret if B4 slips; fake Letta + fake OPA decision point.

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/1` (contains A10; consumes B4 also in W1).
### 6.2 PR — `piece/B11-memory-backend-adapter` → base `wave/1`; carries spec-B11 + plan-B11.
### 6.3 Merge order — independent of W1 siblings; coordinate with B4 (consumed). wave/1 rolls up to main.

## 7. Effort Estimate
- TASK-01 S · TASK-02 M · TASK-03 M · TASK-04 M · TASK-05 S · TASK-06 S · TASK-07 S · TASK-08 M.
- Rollup: **M** (matches CSV). Critical path: TASK-01 → 02 → 03 → 06 → 08.

## 8. Rollback / Reversibility
Reverting B11 disables the platform's only sanctioned path to memory — so any agent using `memory.*`
loses memory. Because Postgres is the system of record, **no data is lost** on rollback: state
persists; re-deploying a fixed B11 reconnects and the OpenSearch index is rebuildable. Rollback is
operationally safe but functionally breaking for memory-using agents; mitigate by keeping the prior
B11 image available and gating the swap behind the compatibility-matrix pin.
