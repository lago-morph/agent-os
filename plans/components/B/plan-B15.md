# PLAN B15 — CI/CD reference pipeline (GitHub Actions only, v1.0)

> spec: SPEC-B15 · kind: COMPONENT · tier: T1
> wave: W4 · estimate: M
> upstream-pieces: [B9, B14] · downstream-pieces: []

## 1. Implementation Strategy
Build **one hardened GitHub Actions reference pipeline** that is **thin native syntax around the
`agent-platform` CLI** (B9) and the `agent-platform test` orchestration (B14) — the deliberate
design (ADR 0010) that makes a second CI mechanical later. The pipeline runs **pre-merge static
checks** (`kubeval`, `conftest`, `helm`/`kustomize` render, ArgoCD server-side dry-run) and **gated
scan steps** (Trivy/Grype, dependency scan, OPA bundle lint, CRD+CloudEvent schema validation),
**complementing** Kargo's post-merge promotion gates (A23/ADR 0040) rather than duplicating them.
Ship the **security-first posture from the start** (OIDC to AWS, SHA-pinned actions, split build/
deploy runners, GitHub Environments, signed artifacts) and the **scheduled container-maintenance**
pipeline (rebuild+scan+regression-eval+`Agent`-image-bump PR). Publish the pipeline as a doc-portal
how-to and document the env-var/secret + network-endpoint schema.

## 2. Ordered Task List
- **TASK-01:** Freeze the CI **env-var/secret schema** + **network-endpoint list** (Langfuse, Git,
  registry, ArgoCD, OPA bundle store, OpenSearch) with B9. — produces: contract doc — depends-on: []
- **TASK-02:** Author the core PR/push/tag/dispatch workflow calling CLI subcommands
  (`validate`/`package`/`test`/`scan`/`deploy-preview`/`promote`) with PR-comment + commit-status
  posting. — produces: core workflow — depends-on: [TASK-01]
- **TASK-03:** Add **pre-merge static checks** (`kubeval`, `conftest`, render, ArgoCD dry-run) as
  required checks. — produces: static-check jobs — depends-on: [TASK-02]
- **TASK-04:** Add **gated scan steps** (Trivy/Grype, dependency scan, OPA bundle lint, CRD+
  CloudEvent schema validation). — produces: scan jobs — depends-on: [TASK-02]
- **TASK-05:** Harden: OIDC→AWS (no static creds), SHA-pin actions, separate build/deploy runners
  (deploy=elevated), GitHub Environments gating prod jobs, signed artifacts. — produces: hardening —
  depends-on: [TASK-02]
- **TASK-06:** Wire the test stage to `agent-platform test` (B14) incl. stress-probe invocation. —
  produces: test stage — depends-on: [TASK-02]
- **TASK-07:** Build the **scheduled container-maintenance** workflow (rebuild+scan+regression eval+
  tag+`Agent`-image-bump PR; out-of-cycle CVE trigger). — produces: maintenance workflow —
  depends-on: [TASK-04, TASK-05]
- **TASK-08:** Define the **Kargo `promote` seam** (hand off across-Stage progression to A23; do not
  duplicate Stage verification). — produces: promote seam — depends-on: [TASK-02]
- **TASK-09:** Publish the reference-pipeline how-to + runbook; self-tests. — produces: docs + tests —
  depends-on: [TASK-03, TASK-04, TASK-05, TASK-06, TASK-07, TASK-08]

## 3. Dependency Map
### 3.1 Upstream pieces that must ship first (HARD)
- **B9 (`agent-platform` CLI)** — the subcommand contract + env-var/secret schema the pipeline calls.
- **B14 (test framework)** — `agent-platform test` + stress-probe harness the test stage invokes.

### 3.2 Downstream pieces blocked on this
- None (`piece-index.csv` lists no downstream). Consumed by platform **adopters** at runtime; outputs
  feed ArgoCD/Kargo.

### 3.3 Continuous (non-blocking) inputs
- **A23 (Kargo)** — the post-merge promotion fabric B15 complements (W4 co-land; `promote` seam).
- **B3/B16** — OPA Rego library/content that `conftest` + bundle-lint run against.
- **B12** — CloudEvent schemas validated by the schema-validation gate.
- **B22** — security standards the security-first pipeline design honors.

## 4. Parallelizable Subtasks
- After TASK-02: TASK-03, TASK-04, TASK-05, TASK-06, TASK-08 are independent fan-out groups.
- TASK-07 depends on TASK-04+05; TASK-09 gathers all.

## 5. Test Strategy
- **PyTest / workflow-lint:** unpinned-action lint fails (AC-B15-08); env/endpoint reachability matches
  the documented schema (AC-B15-11); the pipeline drives CLI subcommands not bespoke runners
  (AC-B15-01,13).
- **Playwright (HTTP/CI dry-run):** PR-comment + commit-status posting (AC-B15-03); required scan/
  check gates block on bad input — critical CVE, vulnerable dep, malformed bundle, invalid schema
  (AC-B15-05); pre-merge checks catch manifest/policy/render/dry-run errors (AC-B15-04).
- **Chainsaw:** ArgoCD server-side dry-run path + the container-maintenance `Agent`-image-bump PR
  flow (AC-B15-06).
- **Hardening checks:** OIDC (no static AWS keys), GitHub Environment gate on a prod job, split
  runners, signed artifacts (AC-B15-07,09,10).
- **Kargo seam:** pre-merge runs before merge; promotion delegated to Kargo, no Stage verification
  duplicated (AC-B15-12).
- **Fakes:** fake registry + fake ArgoCD dry-run endpoint; single-Stage Kargo stub for the promote seam.

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/4` (contains B9, B14 rolled up; A23 co-lands in W4).
### 6.2 PR — `piece/B15-cicd-reference-pipeline` → base `wave/4`; carries spec-B15 + plan-B15.
### 6.3 Merge order — after B9 (W2) and B14 (W3); coordinate the promote seam with A23 (W4 sibling).
wave/4 rolls up to main.

## 7. Effort Estimate
- TASK-01 S · TASK-02 M · TASK-03 S · TASK-04 M · TASK-05 M · TASK-06 S · TASK-07 M · TASK-08 S ·
  TASK-09 S.
- Rollup: **M** (matches CSV). Critical path: TASK-01 → 02 → 05 (hardening) → 07 → 09.

## 8. Rollback / Reversibility
B15 is workflow YAML + docs; rollback is reverting the workflow files. Backing it out removes the
reference CI path (adopters lose the worked example and the container-maintenance automation) but
touches no cluster state — ArgoCD/Kargo continue independently, and an in-flight `Agent`-image-bump
PR is just an open PR that can be closed. The reversibility risk is **operational, not data**: if the
pipeline held the only OIDC trust path to the registry/GitOps, reverting it severs automated deploys
until re-applied; mitigate by keeping the prior workflow revision and the OIDC trust config in Git.
