# PLAN ADR-0040 — Kargo as the v1.0 promotion fabric for GitOps environments `[PROPOSED]`

> spec: SPEC-ADR-0040 · kind: ADR · tier: T0
> wave: authoring-parallel · estimate: S
> upstream-pieces: [A7; B19; B4; A9] · downstream-pieces: [A23; B5; A18]

(Front-matter note: `_meta/piece-index.csv` carries `wave=auth`, tier `T0`, estimate `S`, and
empty upstream/downstream for ADR rows. The edge set above is taken from the spec front-matter —
the enforcer/consumer set this plan maps — since the ADR is a constraint piece with no build wave.)

## 1. Implementation Strategy

This is an **ADR plan**: an **enforcement & verification map** for a settled decision, not a build
sequence. ADR 0040 fixes Kargo as the v1.0 promotion fabric (single-cluster, one Stage initially,
growing dev→staging→prod), a single mainline with `envs/<stage>/` path overlays (no per-environment
branches), the Warehouse → Stage flow with a per-Stage verification (smoke) step, the `Approval` CRD
as Kargo's human gate (composition — neither subsumes the other), OPA-gated promotion actions,
Keycloak OIDC for the UI + controller, uniform Crossplane claim shapes across substrates, and
recovery by **re-promoting a previous Warehouse commit** (not git-revert; history stays append-only).
The decision is **enforced by component pieces, not by the ADR**. The four enforcer pieces named in
the brief are **A23** (the Kargo install component: Helm values, GitOps wiring, per-Stage
verification configs, single-Stage→multi-Stage phasing, Keycloak OIDC, audit emission, the two-kind
promotion test), **B19** (the `Approval` CRD + Argo Workflow human gate Kargo composes with), **B5**
(the cross-cutting Kargo Headlamp plugin + per-Stage deep-links), and **B9** (the `agent-platform`
CLI that orchestrates the three-layer tests, including the two-kind promotion test). Supporting
enforcers: A7/B16 (OPA promotion-action policies), B4/ADR-0041 (uniform claim shapes), A18
(promotion-action audit), A9 (Headlamp deep-link host). The load-bearing question is conformance,
verified per §5 against the SPEC's eleven ACs.

## 2. Ordered Task List

`TASK-NN` here = a conformance-verification step mapped to an enforcer piece, not a build step (the
build lives in A23/B5/B19). Ordered so the verification critical path is visible.

- `TASK-01: Verify Kargo runs single-cluster with exactly one Stage; adding an environment adds a Stage with no re-architecture (A23) — produces: AC-01 evidence — depends-on: []`
- `TASK-02: Verify repo is single mainline + `envs/<stage>/` overlays, no long-lived per-env branches (A23) — produces: AC-02 evidence — depends-on: []`
- `TASK-03: Verify merged candidate lands in Warehouse; promotion updates `envs/<stage>/` to a new Warehouse commit hash (A23) — produces: AC-03 evidence — depends-on: [TASK-02]`
- `TASK-04: Verify each Stage's smoke-suite verification gate blocks onward promotion on failure (A23) — produces: AC-04 evidence — depends-on: [TASK-03]`
- `TASK-05: Verify approval-gated Stage creates an `Approval` CRD and proceeds only on the reported decision (A23 + B19) — produces: AC-05 evidence — depends-on: [TASK-03]`
- `TASK-06: Verify OPA promotion-action policy (time-of-day window) blocks/allows a promotion (A7 + B16) — produces: AC-06 evidence — depends-on: []`
- `TASK-07: Verify Keycloak OIDC authenticates UI (human) + cluster-OIDC workload identity (controller) (A23) — produces: AC-07 evidence — depends-on: []`
- `TASK-08: Verify same claim shape promotes across kind→AWS with no Kargo-visible substrate difference (B4, cross-cuts ADR-0041) — produces: AC-08 evidence — depends-on: []`
- `TASK-09: Verify recovery by promoting a previous Warehouse commit; no git-revert; history append-only (A23) — produces: AC-09 evidence — depends-on: [TASK-03]`
- `TASK-10: Verify every promotion action emits an audit event via the platform adapter (A23 + A18) — produces: AC-10 evidence — depends-on: [TASK-03]`
- `TASK-11: Verify a two-kind (dev+staging) setup exercises promotion mechanics in CI without AWS (B9 CLI + B14) — produces: AC-11 evidence — depends-on: []`
- `TASK-12: Assemble the conformance matrix (REQ→AC→enforcer-piece→test-layer) — produces: §5 map — depends-on: [TASK-01..TASK-11]`

## 3. Dependency Map

### 3.1 Upstream pieces that must ship first (HARD) — id + what is consumed
- **A7** (OPA/Gatekeeper) — promotion-action gating engine (who/what/where/when incl. time-of-day); consumed for REQ-06. Policy content is B16.
- **B19** (Generalized approval system; **B19-core** lands W3) — `Approval` CRD + Argo Workflow that reports the decision back to Kargo; consumed for REQ-05.
- **B4** (Crossplane v2 Compositions) — uniform claim shapes that absorb substrate asymmetry; consumed for REQ-08.
- **A9** (Headlamp install + framework) — deep-link host for the Kargo UI per Stage; consumed by §4.2 deep-links.

### 3.2 Downstream pieces blocked on this — ids
- **A23** (Kargo install component) — primary realizer; honors REQ-01..04, 07, 09, 10.
- **B5** (cross-cutting Kargo Headlamp plugin) — co-lands W4 with A23 (B19-ui co-lands); UI surface.
- **A18** (audit endpoint + adapter) — receives Kargo promotion-action audit events.
- (Cross-reference) **B9** CLI orchestrates the REQ-11 two-kind promotion test; **B16** carries the promotion-action Rego.

### 3.3 Continuous (non-blocking) inputs
- **B14** (agent-platform test framework) — the three-layer harness the two-kind promotion test runs in (fan-out edge, non-wave-setting).
- **B22** (security threat-model design) — OPA-gating / OIDC posture standards (fan-out edge).
- **B12** (CloudEvent schema registry) — concrete promotion-event-type names defer here (deferred per interface-contract §6).

## 4. Parallelizable Subtasks

- TASK-01, TASK-02, TASK-06, TASK-07, TASK-08, TASK-11 are mutually independent and run concurrently (distinct enforcers/surfaces).
- The Warehouse-flow chain TASK-03 → {TASK-04, TASK-05, TASK-09, TASK-10} fans out once TASK-03 holds.
- TASK-12 is the join point over all verification tasks.

## 5. Test Strategy

Conformance is proven by the **realizing components' three-layer tests** (the ADR ships no code).
Map of AC → layer → enforcing piece:

| AC | Requirement | Layer | Enforcing piece / fixture |
|---|---|---|---|
| AC-01 | one Stage; +1 env adds +1 Stage no re-arch | Chainsaw (Kargo objects) | A23 |
| AC-02 | single mainline + `envs/<stage>/`, no per-env branches | PyTest (repo-layout assertion) | A23 / B15 |
| AC-03 | merge → Warehouse; promotion updates `envs/<stage>/` to commit hash | Chainsaw (Warehouse/Stage/Promotion) | A23 |
| AC-04 | smoke-suite failure blocks onward promotion | Chainsaw + PyTest (verification step) | A23 |
| AC-05 | approval-gated Stage creates `Approval`; proceeds on reported decision | Chainsaw (`Approval` CRD) | A23 + B19-core |
| AC-06 | OPA time-of-day policy blocks/allows promotion | PyTest/conftest (Rego) + Chainsaw (admission) | A7 + B16 |
| AC-07 | Keycloak OIDC for UI + cluster-OIDC workload identity for controller | Playwright (UI login) + PyTest (controller auth) | A23 + B5 |
| AC-08 | same claim shape across kind→AWS, no Kargo-visible diff | Chainsaw (claim parity) | B4 (cross-cuts ADR-0041) |
| AC-09 | recovery by re-promoting a previous Warehouse commit; git append-only | Chainsaw + PyTest (no git-revert) | A23 |
| AC-10 | every promotion action appears as an audit event | PyTest (audit adapter integration) | A23 + A18 |
| AC-11 | two-kind (dev+staging) promotion mechanics in CI, AWS-free | PyTest/integration (two-kind harness) | B9 CLI + B14 |

Fixtures/fakes for not-yet-landed upstreams: a **fake `Approval` decision reporter** (stub Argo
Workflow) until B19-core lands; a **two-kind cluster fixture** (one kind labelled dev, one staging)
standing in for AWS; a **stub B4 Composition** presenting the canonical connection-secret shape so
AC-08 can be exercised before AWS Compositions exist.

## 6. PR / Branch Mapping

### 6.1 Stack position — base branch = `wave/auth` (ADRs are authoring-parallel context)
ADR-0040 is a constraint piece (`wave=auth`); no build wave. Its spec+plan land on the
authoring-parallel base so all enforcer specs (A7, B4, B19, A23, B5, A18, B9) bind to it.

### 6.2 PR — `piece/ADR-0040-kargo-promotion-fabric` → base `wave/auth`; carries spec + plan for this piece
Single PR with `spec-ADR-0040.md` + `plan-ADR-0040.md`. No code artifacts (enforcement lives in the
component PRs).

### 6.3 Merge order — within-wave siblings independent; wave rolls up to main
Independent of other `auth` siblings; merges as soon as reviewed. Real-build enforcement then threads
through the component waves: A7/B4 (W0–W1) → B19-core (W3) → **A23 + B5 + B19-ui (W4)**, where
conformance is demonstrated end-to-end.

## 7. Effort Estimate

Per-task: TASK-01..11 each S (conformance checks); TASK-12 S (the matrix). **Rollup: S** (matches
piece-index; the real cost lives in A23, an L component). **Critical path within the piece:**
TASK-02 → TASK-03 → {TASK-04/05/09/10} → TASK-12.

## 8. Rollback / Reversibility

Backing out the spec+plan PR removes the constraint document; it touches no running system. The real
reversibility is at A23: because recovery is **re-promote a previous Warehouse commit** (REQ-09) and
git history is append-only, reverting a bad release is a forward promotion, not a git-revert. If the
ADR were withdrawn, A23/B5/B19-ui lose their binding contract and would need a replacement promotion
model; A18 audit-emission and B9 two-kind-test expectations would also need rework. Reverting the ADR
document itself implies no data migration.
