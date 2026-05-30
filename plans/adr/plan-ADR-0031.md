# PLAN ADR-0031 — CloudEvent top-level type taxonomy `[PROPOSED]`

> spec: SPEC-ADR-0031 · kind: ADR · tier: T0
> wave: authoring-parallel · estimate: S
> upstream-pieces: [A4] · downstream-pieces: [B12; B8; B19; A18; A19; A20; A23]

## 1. Implementation Strategy
Enforcement/verification map. ADR 0031 is enforced primarily at **B12** (the schema registry),
which is the single source of truth for concrete event types and the natural gate for the
closed-namespace invariant; secondarily at **A4/B8** (Trigger filtering must work on namespace
prefix); and conformance is asserted at the architecture-CI layer (conftest/registry-admission)
that rejects any event type outside the ten namespaces. Each emitting component (A18, A19, A20,
A23, B19) honors the taxonomy by registering its types in B12 before emission.

## 2. Ordered Task List
- TASK-01: Encode the closed ten-namespace set as a machine-checkable allowlist — produces: conformance rule/fixture — depends-on: []
- TASK-02: Gate B12 registry admission on namespace membership + `schemaVersion` presence — produces: B12 registry validation — depends-on: [TASK-01]
- TASK-03: Verify Trigger prefix-filtering routes audit vs security disjointly — produces: Chainsaw eventing test — depends-on: [TASK-01]
- TASK-04: Verify breaking-change-mints-new-type vs minor-bump rule — produces: PyTest versioning test — depends-on: [TASK-02]
- TASK-05: Verify new-top-level-namespace-without-ADR fails the gate — produces: negative conformance test — depends-on: [TASK-01]

## 3. Dependency Map
### 3.1 Upstream pieces that must ship first (HARD)
- A4 — Knative + NATS broker (transport; `type` is the routing key).
### 3.2 Downstream pieces blocked on this
- B12, B8, B19, A18, A19, A20, A23 (all bind events to these namespaces).
### 3.3 Continuous (non-blocking) inputs
- B14 test framework (conformance harness); B22 threat model (security-namespace classification).

## 4. Parallelizable Subtasks
TASK-03, TASK-04, TASK-05 run concurrently once TASK-01/02 land.

## 5. Test Strategy
- AC-01, AC-05 → conftest/registry conformance (TASK-01/02/05).
- AC-02 → PyTest sampling broker events for `specversion`+`schemaVersion`.
- AC-03 → PyTest versioning test (TASK-04).
- AC-04 → Chainsaw eventing test on a live broker (TASK-03); fake B12 fixture until B12 lands.

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/authoring`
### 6.2 PR — `piece/ADR-0031-cloudevent-taxonomy` → base `wave/authoring`; carries spec + plan.
### 6.3 Merge order — independent of sibling ADR specs; rolls up to main.

## 7. Effort Estimate
TASK-01 S · TASK-02 S · TASK-03 S · TASK-04 S · TASK-05 S. Rollup S. Critical path: TASK-01 → TASK-02 → TASK-04.

## 8. Rollback / Reversibility
Reverting the conformance gate re-permits ad-hoc namespaces; downstream Trigger filters and the
audit-consumption boundary (ADR 0034) lose their guarantee. No data migration to back out — the
artifact is a validation rule plus tests.
