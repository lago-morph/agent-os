# SPEC B15 — CI/CD reference pipeline (GitHub Actions only, v1.0)

> kind: COMPONENT · workstream: B · tier: T1
> upstream: [B9, B14] · downstream: [] · adrs: [0010, 0033, 0011, 0040, 0002, 0034, 0030, 0031]
> · views: [6.6]
> canon-glossary: see _meta/glossary.md · canon-interface: see _meta/interface-contract.md

## 1. Purpose & Problem Statement

B15 is the **reference CI/CD pipeline** the platform ships — **GitHub Actions only for v1.0**
(ADR 0010; §8). It is the worked example of "option 2": **native pipeline syntax around the
`agent-platform` CLI**. The CLI (B9) is the integration contract; B15's GitHub Actions workflows
are syntax that calls `validate`, `eval`, `redteam`, `package`, `test`, `deploy-preview`,
`promote`, `scan`, `update-base`. Because the real work lives in the CLI (and the test
orchestration in B14, ADR 0011), adding Jenkins/GitLab CI later is **mechanical** — author native
pipeline syntax against the same CLI — which is exactly why v1.0 scopes to one reference pipeline.

The problem it solves: a privileged CI/CD pipeline that has registry push and GitOps-merge-trigger
access must be **security-first by construction** (§8) — scoped tokens / OIDC federation, SHA-pinned
actions, separate build/deploy runners, signed artifacts, and gated scan steps — and that hardening
has to be designed and maintained per CI system (ADR 0010's whole rationale). B15 ships that one
hardened reference, plus the **container-maintenance pipeline** that keeps the agent base image
current, and the **pre-merge static checks** that complement Kargo's runtime promotion gates
(ADR 0040).

## 2. Scope

### 2.1 In scope
- The **reference GitHub Actions pipeline** (published in the doc portal as a how-to guide, §8),
  calling the `agent-platform` CLI subcommands as the contract.
- **Trigger coverage**: PR, push, tag, schedule, manual dispatch (§8 "What CI is responsible for").
- **Result posting**: PR comments + commit statuses (§8).
- **Pre-merge static checks** complementing Kargo (§8, ADR 0040): `kubeval` on manifests,
  `conftest` on the OPA Rego library, `helm template` / `kustomize build` render verification,
  ArgoCD **server-side dry-run** before merge.
- **Required scan steps** gated as required checks (§8): **Trivy / Grype** on images, dependency
  scanning on language packages, **OPA bundle linting**, schema validation on CRDs + CloudEvents.
- The **container-maintenance pipeline** (scheduled): rebuild base image with latest base+OS+deps,
  run scans (block on critical CVEs), run **regression eval suite**, tag new image, **open a PR to
  bump Agent CRD image references**; triggered out-of-cycle for critical CVEs.
- **Security-first pipeline design**: no long-lived static creds (OIDC federation to AWS where
  possible), **GitHub Environments** gating production-affecting jobs, SHA-pinned third-party
  actions, separate build-vs-deploy runners (deploy runner = elevated trust), signed artifacts.
- The **env-var/secret schema** and **network-endpoint list** CI runners need (§8: Langfuse, Git,
  container registry, ArgoCD, OPA bundle store, OpenSearch).

### 2.2 Out of scope (and where it lives instead)
- **The `agent-platform` CLI** (subcommands, packaging, versioning) → **B9**.
- **Test orchestration / stress probes / result emission / `--debug`-`--trace`** → **B14** (ADR 0011);
  B15 *calls* `agent-platform test`.
- **Kargo promotion fabric** (Warehouse, Stages, per-Stage verification, `Approval`-gated promotion,
  rollback-via-promotion) → **A23** (ADR 0040). B15 is **pre-merge**; Kargo runs **after** merge into
  the Warehouse. The two **complement**, not overlap (§8).
- **Jenkins / GitLab CI reference pipelines** → **out of v1.0 scope** (ADR 0010); adopters hand-write
  against the documented CLI/env-var/endpoint contract.
- **The OPA Rego library + bundles** that `conftest`/bundle-lint run against → **B3 / B16**; B15 runs
  the checks, it doesn't author policy.
- **CRD / CloudEvent schemas** validated → owned by their components / **B12**; B15 validates against them.
- **Pipeline hardening beyond v1.0** (admission-time enforcement of scan-artifact presence,
  continuous scanning of running images, image signing+attestation) → **future-enhancements** (§8).
- **Azure / non-GitHub targets** → deferred (ADR 0033); v1.0 targets **AWS (EKS) + GitHub**.

## 3. Context & Dependencies

**Upstream consumed (HARD):**
- **B9 (`agent-platform` CLI)** — the integration contract; B15 workflows invoke its subcommands as
  a container image. B15 binds to the CLI's documented subcommand surface + env-var/secret schema.
- **B14 (test framework)** — B15's test stage calls `agent-platform test` (three layers) + the
  stress-probe harness; B14 is the orchestration B15 wraps in native GitHub Actions syntax.

**Downstream consumers:** none in `piece-index.csv`. (The pipeline is consumed by *platform
adopters* at runtime, and its outputs feed ArgoCD/Kargo, but no build piece depends on B15.)

**Relationship to Kargo (A23, ADR 0040):** B15 handles **within-environment** pre-merge CI;
**Kargo** handles **environment-to-environment** promotion after merge. v1.0 Kargo starts with a
**single Stage** (glossary; §14). B15's `promote` subcommand call is the seam to Kargo.

**ADRs honored:**
- **ADR 0010** — GitHub Actions only; CLI is the contract; security-first reference pipeline;
  container-maintenance pipeline in GitHub Actions; one how-to (not three).
- **ADR 0033** — AWS (EKS) + GitHub as v1.0 targets; OIDC federation to AWS; kind for dev/test; dual-
  mode primitives consumed via Crossplane claims (tests run against both modes).
- **ADR 0011** — CLI-based test orchestration is what makes a second CI mechanical.
- **ADR 0040** — pre-merge checks complement Kargo's runtime promotion gates; rollback is
  promote-previous-Warehouse-commit, not `git revert` (§7.4).
- **ADR 0002** — OPA bundle linting / `conftest` as required checks; SHA-pinned bundles discipline.
- **ADR 0034** — pipeline-driven test/eval runs emit audit through the adapter (via the CLI).
- **ADR 0030 / 0031** — schema validation on CRDs + CloudEvents as gated checks; pipeline references
  versioned CLI.

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
**N/A — B15 defines no CRD/XRD.** It is GitHub Actions workflow definitions + the CLI contract. It
*validates against* CRD/XRD/CloudEvent schemas (a gated check) but owns none. The
container-maintenance pipeline **opens a PR to bump `Agent` image references** (`Agent.image`,
interface-contract §1.2) — it edits, not defines, that field.

### 4.2 APIs / SDK surfaces
- **`agent-platform` CLI subcommands** (the contract, ADR 0010): `validate`, `eval`, `redteam`,
  `package`, `test`, `deploy-preview`, `promote`, `scan`, `update-base` — **all nine are
  source-named** (§8, ADR 0010). B15 calls these; it does not invent new subcommands.
- **Env-var / secret schema** + **network-endpoint list** — source names the *endpoints* (Langfuse,
  Git, container registry, ArgoCD, OPA bundle store, OpenSearch); the **concrete env-var/secret
  names are `[PROPOSED — not in source]`** and are co-owned with B9.
- **GitHub Actions surface** — workflows, `permissions:` blocks, GitHub Environments, OIDC
  `id-token` federation to AWS. Specific job/workflow names are `[PROPOSED — not in source]`.
- **Versioning (ADR 0030):** B15 pins the CLI image version and SHA-pins third-party actions; the
  pipeline binds to the CLI's semantic version.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
- B15 itself **emits no CloudEvents directly** — it is workflow YAML. Audit of test/eval runs it
  triggers flows through the **CLI → audit adapter** under `platform.audit.*` (ADR 0034), emitted by
  B14/B9, not by the workflow.
- **Consumes**: GitHub webhook/trigger events (native GitHub, not platform CloudEvents) — PR, push,
  tag, schedule, dispatch.
- Promotion events under `platform.approval.*` / Kargo Stage events are **A23/B19's** emission, not
  B15's. `[PROPOSED — not in source]` if the pipeline surfaces any `platform.*` event of its own.

### 4.4 Data schemas / connection-secret contracts
- **No platform connection-secret of its own.** B15 reaches AWS via **OIDC federation** (no
  long-lived static credentials, §8/ADR 0033) and stores any required secrets as **encrypted CI
  secrets** (CI responsibility, §8). The deploy runner holds elevated trust; build runner does not.
- **Signed artifacts** (§8) — image/artifact signatures produced in the pipeline; the signing scheme
  detail is `[PROPOSED — not in source]` (image signing + attestation hardening is explicitly a
  *future-enhancement*, so v1.0 "signed artifacts" is the baseline, not full attestation).
- **Scan artifacts** (Trivy/Grype reports, dependency-scan, bundle-lint, schema-validation results)
  are produced as gated required checks; their storage/format is `[PROPOSED — not in source]`.

## 5. OSS-vs-Custom Decision
**Custom reference workflows over a hosted CI + OSS tools.** Platform: **GitHub Actions** (hosted;
ADR 0010/0033). Approach: **build-new** reference workflows that are *thin syntax around the CLI*
(B9) — the deliberate design so multi-CI is later mechanical. Scan/check tools are OSS and **wrapped**
as steps: **Trivy or Grype** (image scan — source offers the choice), `kubeval`, `conftest`,
`helm`/`kustomize`, ArgoCD dry-run, dependency scanners. No versions pinned in source (`[PROPOSED —
not in source]` to pin + SHA-pin actions). Rationale (ADR 0010): one hardened reference pipeline; the
CLI is the contract so the pipeline is intentionally minimal logic.

## 6. Functional Requirements
- **REQ-B15-01:** B15 MUST ship a **GitHub Actions** reference pipeline (published as a doc-portal
  how-to) that drives all work through `agent-platform` CLI subcommands — **GitHub Actions only**
  (ADR 0010). No Jenkins/GitLab pipeline is shipped.
- **REQ-B15-02:** The pipeline MUST support triggers on **PR, push, tag, schedule, and manual
  dispatch** (§8).
- **REQ-B15-03:** The pipeline MUST **post results as PR comments and commit statuses** (§8).
- **REQ-B15-04:** The pipeline MUST run **pre-merge static checks**: `kubeval` (manifests),
  `conftest` (OPA Rego), `helm template`/`kustomize build` (render), and **ArgoCD server-side
  dry-run** (§8, ADR 0040).
- **REQ-B15-05:** The pipeline MUST run **required scan steps**: Trivy **or** Grype on images,
  dependency scanning, OPA bundle linting, and CRD + CloudEvent schema validation — **gated as
  required checks** (§8).
- **REQ-B15-06:** A **scheduled container-maintenance** workflow MUST rebuild the agent base image
  (latest base+OS+deps), scan and **block on critical CVEs**, run the **regression eval suite**, tag
  the new image, and **open a PR to bump `Agent` image references**; it MUST also be triggerable
  **out-of-cycle for critical CVEs** (§8).
- **REQ-B15-07:** The pipeline MUST use **scoped tokens with OIDC federation to AWS** (no long-lived
  static credentials) and **GitHub Environments** to gate production-affecting jobs (§8, ADR 0033).
- **REQ-B15-08:** Third-party actions MUST be **SHA-pinned**; the CLI image and tool versions pinned
  (§8).
- **REQ-B15-09:** Build and deploy MUST run on **separate runners**, with the deploy runner holding
  **elevated trust** (§8).
- **REQ-B15-10:** The pipeline MUST produce **signed artifacts** (§8 baseline; full image
  signing+attestation is a future-enhancement, out of v1.0 scope).
- **REQ-B15-11:** B15 MUST document the **env-var/secret schema** and **network-endpoint list**
  (Langfuse, Git, container registry, ArgoCD, OPA bundle store, OpenSearch) CI runners require (§8).
- **REQ-B15-12:** Pre-merge checks MUST **complement, not duplicate, Kargo** — promotion across
  Stages (verification suites, `Approval`-gated human gates, rollback-via-promotion) is A23/ADR 0040,
  invoked via the `promote` subcommand seam (§8, §7.4).
- **REQ-B15-13:** Test runs invoked by the pipeline MUST go through `agent-platform test` (B14/ADR
  0011), so the same orchestration runs in CI as on a laptop/schedule.

## 7. Non-Functional Requirements
- **Security (primary, §8):** the pipeline is a privileged component (registry push + GitOps-merge
  trigger) and is hardened from the start — OIDC over static creds, SHA-pinned actions, split
  build/deploy trust, GitHub Environments gating prod jobs, gated scans, signed artifacts.
- **Multi-tenancy (§6.9):** production-affecting jobs are gated by GitHub Environments; deploy-runner
  elevation is the trust boundary. v1.0 single-Stage Kargo (glossary) bounds environment progression.
- **Observability (§6.5):** CLI-driven test/eval runs emit audit (ADR 0034) and stream results to
  OpenSearch/OTel (via B14); the pipeline posts PR/commit status for per-run detail.
- **Scale / maintainability:** one reference pipeline (not three) per ADR 0010; pipeline logic is
  thin syntax around the CLI so a second CI is mechanical translation.
- **Versioning (ADR 0030):** pins the CLI image + tool/action SHAs; binds to the CLI's semver.

## 8. Cross-Cutting Deliverable Checklist
- Helm/manifests — **N/A (workflow YAML, not a deployed workload)** — B15 ships GitHub Actions
  workflow definitions, not a cluster workload.
- Per-product docs (10.5) — **applicable** — the reference pipeline **is** a doc-portal how-to (§8);
  plus env-var/secret + endpoint reference.
- Runbook (10.7) — **applicable** (broken pipeline, failed scan gate, CVE-block override, rollback
  via Kargo promotion).
- Alerts — **applicable** (scheduled maintenance pipeline failure, scan-gate failure trend).
  `[PROPOSED — not in source]`.
- Grafana dashboard (Crossplane XR) — **N/A** — pipeline run telemetry is GitHub-native + (for test
  runs) B14/D3's OpenSearch+OTel views; B15 adds no dashboard of its own.
- Headlamp plugin — **N/A** — no admin UI; the Kargo plugin is B5; GitHub is the CI surface.
- OPA/Rego integration — **applicable** (`conftest` + OPA bundle linting as required checks; content
  from B3/B16).
- Audit emission (ADR 0034) — **applicable (indirect)** — via the CLI/B14 for test/eval runs; the
  workflow YAML itself emits none.
- Knative trigger flow — **N/A** — GitHub webhooks/triggers, not platform CloudEvent trigger flows.
- HolmesGPT toolset — **N/A** — no toolset contributed.
- 3-layer tests (Chainsaw/Playwright/PyTest) — **applicable** — B15's own correctness (the pipeline
  invokes the right CLI calls, gates fail on bad input) is tested; e.g. PyTest/Playwright over a
  pipeline dry-run, Chainsaw over the ArgoCD dry-run + `Agent`-bump PR flow.
- Tutorials & how-tos — **applicable** — the reference pipeline how-to is a core deliverable (§8).

## 9. Acceptance Criteria
- **AC-B15-01:** The repo ships one GitHub Actions reference pipeline (and **no** Jenkins/GitLab
  pipeline) that drives work via `agent-platform` subcommands. (→ REQ-B15-01)
- **AC-B15-02:** The pipeline fires on PR, push, tag, schedule, and manual dispatch. (→ REQ-B15-02)
- **AC-B15-03:** A run posts a PR comment and sets a commit status. (→ REQ-B15-03)
- **AC-B15-04:** A manifest/policy/render error is caught pre-merge by `kubeval` / `conftest` /
  `helm`-`kustomize` / ArgoCD dry-run respectively. (→ REQ-B15-04)
- **AC-B15-05:** An image with a critical CVE, a vulnerable dependency, a malformed OPA bundle, and an
  invalid CRD/CloudEvent schema each fail their **required** check and block merge. (→ REQ-B15-05)
- **AC-B15-06:** The scheduled maintenance workflow rebuilds + scans + runs the regression eval +
  tags + opens an `Agent`-image-bump PR; a critical CVE blocks the new image and can trigger an
  out-of-cycle run. (→ REQ-B15-06)
- **AC-B15-07:** The pipeline authenticates to AWS via OIDC (no static AWS keys in secrets) and a
  production-affecting job is gated by a GitHub Environment. (→ REQ-B15-07)
- **AC-B15-08:** All third-party actions are SHA-pinned; an unpinned action fails a lint gate.
  (→ REQ-B15-08)
- **AC-B15-09:** Build and deploy run on separate runners; the deploy runner alone holds the elevated
  GitOps-trigger credential. (→ REQ-B15-09)
- **AC-B15-10:** The pipeline emits signed artifacts. (→ REQ-B15-10)
- **AC-B15-11:** The env-var/secret schema + network-endpoint list are documented and a runner
  reaches exactly that endpoint set. (→ REQ-B15-11)
- **AC-B15-12:** Pre-merge checks run before merge; promotion across Stages is delegated to Kargo via
  `promote` (no Stage verification duplicated in B15). (→ REQ-B15-12)
- **AC-B15-13:** The pipeline's test stage invokes `agent-platform test`, not a bespoke runner.
  (→ REQ-B15-13)

## 10. Risks & Open Questions
- **R1 (high):** B15 is a **privileged component** (registry push + GitOps merge trigger). A
  misconfigured `permissions:` block or leaked deploy credential is high blast radius. Mitigation:
  OIDC, split runners, GitHub Environments, SHA-pins (REQ-B15-07..09). Blast radius: high.
- **R2 (med):** **Env-var/secret names** are `[PROPOSED — not in source]` and co-owned with B9; drift
  between the CLI's expected env and the pipeline's provided env breaks runs silently. Mitigation:
  freeze the schema with B9 (REQ-B15-11).
- **R3 (med):** **Trivy vs Grype** is an unresolved choice in source ("Trivy or Grype"); picking one
  vs. supporting both affects the gate. `[PROPOSED]` pick one for the reference, document swapping.
- **R4 (med):** **Kargo seam** — exact handoff from B15 `promote` to A23 Stages is `[PROPOSED — not in
  source]` beyond "complements Kargo"; with v1.0 single-Stage Kargo the seam is thin but must not
  duplicate Stage verification. Blast radius: med (A23 coupling).
- **R5 (low/med):** "**Signed artifacts**" baseline vs. full **image signing + attestation** (a named
  future-enhancement) — scope must stay at baseline signing for v1.0 to avoid scope creep.
- **OQ1:** Does the container-maintenance regression eval reuse `agent-platform eval`/`redteam` +
  promptfoo, or a separate suite? `[PROPOSED]` reuse the CLI eval subcommands.
- **OQ2:** Is the kind path exercised in CI for integration (dual-mode tests, ADR 0033) on GitHub-
  hosted runners, or only self-hosted? `[PROPOSED — not in source]`; affects runner topology.

## 11. References
- architecture-overview.md §8 (line 1364; GitHub-Actions-only, option-2 CLI contract, nine
  subcommands, trigger/result responsibilities, container-maintenance pipeline, pre-merge checks
  complementing Kargo, security-first design, scan steps), §7.4 (line 1219; PR+ArgoCD within-env,
  Kargo across-env, rollback-via-promotion), §6.6 (line 436; OPA/policy hooks), §9 (OSS fill-ins),
  §14 (single-Stage Kargo phasing).
- ADR 0010 (GitHub Actions only; CLI contract; one reference pipeline; container-maintenance in GHA),
  ADR 0033 (AWS+GitHub targets; OIDC; dual-mode; kind dev/test), ADR 0011 (CLI test orchestration),
  ADR 0040 (Kargo promotion; complement pre-merge checks; rollback-via-promotion), ADR 0002 (OPA
  bundle lint/conftest), ADR 0034 (audit via adapter), ADR 0030/0031 (versioning/schema validation).
- interface-contract §3.3 (`agent-platform` CLI versioned surface; HTTP/path versioning), §1.2
  (`Agent.image`), §2 (taxonomy). Glossary: `agent-platform`, Kargo, Warehouse, Stage.
- Related pieces: B9 (CLI), B14 (test framework), A23 (Kargo), B3/B16 (OPA content), B12 (schemas).
