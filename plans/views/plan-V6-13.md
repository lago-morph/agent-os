# PLAN V6-13 — Versioning policy [PROPOSED]

> spec: SPEC-V6-13 · kind: VIEW · tier: T0
> wave: authoring-parallel · estimate: S
> upstream-pieces: [] · downstream-pieces: [A5, A6, B13, B19, B4, B12, B6, B9, B17, B18]

## 1. Implementation Strategy
Realization map. This view is not built directly; it is realized by **every component that exposes a
versioned surface** — CRD owners (A5/A6/B13/B19/B4 + per-component LogLevel), the CloudEvent registry
(B12), the Platform SDK (B6), the CLI (B9), agents that expose A2A/MCP interfaces (B17/B18), and
custom-HTTP-service owners. The view fixes the per-surface versioning model, the per-component
ownership invariant, and the no-lockstep constraint; each owner realizes it with conversion webhooks,
semver tooling, `schemaVersion` fields, URL-path versioning, and deprecation-warning emission.
Verification is a per-surface conformance check plus a breaking-change drill, not a build.

## 2. Ordered Task List
- **TASK-01:** Bind the six surface classes to owning components and the SPEC §6 REQs — produces:
  surface→owner→model trace (CRDs → A5/A6/B13/B19/B4; events → B12; SDK → B6; CLI → B9; agent
  interfaces → B17/B18; HTTP → service owners) — depends-on: [].
- **TASK-02:** Resolve the "deprecate ≥1 minor *platform* release" vs "no lockstep platform version"
  tension; adopt the cadence-marker reading and route confirmation to ADR 0030 owners — produces:
  reconciliation note — depends-on: [TASK-01].
- **TASK-03:** Define the conformance + breaking-change verification path per surface class —
  produces: §5 AC→layer map — depends-on: [TASK-01].
- **TASK-04:** Carry the `LogLevel`-versioning-owner open question from V6-12; cross-link V6-12 /
  V6-07 / V6-03 — produces: routing + reference set — depends-on: [TASK-01].

## 3. Dependency Map
### 3.1 Upstream pieces that must ship first (HARD)
None — VIEW is authoring-parallel; it imposes constraints, consumes no build outputs.
### 3.2 Downstream pieces blocked on this — ids
All CRD/SDK/CLI/interface/HTTP owners: A5, A6, B13, B19, B4, B12, B6, B9, B17, B18 + custom-HTTP
owners (constraint dependency, not a build block).
### 3.3 Continuous (non-blocking) inputs
B14 test framework (conversion-webhook + semver conformance harness); B12 registry (event-schema
versioning); interface-contract.md §1.1/§2/§3 (the versioning clauses this view consolidates);
B22 threat model (unversioned-surface-as-risk).

## 4. Parallelizable Subtasks
TASK-02, TASK-03, TASK-04 run concurrently after TASK-01.

## 5. Test Strategy
Maps SPEC §9 ACs to layers; realized by each surface owner, verified per-surface + via drills:
- AC-V6-13-01 → **PyTest**: each surface class in a sampled component declares a version in its model;
  an unversioned exposed surface fails.
- AC-V6-13-02 → **PyTest**: no artifact encodes a single global platform version gating component API
  versions in lockstep.
- AC-V6-13-03 → **Chainsaw**: a breaking CRD change is served via a new `vN` group + working
  conversion webhook; `vN-1` still served + deprecated ≥1 minor release.
- AC-V6-13-04 → **PyTest**: breaking event → new event type (old subscribers unaffected); additive →
  `schemaVersion` minor bump; new namespace blocked without an ADR.
- AC-V6-13-05 → **PyTest**: Platform SDK release ships a gateway/ARK/Letta compatibility matrix.
- AC-V6-13-06 → **PyTest**: agent exposes `name.vN`; a peer pinned to `vN` survives `vN+1` side-by-side.
- AC-V6-13-07 → **PyTest**: custom HTTP service serves `/v1/...` and keeps a deprecated prior version
  reachable ≥1 platform release.
- AC-V6-13-08 → **PyTest**: using a deprecated version emits a deprecation warning (log/metric/audit);
  per-product docs carry migration guidance.
Fixtures: fake conversion-webhook harness; semver release-fixture; B12 event-schema stub; deprecation
telemetry assertion against the audit adapter (faked until A18).

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/auth` (views authoring band)
### 6.2 PR — `view/V6-13-versioning-policy` → base `wave/auth`; carries spec + plan
### 6.3 Merge order — independent of sibling views; rolls up to main with the auth band. Review
alongside V6-12 (inventory) since both are T0 and share the `LogLevel`-owner open question.

## 7. Effort Estimate
S overall. TASK-01 S, TASK-02 S, TASK-03 S, TASK-04 S. Critical path: TASK-01 → TASK-03.

## 8. Rollback / Reversibility
Pure documentation/contract; reverting removes the versioning policy every CRD/SDK owner binds to. No
runtime blast radius, but high authoring blast radius (T0): without it, surfaces could evolve without
a common deprecation/compatibility discipline. Revert only with a replacement policy in place.
