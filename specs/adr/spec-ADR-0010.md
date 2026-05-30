# SPEC ADR-0010 — GitHub Actions only for v1.0 CI/CD [PROPOSED]

> kind: ADR · workstream: — · tier: T2
> upstream: [B9] · downstream: [B14;B15] · adrs: [0010] · views: []
> canon-glossary: 6aadcc2a4f38 · canon-interface: 54f5ede58e5f

## 1. Purpose & Problem Statement
ADR 0010 fixes GitHub Actions as the only reference CI/CD system for v1.0. This SPEC states what honoring that decision requires: the `agent-platform` CLI image is the integration contract — its subcommands (`validate`, `eval`, `redteam`, `package`, `test`, `deploy-preview`, `promote`, `scan`, `update-base`) are what pipelines call; the one shipped reference pipeline is a security-first GitHub Actions pipeline (scoped tokens + OIDC, GitHub Environments gating, SHA-pinned actions, separate build/deploy runners, signed artifacts, required gated scans); and no Jenkins/GitLab CI reference pipelines ship in v1.0. The decision is settled; this SPEC captures the obligations it imposes and the acceptance criteria that prove it is honored.

## 2. Scope
### 2.1 In scope
- GitHub Actions as the only v1.0 reference CI/CD system (B15 reference pipeline).
- The `agent-platform` CLI image (B9) as the integration contract; pipelines are native syntax around CLI calls.
- Security-first pipeline behaviors per overview §8 (scoped tokens/OIDC, Environments gating, SHA-pinned actions, separate build/deploy runners, signed artifacts, required scan checks).
- The container-maintenance pipeline (scheduled base-image rebuild + scans + regression eval + auto-PR to bump Agent CRD refs) in GitHub Actions.

### 2.2 Out of scope (and where it lives instead)
- The CLI subcommand surface itself — component B9; three-layer CLI test orchestration — ADR 0011.
- Jenkins / GitLab CI reference pipelines — explicitly NOT in v1.0 (revisit backlog §3.14).
- AWS/GitHub initial-target selection — ADR 0033; pipeline hardening beyond v1.0 (admission-time scan-artifact enforcement, continuous image scanning, signing/attestation) — future-enhancements, not this ADR.
- Test framework + dashboards — B14 / D3.

## 3. Context & Dependencies
Upstream consumed: B9 (`agent-platform` CLI image — the integration contract). Downstream consumers: B14 (test framework) and B15 (CI/CD reference pipeline) build on the CLI contract.
ADR decisions honored: **0010** — GitHub Actions only, CLI-as-contract, security-first pipeline, no Jenkins/GitLab in v1.0; **0011** — the same CLI drives local/harness/CI orchestration; **0033** — AWS (EKS) + GitHub as the v1.0 initial targets; **0030** — CLI subcommand surface is the versioned API.

## 4. Interfaces & Contracts
### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
N/A — this ADR introduces no CRD/XRD. The base-image maintenance pipeline auto-PRs bumps to `Agent` CRD image references (the `Agent` CRD is owned by A5/ARK, not redefined here).

### 4.2 APIs / SDK surfaces
The `agent-platform` CLI (owner B9) is the versioned integration contract — subcommands `validate`, `eval`, `redteam`, `package`, `test`, `deploy-preview`, `promote`, `scan`, `update-base` (the subcommand surface is the versioned API per ADR 0030). Pipelines invoke the CLI image; they add no orchestration logic.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
N/A — the reference pipeline emits no platform CloudEvents directly. Evaluation/red-team results produced by CLI subcommands surface under `platform.evaluation.*` via their owning components, not via the CI system.

### 4.4 Data schemas / connection-secret contracts
N/A — no substrate primitive. CI authenticates to AWS via OIDC federation (scoped tokens), not a connection-secret XRD; secrets follow GitHub Environments + scoped-token conventions.

## 5. OSS-vs-Custom Decision
Reference pipeline built on **GitHub Actions** as native syntax around the custom `agent-platform` CLI image (B9/B15). Mode: build-new pipeline (thin) over the CLI contract. Rationale per ADR 0010: v1.0 ships against AWS (EKS) + GitHub (ADR 0033), so the marginal value of additional CI systems on day one is low while multi-CI security hardening is high-cost; CLI-based orchestration (ADR 0011) makes adding another CI later mechanical. Rejected: multi-CI from day one (triples hardening/secret/env-var surface for overview §8 without unblocking v1.0).

## 6. Functional Requirements
- REQ-ADR-0010-01: GitHub Actions MUST be the only reference CI/CD system shipped in v1.0; no Jenkins/GitLab CI reference pipelines ship.
- REQ-ADR-0010-02: The `agent-platform` CLI image MUST be the integration contract; pipelines MUST be native syntax around CLI subcommand calls with no separate orchestration logic.
- REQ-ADR-0010-03: The reference pipeline MUST be security-first per overview §8: scoped tokens + OIDC to AWS, GitHub Environments gating production-affecting jobs, SHA-pinned third-party actions, separate build vs deploy runners, signed artifacts.
- REQ-ADR-0010-04: Required scan steps (Trivy/Grype, dependency scanning, OPA bundle linting, CRD/CloudEvent schema validation) MUST be gated as required checks.
- REQ-ADR-0010-05: The container-maintenance pipeline (scheduled base-image rebuild + scans + regression eval + auto-PR to bump Agent CRD refs) MUST be implemented in GitHub Actions for v1.0.
- REQ-ADR-0010-06: Adopters on other CI systems MUST be able to hand-write pipelines against the documented CLI/env-var schema/endpoint list (unsupported in v1.0); adding a CI system later MUST be pipeline translation, not new orchestration.

## 7. Non-Functional Requirements
- Security: scoped tokens, OIDC federation, SHA-pinned actions, separate build/deploy runners, signed artifacts, gated scans (overview §8).
- Versioning: CLI subcommand surface is the versioned API (ADR 0030); pipelines pin the CLI image.
- Documentation: one reference pipeline published as a how-to (GitHub Actions); GitLab/Jenkins how-tos dropped from v1.0.
- Future scope: admission-time scan-artifact enforcement, continuous image scanning, signing/attestation deferred to future-enhancements.

## 8. Cross-Cutting Deliverable Checklist
N/A — ADR. The §8/§14.2 deliverable set is owned by B15 (reference pipeline) + B9 (CLI); conformance items appear in §9 / the PLAN.

## 9. Acceptance Criteria
Decision honored when:
- AC-ADR-0010-01: The repository ships exactly one reference CI pipeline (GitHub Actions); no Jenkins/GitLab reference pipeline is present. (REQ-01)
- AC-ADR-0010-02: Every pipeline job's substantive step is a call to an `agent-platform` CLI subcommand; the pipeline contains no orchestration logic beyond CLI invocation. (REQ-02)
- AC-ADR-0010-03: The pipeline authenticates to AWS via OIDC scoped tokens, gates production jobs behind GitHub Environments, SHA-pins all third-party actions, and uses separate build/deploy runners. (REQ-03)
- AC-ADR-0010-04: Trivy/Grype, dependency scan, OPA bundle lint, and CRD/CloudEvent schema validation are configured as required checks that block merge on failure. (REQ-04)
- AC-ADR-0010-05: The scheduled base-image maintenance pipeline runs in GitHub Actions and opens an auto-PR bumping Agent CRD image references. (REQ-05)
- AC-ADR-0010-06: A documented CLI/env-var/endpoint contract exists sufficient for an adopter to author a non-GitHub pipeline (unsupported). (REQ-06)

## 10. Risks & Open Questions
- Single-CI lock-in for v1.0 (blast radius: low) — CLI-as-contract makes a second CI mechanical; revisit trigger is a deployment env requiring Jenkins/GitLab (backlog §3.14).
- Adopters on other CI hand-roll their own pipeline (low) — unsupported in v1.0 by design; documented contract provided.
- Pipeline hardening beyond v1.0 deferred (low) — future-enhancements; not this ADR.

## 11. References
- ADR 0010 (`adr/0010-github-actions-only-v1.md`) — the decision.
- Enforcing components: B15 (CI/CD reference pipeline — GitHub Actions, owner), B9 (`agent-platform` CLI contract), B14 (test framework), A5 (Agent CRD refs bumped by maintenance pipeline).
- architecture-overview.md §8, §14.2; architecture-backlog.md §2.17, §3.14, §7.
- Related: ADR 0011, 0033, 0030.
