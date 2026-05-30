# PLAN V6-12 — CRD inventory [PROPOSED]

> spec: SPEC-V6-12 · kind: VIEW · tier: T0
> wave: authoring-parallel · estimate: S
> upstream-pieces: [] · downstream-pieces: [A5, A6, B13, B19, B4]

## 1. Implementation Strategy
Realization map. This view is not built directly; it is realized collectively by **every CRD
reconciler owner** — A5 (ARK CRDs), A6 (sandbox CRDs), B13 (capability/key/budget CRDs), B19
(`Approval`), B4 (all XRs/XRDs), and per-component `LogLevel`. The view's job is to keep the union of
all component-defined CRDs equal to the master inventory and to hold the namespaced-everything,
per-component-ownership, and X-prefix/claim invariants. The realization is continuous: each owner's
spec is checked against this inventory, and the inventory is checked against interface-contract.md §1
(done in SPEC §10 — no membership divergence found). Verification is a conformance diff, not a build.

## 2. Ordered Task List
- **TASK-01:** Bind the §4.1 master inventory to reconciler owners (A5/A6/B13/B19/B4 + per-component
  LogLevel) — produces: CRD→owner trace — depends-on: [].
- **TASK-02:** Run the interface-contract.md §1 cross-check; record divergences — produces: §10
  divergence log (completed: no missing/extra CRDs; 5 naming nuances + 3 `[PROPOSED]` claim
  spellings) — depends-on: [TASK-01].
- **TASK-03:** Define the conformance-diff verification path (union-of-specs == inventory;
  namespaced-only; owner-traceable; XR↔claim resolves to one XRD) — produces: §5 AC→layer map —
  depends-on: [TASK-01].
- **TASK-04:** Route open questions (`LogLevel` versioning-column owner; derived claim spellings) to
  V6-13 / ADR 0041 owners; cross-link realizing views — produces: routing + reference set —
  depends-on: [TASK-02].

## 3. Dependency Map
### 3.1 Upstream pieces that must ship first (HARD)
None — VIEW is authoring-parallel; it imposes constraints, consumes no build outputs.
### 3.2 Downstream pieces blocked on this — ids
A5, A6, B13, B19, B4 (and per-component LogLevel owners) bind to this inventory (constraint
dependency). V6-13 consumes the inventory it versions.
### 3.3 Continuous (non-blocking) inputs
B14 test framework (Chainsaw CRD-apply conformance harness); interface-contract.md §1 (the registry
this view diffs against); B22 threat model (complete-inventory-as-security-property).

## 4. Parallelizable Subtasks
TASK-02 and TASK-03 run concurrently after TASK-01; TASK-04 follows TASK-02.

## 5. Test Strategy
Maps SPEC §9 ACs to layers; realized by the reconciler owners, verified as a conformance diff:
- AC-V6-12-01 → **PyTest**: union of CRDs across all component specs == §4.1 inventory; no orphan, no
  unlisted CRD.
- AC-V6-12-02 → **Chainsaw**: every installed platform CRD is namespaced; a cluster-scoped platform
  CRD (other than Crossplane composite XRs) fails.
- AC-V6-12-03 → **PyTest**: each CRD's `CustomResourceDefinition`/conversion-webhook ownership traces
  to a §3 reconciler.
- AC-V6-12-04 → **Chainsaw**: for each substrate XRD, the `X`-prefixed XR and unprefixed claim resolve
  to one XRD; no variant name appears in any spec.
- AC-V6-12-05 → **PyTest**: diff of inventory vs interface-contract.md §1 yields only the §10 log
  items (target: empty after reconciliation).
Fixtures: a spec-index scraper to build the union set; CRD manifests stubbed until owners land.

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/auth` (views authoring band)
### 6.2 PR — `view/V6-12-crd-inventory` → base `wave/auth`; carries spec + plan
### 6.3 Merge order — independent of sibling views; rolls up to main with the auth band. Note: as the
master CRD list, this view should be reviewed alongside V6-13 (versioning) and interface-contract.md.

## 7. Effort Estimate
S overall. TASK-01 S, TASK-02 S, TASK-03 S, TASK-04 S. Critical path: TASK-01 → TASK-02 → TASK-04.

## 8. Rollback / Reversibility
Pure documentation/contract; reverting removes the master inventory every reconciler owner binds to.
No runtime blast radius, but high authoring blast radius (T0): without it, component specs lose the
single source of CRD truth and divergence detection. Revert only with a replacement inventory.
