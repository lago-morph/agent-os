# PLAN B22 — Security threat model design specification

> spec: SPEC-B22 · kind: COMPONENT (design specification, not code) · tier: T0
> wave: W0 (foundation; non-blocking fan-out to A and B) · estimate: L
> upstream-pieces: [] · downstream-pieces: [A, B] (continuous, non-blocking)

> **This is a design / authoring plan + enforcement map, NOT a build plan.** B22 is design, not
> code (ADR 0027). There is no deployable, no Helm, no controller. The "tasks" below are *authoring*
> tasks that produce *analysis and standards documents*; §2's "ordered task list" is the authoring
> sequence and §5 is a publish/conformance map rather than a Chainsaw/Playwright/PyTest test plan.

## 1. Implementation Strategy
Author the adversarial threat model as a structured design specification and publish it **before the
first wave of A/B implementation** so it feeds forward (ADR 0027). Work outward from the four
analytical pillars — adversary classes, asset inventory, trust boundaries, attack catalog — then
produce the per-attack mapping (control → residual risk → detecting observability), then distil the
**output standards** (mandatory acceptance criteria, mandatory test cases, mandatory dashboard
signals, OPA policy targets) and the **enforcement map** binding each standard to its owning
component. Because B22 is a **non-blocking fan-out input**, publish **incrementally per attack** so
each completed attack-mapping immediately unblocks the security review of the components it touches,
rather than holding everything for a single big-bang release. Finally, apply the forward-feed edits
to §6.6, the Workstream A deliverable lists, and the B16 OPA policy targets.

## 2. Ordered Task List (authoring sequence)
- **TASK-01:** Choose the analytical structure (`[PROPOSED]` STRIDE-per-trust-boundary + attack-tree
  per attack) and the standards/enforcement-map template — produces: method + templates — depends-on: []
- **TASK-02:** Adversary-class catalog (all §6.6 classes) — produces: adversary catalog — depends-on: [TASK-01]
- **TASK-03:** Asset inventory (all §6.6 assets incl. HolmesGPT/Coach) — produces: asset inventory —
  depends-on: [TASK-01]
- **TASK-04:** Trust-boundary map (all six §6.6 pairs) — produces: trust-boundary map — depends-on: [TASK-01]
- **TASK-05:** Attack catalog (all eight §6.6 patterns) — produces: attack catalog — depends-on: [TASK-02, TASK-03, TASK-04]
- **TASK-06:** Per-attack mapping for each catalogued attack: applicable controls (ref ADR
  0002/0003/0018 layers), residual risk, detecting observability — produces: per-attack mappings —
  depends-on: [TASK-05]
- **TASK-07:** Distil **output standards** — mandatory ACs, mandatory test cases, mandatory dashboard
  signals, OPA policy targets — per standard/attack — produces: standards set — depends-on: [TASK-06]
- **TASK-08:** **Enforcement map** — bind every standard/attack to owning component(s); confirm no
  standard is ownerless — produces: enforcement map (canonical registry) — depends-on: [TASK-07]
- **TASK-09:** Forward-feed edits: §6.6, Workstream A per-component deliverable lists, **B16 OPA policy
  targets** (ADR 0027) — produces: the actual cross-doc updates + B16 extension-point targets —
  depends-on: [TASK-08]
- **TASK-10:** Fence continuous-runtime-posture out of v1.0 into future-enhancements §2; state the
  scope-decision-not-permanent stance + revisit trigger; define the per-component **security-review
  gate** — produces: scope fence + review-gate definition — depends-on: [TASK-08]
- **TASK-11:** Publish to the docs portal + Knowledge Base; author the "how to run a component security
  review against B22" how-to — produces: published spec + how-to — depends-on: [TASK-09, TASK-10]

## 3. Dependency Map
### 3.1 Upstream pieces that must ship first (HARD)
- **None.** B22 is W0 foundation with no internal dependencies; it binds to frozen architecture
  context (§6.6) and the ADRs, not to build outputs.
### 3.2 Downstream pieces "blocked" on this (non-blocking fan-out)
- **Every A and B component** consumes B22's standards into its deliverable list / ACs / mandatory
  tests / dashboard signals — but as a **continuously available input, not a wave-depth-setting
  predecessor** (`_meta/waves.md`). Each component reaches **security-complete** only after review
  against B22's standards; B22 delay directly delays that state.
- **B16** — B22 updates its OPA policy targets (via B16's extension point, REQ-B16-13).
- **A13 / D1 / D2** — B22's detection signals become their alert rules / dashboard panels.
- **F4 (security review)** — compiles the per-component reviews against B22's standards.
### 3.3 Continuous (non-blocking) inputs into B22
- None required. (B22 *is* the canonical continuous input for the rest of the platform.)

## 4. Parallelizable Subtasks
- **TASK-02 (adversaries) / TASK-03 (assets) / TASK-04 (boundaries)** are an independent fan-out group
  after TASK-01.
- Within TASK-06, the **per-attack mappings fan out one-per-attack** — each can be authored
  concurrently and **published as it completes** to unblock the touched components early.
- TASK-09 forward-feed edits can begin per-component as soon as that component's standards in TASK-07/08
  are settled (no need to wait for the full set).

## 5. Publish / Conformance Map (replaces a code test strategy)
> B22 ships no test code; instead it *defines* the mandatory tests others run and is itself verified
> by completeness/forward-feed conformance (mapping to SPEC §9 ACs):
- **Completeness conformance:** AC-B22-01..05 — adversary/asset/boundary/attack-catalog coverage and
  no-unmapped-attack — verified by a structured checklist against the §6.6 minimum sets.
- **Output conformance:** AC-B22-06/07 — every standard yields a concrete AC/test/dashboard/OPA target
  and an owner — verified by the enforcement map having no ownerless rows.
- **Forward-feed conformance:** AC-B22-08 — concrete edits to §6.6, Workstream A lists, B16 targets
  exist and are referenced (not copied) by consuming specs.
- **Scope conformance:** AC-B22-09/10/11 — continuous-runtime-posture fenced to future-enhancements §2;
  scope-decision stance + revisit trigger stated; per-component security-review gate defined and
  referenceable.
- **Verification owner:** F4 (security review) confirms components conform to the standards at
  implementation-complete; B22 itself is "done" when the enforcement map + forward-feed edits publish.

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/0` (foundation; B22 has no upstream specs to sit on).
### 6.2 PR — `piece/B22-threat-model-design` → base `wave/0`; carries SPEC-B22 + PLAN-B22 + the threat
model standards documents + enforcement map. Per-attack mappings may land as stacked sub-PRs under the
piece branch to enable incremental publish.
### 6.3 Merge order — lands early in W0 (before the first A/B implementation wave). Forward-feed edits
to §6.6 / Workstream A lists / B16 targets are separate follow-up PRs against the touched pieces, each
referencing the B22 enforcement map.

## 7. Effort Estimate
- TASK-01 S, TASK-02 M, TASK-03 M, TASK-04 M, TASK-05 M, TASK-06 L (the bulk — per-attack mappings),
  TASK-07 M, TASK-08 M, TASK-09 M, TASK-10 S, TASK-11 S.
- **Rollup: L** (matches CSV). **Critical path:** TASK-01 → (TASK-02/03/04) → TASK-05 → TASK-06 →
  TASK-07 → TASK-08 → TASK-09. Incremental per-attack publishing shortens *time-to-first-unblock* even
  though full rollup stays on this path.

## 8. Rollback / Reversibility
B22 is a design document — "rollback" means superseding a published standards revision with a new one
(ADR 0030 versioning of the standards artifacts). Reverting B22 does not break a running system but
**removes the security-complete gate** for every dependent component (ADR 0027), so reversion is only
for replacing one revision with a better one, never for removing the standard. The scope stance
(everyday-mistake primary) is explicitly revisable (REQ-B22-10): as adversarial controls mature, a new
revision shifts the primary modeled threat and updates §6.6 rather than rolling B22 back.
