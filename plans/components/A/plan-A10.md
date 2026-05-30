# PLAN A10 — Letta memory backend

> spec: SPEC-A10 · kind: COMPONENT · tier: T1
> wave: W1 · estimate: M
> upstream-pieces: [A11] · downstream-pieces: [B11]

## 1. Implementation Strategy

Install Letta unmodified at a pinned version (config + wrap, ADR 0005), namespace-scoped, and wire it to the substrate-abstracted Postgres (system of record) and OpenSearch (reproducible retrieval index) through the connection secrets B4's `XPostgres`/`XSearchIndex` write — never branching on substrate. The non-negotiable invariants to enforce and test are: (1) agents reach Letta only via `memory.*`/the B11 adapter behind the `Memory` CRD; (2) the three `MemoryStore` access modes behave identically regardless of backend; (3) everything Letta writes to OpenSearch is reproducible from Postgres/object storage. Critical path: pin Letta → Postgres wiring → access-mode behavior → audit + reproducibility → deliverables. Build with fakes for A5 `Memory`, B4 XRs, B6 SDK, and B11 adapter until they land.

## 2. Ordered Task List

- TASK-01: Select + pin Letta version; author namespace-scoped Helm values/manifests — produces: ArgoCD-syncable Letta install — depends-on: []
- TASK-02: Wire Letta state to managed Postgres via `XPostgres` connection secret (kind + AWS, no substrate branch) — produces: durable state path — depends-on: [TASK-01]
- TASK-03: Wire Letta retrieval index to OpenSearch (A11) via `XSearchIndex`; verify reproducibility from primaries — produces: retrieval path + reproducibility test — depends-on: [TASK-01]
- TASK-04: Enforce reach-only-via-`memory.*`/adapter (network policy + service exposure) — produces: locked-down access surface — depends-on: [TASK-01]
- TASK-05: Implement access-mode behavior for private / namespace-shared / RBAC-OPA against `MemoryStore.accessMode` — produces: access-mode matrix — depends-on: [TASK-02, TASK-04]
- TASK-06: Link audit adapter; emit memory read/write events with identity + access mode; emit `platform.lifecycle.*` via `MemoryStore` lifecycle — produces: audit + lifecycle events — depends-on: [TASK-05]
- TASK-07: Author Gatekeeper admission for `Memory`/`MemoryStore` (incl. `accessMode` validation); contribute targets to B16 — produces: admission gating — depends-on: [TASK-01]
- TASK-08: Document + test backup/restore (Postgres PITR + OpenSearch rebuild) — produces: DR procedure — depends-on: [TASK-02, TASK-03]
- TASK-09: Build HolmesGPT memory toolset + (where useful) Headlamp memory-store inspector plugin — produces: toolset + plugin — depends-on: [TASK-05]
- TASK-10: 3-layer tests, docs, runbook, alerts, `GrafanaDashboard` XR, tutorials/how-tos; register version pin in SDK matrix — produces: full deliverable set — depends-on: [TASK-06, TASK-07, TASK-08, TASK-09]

## 3. Dependency Map

### 3.1 Upstream pieces that must ship first (HARD)
- **A11** (OpenSearch) — retrieval index host.
- **B4 `XPostgres`** (effective HARD, `[PROPOSED]` — stated in §6.3/ADR 0005, not in CSV edge) — Postgres system of record.

### 3.2 Downstream pieces blocked on this
- **B11** (memory backend adapter wraps Letta).

### 3.3 Continuous (non-blocking) inputs
- B14 test framework, B22 threat model. A5 `Memory` CRD, B6 SDK, A18 audit endpoint — consumed as they land (fakes until then).

## 4. Parallelizable Subtasks

- After TASK-01: TASK-02, TASK-03, TASK-04, TASK-07 run concurrently.
- After TASK-05: TASK-06 and TASK-09 run concurrently; TASK-08 follows TASK-02/03.
- TASK-10 fans in.

## 5. Test Strategy

| AC | Layer | Notes / fixtures |
|---|---|---|
| AC-A10-08 (admission), AC-A10-07 (lifecycle CloudEvents) | Chainsaw | `MemoryStore` admission + lifecycle; fake B4 XR |
| AC-A10-11 (Headlamp plugin) | Playwright | memory-store inspector |
| AC-A10-02/03/04/05/06/09/10/12 | PyTest | substrate-agnostic Postgres start, OpenSearch reproducibility rebuild, lockdown, access-mode matrix, audit attribution, restore, toolset, SDK-matrix entry |

Fakes for not-yet-landed upstreams: A5 `Memory` CRD, B4 `XPostgres`/`XSearchIndex` + connection secrets, B6 SDK `memory.*`, B11 adapter, A18 audit sink, A7/B16 OPA bundle.

## 6. PR / Branch Mapping

### 6.1 Stack position — base branch = `wave/1` (rolls up A11 from W0)
### 6.2 PR — `piece/A10-letta-memory-backend` → base `wave/1`; carries SPEC-A10 + PLAN-A10
### 6.3 Merge order — independent of W1 siblings (A5, A18, B13, B4); `wave/1` rolls up to main.

## 7. Effort Estimate

- S: TASK-01, TASK-07, TASK-09. M: TASK-02, TASK-03, TASK-04, TASK-06, TASK-08, TASK-10. L: TASK-05.
- Rollup: **M** (matches CSV).
- Critical path: TASK-01 → 02 → 05 → 06 → 10.

## 8. Rollback / Reversibility

Revert the ArgoCD Letta release. Because Postgres is the system of record (owned by B4 substrate, not A10), reverting the Letta install does not destroy memory state — reinstalling reconnects to the same Postgres; OpenSearch indexes rebuild from primaries. Downstream breakage: B11 adapter and all `memory.*` traffic stall while Letta is absent; agents depending on memory degrade. Access-mode regressions are the highest-blast-radius reversal risk (cross-namespace sharing correctness) — gate any access-mode change behind the policy simulator before rollout.
