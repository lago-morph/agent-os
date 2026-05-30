# PLAN B3 — OPA policy library (framework)

> spec: SPEC-B3 · kind: COMPONENT · tier: T0
> wave: W1 · estimate: M
> upstream-pieces: [A7] · downstream-pieces: [B16]

## 1. Implementation Strategy
B3 builds the *conventions, shared Rego library, and tooling* that turn OPA (installed by A7)
into a contributable platform policy library, leaving all restriction content to B16. The
approach: (1) freeze the canonical decision input/output document contract first, because every
downstream decision point (B2, A20, B16, B19, Envoy egress, Headlamp gates, CLI) binds to it and
a late change is a cross-component break; (2) ship the shared helper packages (claim extraction,
RBAC-floor predicate, decision constructor, deny aggregation, dry-run wrapper) with their own
Rego unit tests so they are reusable from day one; (3) define the monorepo bundle layout, package
roots per decision point, `.manifest` revision convention, SHA-pin build tooling, and the
ArgoCD-reconciled delivery layout; (4) write the how-to-add-a-policy guide plus the CI gate set
(Rego unit tests, SHA-pin verification, RBAC-floor lint) so B16 and every contributor land
without framework edits. Build-new conventions wrapping OSS OPA; never fork OPA.

## 2. Ordered Task List
- **TASK-01:** Freeze the canonical decision **input-document** schema (subject claims, action,
  target, context, `dry_run`) and **decision-document** shape (`allow`, `reasons[]`, `layer`,
  `simulated`) — produces: decision-contract reference doc — depends-on: [].
- **TASK-02:** Implement the shared Rego library packages: Platform JWT claim extraction,
  RBAC-floor predicate, decision constructor, deny aggregation, dry-run wrapper — produces: shared
  library Rego + unit tests — depends-on: [TASK-01].
- **TASK-03:** Define the bundle directory layout, package namespacing per §6.6 decision point,
  `data`-document namespacing convention, and `.manifest` roots/revision — produces: bundle layout
  spec + skeleton — depends-on: [TASK-01].
- **TASK-04:** Build the bundle build + SHA-pin tooling and the ArgoCD-reconciled delivery layout
  (Helm/manifests + Application) — produces: build tooling + delivery manifests — depends-on:
  [TASK-03].
- **TASK-05:** Implement the CI gate set: Rego unit tests, SHA-pin verification, RBAC-floor lint
  (flag any allow-rule not gated by the floor predicate) — produces: CI workflow — depends-on:
  [TASK-02, TASK-04].
- **TASK-06:** Write the how-to-add-a-policy contribution guide + decision-contract reference +
  rollback/pin-mismatch runbook — produces: per-product docs + runbook — depends-on: [TASK-03,
  TASK-05].
- **TASK-07:** Author the decision-point → bundle-package coverage map and a coverage check —
  produces: coverage map + check — depends-on: [TASK-03].

## 3. Dependency Map
### 3.1 Upstream pieces that must ship first (HARD)
- **A7 (OPA / Gatekeeper)** — the OPA engine, bundle-loading mechanism, and Gatekeeper
  constraint-template model the framework's bundles are served into.

### 3.2 Downstream pieces blocked on this
- **B16** (initial OPA policy content). Indirectly: B2, A20, B19, A22, B9 bind to the decision
  contract.

### 3.3 Continuous (non-blocking) inputs
- **B22** threat model → policy-library targets feed B16, not B3 directly.
- **B14** test framework for the 3-layer harness.

## 4. Parallelizable Subtasks
After TASK-01: TASK-02 and TASK-03 run concurrently. TASK-04 follows TASK-03; TASK-07 runs
concurrently with TASK-04. TASK-05 joins TASK-02+TASK-04. TASK-06 is last.

## 5. Test Strategy
- **Chainsaw:** ArgoCD bundle reconcile — a `.manifest` revision bump propagates to running OPA
  (AC-B3-10).
- **PyTest:** build/SHA-pin tooling, coverage check, CI gate behavior (AC-B3-08, AC-B3-09,
  AC-B3-11).
- **Rego unit tests (`opa test`):** shared library helpers, decision-shape `simulated` field,
  claim extraction, dry-run no-side-effect, RBAC-floor grant rejection (AC-B3-02..06).
- **Fixtures/fakes:** a sample policy + tests for the how-to walkthrough (AC-B3-01, AC-B3-07);
  fake Platform JWT claim fixtures; a stub OPA bundle server if A7 not yet landed.

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/1` (contains A7 spec)
### 6.2 PR — `piece/B3-opa-policy-framework` → base `wave/1`; carries spec-B3 + plan-B3
### 6.3 Merge order — independent of B1/B2/B4/B13 W1 siblings; wave/1 rolls up to main; B16 (W2)
branches after merge.

## 7. Effort Estimate
TASK-01 M · TASK-02 M · TASK-03 M · TASK-04 M · TASK-05 S · TASK-06 S · TASK-07 S. Rollup: **M**.
Critical path: TASK-01 → TASK-03 → TASK-04 → TASK-05 → TASK-06.

## 8. Rollback / Reversibility
Framework is library + tooling + conventions, no running service. Back out by reverting the PR
and the ArgoCD Application; OPA falls back to A7's prior bundle. Downstream breakage: B16 cannot
build/place content, and decision-contract consumers (B2, A20, B19) lose the shared shape — high
blast radius, so the contract (TASK-01) must be merged before any consumer binds.
