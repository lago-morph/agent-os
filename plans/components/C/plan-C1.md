# PLAN C1 — Documentation portal infrastructure

> spec: SPEC-C1 · kind: COMPONENT · tier: T1
> wave: consumer (authoring-parallel; portal stands up before content lands) · estimate: M
> upstream-pieces: [] · downstream-pieces: [C2, C3, C4, C5, C6, C7, C8, C9]

## 1. Implementation Strategy

Stand up the **Material for MkDocs** portal (ADR 0008) as docs-as-code first, then layer the
Diataxis-aligned navigation skeleton, search, per-version builds, and the PR-based contribution
workflow around it. Build infrastructure-first so C2–C9 and the Workstream A/B per-product doc
authors have a stable, conventions-documented surface to plug into from day one. Author no
Diataxis content here — only the empty structure, the authoring conventions, and the GitHub
Actions docs check (ADR 0010 / 0033). Co-design the "clean Markdown" output contract with C8 so
re-indexing into the `platform-knowledge-base` `RAGStore` needs no portal-specific preprocessing.

## 2. Ordered Task List

- TASK-01: Author `mkdocs.yml`, theme/branding, docs-as-code repo layout — produces: MkDocs site config — depends-on: []
- TASK-02: Define the Diataxis navigation skeleton (4 quadrants + per-product/maintainer/runbook/docs-on-docs slots) — produces: nav structure + empty section scaffold — depends-on: [TASK-01]
- TASK-03: Configure in-portal search over the published corpus — produces: search config — depends-on: [TASK-01]
- TASK-04: Wire per-version builds (platform docs + pinned-vendor-doc slot) — produces: versioned-build config — depends-on: [TASK-01]
- TASK-05: Document MkDocs/Markdown authoring conventions + local-preview workflow — produces: conventions doc (referenced by C9) — depends-on: [TASK-02]
- TASK-06: Build the GitHub Actions docs check (build + link-check + Markdown lint, security-first per ADR 0010) — produces: CI workflow — depends-on: [TASK-01, TASK-05]
- TASK-07: Define + co-validate the clean-Markdown output contract for C8 — produces: corpus-shape contract — depends-on: [TASK-02]
- TASK-08: Playwright (nav/search e2e) + PyTest (build/link-lint) test suites — produces: test suites — depends-on: [TASK-03, TASK-06]

## 3. Dependency Map

### 3.1 Upstream pieces that must ship first (HARD)
- None — C1 is authoring-parallel and stands the portal up before content arrives.

### 3.2 Downstream pieces blocked on this
- C2, C3, C4, C5 (Diataxis content slots + conventions), C6 (runbook section), C7 (maintainer
  section), C8 (consumes the Markdown output contract), C9 (docs-on-docs section + conventions).

### 3.3 Continuous (non-blocking) inputs
- Every Workstream A/B component continuously contributes per-product docs (§10.5), runbooks
  (§10.7), maintainer docs (§10.6) into the slots C1 defines — recorded as continuous, not blocking.
- C8 indexing pipeline — co-owns the corpus-shape contract (TASK-07), iterated jointly.
- ADR 0024 vendor-doc companion project — feeds version pins into the reserved slot when it exists.

## 4. Parallelizable Subtasks

- After TASK-01: TASK-02, TASK-03, TASK-04 fan out concurrently.
- TASK-05 follows TASK-02; TASK-07 follows TASK-02 — both run in parallel with TASK-03/04.
- TASK-06 converges TASK-01+TASK-05; TASK-08 converges the rest.

## 5. Test Strategy

- Chainsaw: N/A — C1 defines no CRD.
- Playwright: AC-C1-02 (nav has all quadrants + reserved slots), AC-C1-03 (search returns correct
  page), AC-C1-04 (version selector serves two version builds), AC-C1-09 (vendor-doc slot renders
  placeholder). Fixtures: a small fixture corpus so nav/search are testable before real content lands.
- PyTest: AC-C1-01 (`mkdocs build` zero errors), AC-C1-06 (broken-link / build-error PR fails the
  check), AC-C1-08 (convention-following page builds without infra change), AC-C1-10 (build tagged
  to a documented version). AC-C1-07 verified jointly with C8 (fake indexer until C8 lands).

## 6. PR / Branch Mapping

### 6.1 Stack position — base branch = `wave/consumer` (authoring-parallel; no upstream specs required)
### 6.2 PR — `piece/C1-documentation-portal-infrastructure` → base `wave/consumer`; carries spec-C1 + plan-C1
### 6.3 Merge order — C1 lands first among Workstream C siblings (C2–C9 depend on its skeleton); rolls up to main

## 7. Effort Estimate

- TASK-01 S · TASK-02 M · TASK-03 S · TASK-04 M · TASK-05 M · TASK-06 M · TASK-07 S · TASK-08 M
- Rollup: M (configuration + conventions; no runtime service, no CRD).
- Critical path: TASK-01 → TASK-02 → TASK-05 → TASK-06 → TASK-08.

## 8. Rollback / Reversibility

Revert the portal config and CI workflow via GitOps; fully reversible — static site, no runtime
state, no CRD. If reverted, C2–C9 lose their authoring surface and the C8 pipeline loses its
Markdown source — so C1 must precede and remain in place for the entire Workstream C lifecycle.
