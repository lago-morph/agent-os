# PLAN ADR-0010 — GitHub Actions only for v1.0 CI/CD [PROPOSED]

> spec: SPEC-ADR-0010 · kind: ADR · tier: T2
> wave: authoring-parallel · estimate: S
> upstream-pieces: [B9] · downstream-pieces: [B14;B15]

## 1. Implementation Strategy
Enforcement/verification map. ADR 0010 is enforced by component B15 (the single security-first GitHub Actions reference pipeline built as native syntax around the B9 `agent-platform` CLI image), with the CLI-as-contract boundary held by B9 and three-layer orchestration by ADR 0011. Conformance is proven by: (a) a single-pipeline / no-Jenkins-no-GitLab repo check; (b) a "jobs only call the CLI" structural lint; (c) a security-posture audit (OIDC/scoped tokens, Environments gating, SHA-pinned actions, split runners, signed artifacts); (d) a required-scan-checks gate; (e) the scheduled base-image maintenance + auto-PR check. No new product build belongs to this ADR — it is a constraint map over B15/B9.

## 2. Ordered Task List
- TASK-01: Map each REQ to the enforcing component piece — produces: enforcement matrix — depends-on: []
- TASK-02: Specify single-pipeline / no-Jenkins-no-GitLab repo check + CLI-only-jobs structural lint — produces: CI lint (B15/B9) — depends-on: [TASK-01]
- TASK-03: Specify the security-posture audit (OIDC scoped tokens, Environments gating, SHA-pinned actions, split build/deploy runners, signed artifacts) — produces: pipeline audit (B15) — depends-on: [TASK-01]
- TASK-04: Specify required-scan-checks gate (Trivy/Grype, dep scan, OPA bundle lint, CRD/CloudEvent schema validation) — produces: required-check config (B15) — depends-on: [TASK-01]
- TASK-05: Specify the scheduled base-image maintenance + auto-PR (bump Agent CRD refs) check + documented non-GitHub CLI contract — produces: scheduled-workflow test + doc (B15/B9) — depends-on: [TASK-01]

## 3. Dependency Map
### 3.1 Upstream that must ship first (HARD)
- B9 — `agent-platform` CLI image (the integration contract).
### 3.2 Downstream blocked on this
- B14 (test framework), B15 (reference pipeline).
### 3.3 Continuous (non-blocking) inputs
- ADR 0011 CLI orchestration; ADR 0033 AWS+GitHub targets; A5 Agent CRD refs; B22 threat model.

## 4. Parallelizable Subtasks
TASK-02, TASK-03, TASK-04, TASK-05 fan out independently after TASK-01.

## 5. Test Strategy
- AC-01/02 → CI lint (repo pipeline inventory; CLI-only jobs).
- AC-03/04 → pipeline audit + required-check config (security posture; gated scans).
- AC-05 → scheduled-workflow test (base-image rebuild → auto-PR).
- AC-06 → doc-presence check (CLI/env-var/endpoint contract).
Fixtures: stub `agent-platform` CLI image with no-op subcommands until B9 lands; mock OIDC role + GitHub Environments.

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/authoring`
### 6.2 PR — `piece/ADR-0010-github-actions-only` → base `wave/authoring`; carries spec + plan.
### 6.3 Merge order — independent of sibling ADR PRs; rolls up to main.

## 7. Effort Estimate
TASK-01..05 each S. Rollup: S. Critical path: TASK-01 → TASK-03.

## 8. Rollback / Reversibility
Backing out means adopting multi-CI from day one and tripling the security-hardening surface. Because the CLI is the integration contract, adding a CI later is pipeline translation, not redesign — so reversibility is high. Downstream breakage limited to B15's pipeline definitions; B9/B14 are unaffected.
