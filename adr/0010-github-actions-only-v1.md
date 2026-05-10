# ADR 0010: GitHub Actions only for v1.0 CI/CD

- **Status**: Accepted
- **Date**: 2026-05-10

## Context

The platform needs a CI/CD integration story for agent build, evaluation, red-team, packaging, deployment preview, promotion, and base-image maintenance. Real-world deployments use a mix of GitHub Actions, GitLab CI, Jenkins, and others, and a how-to guide that covers all three was originally contemplated. v1.0 ships against AWS (EKS) and GitHub as the initial implementation targets (see ADR 0033), so the marginal value of supporting additional CI systems on day one is low while the ongoing maintenance cost of multiple reference pipelines is high. Section 8 of the architecture overview also requires a security-first pipeline with scoped tokens, OIDC federation, pinned actions, separate build/deploy runners, and gated scan steps — hardening that has to be designed and maintained per CI system.

## Decision

v1.0 supports **GitHub Actions only** as the reference CI/CD system. Jenkins, GitLab CI, and other CI systems are explicitly out of scope for v1.0. The platform exposes integration through a single `agent-platform` CLI image whose subcommands (`validate`, `eval`, `redteam`, `package`, `test`, `deploy-preview`, `promote`, `scan`, `update-base`) are the contract; pipeline definitions are native syntax around CLI calls. Because CLI-based test orchestration (ADR 0011) does the real work, adding another CI system later is mechanical — author native pipeline syntax that calls the same CLI.

## Alternatives considered

- **Multi-CI from day one (GitHub Actions, GitLab CI, Jenkins reference pipelines)** — Triples the surface area for security hardening, schema-required env vars, secret conventions, and the security-first pipeline behaviors required by overview § 8 (scoped tokens, OIDC, pinned actions, gated scans). With AWS + GitHub as the v1.0 initial targets (ADR 0033) and the CLI as the integration contract, a second or third reference pipeline adds maintenance load without unblocking v1.0 use cases. Deferred until a deployment environment actually requires it.

## Consequences

- The reference pipeline ships as a security-first GitHub Actions pipeline per architecture-overview.md § 8: scoped tokens with OIDC federation to AWS, GitHub Environments gating production-affecting jobs, SHA-pinned third-party actions, separate build vs deploy runners, signed artifacts, and required scan steps (Trivy/Grype, dependency scanning, OPA bundle linting, CRD/CloudEvent schema validation) gated as required checks.
- The `agent-platform` CLI image is the integration contract; pipelines are syntax around it. This ties directly to ADR 0011 (three-layer testing with CLI orchestration) — the same CLI that drives local and harness execution drives CI, so a second CI system is mechanical pipeline translation, not new orchestration logic.
- Aligns with ADR 0033 (AWS + GitHub as the v1.0 initial implementation targets); Azure and other Git hosts are supported architecturally but not exercised in v1.0.
- The container maintenance pipeline (scheduled rebuild of the agent base image with scans, regression eval, and auto-PR to bump Agent CRD references) is implemented in GitHub Actions only for v1.0.
- Documentation publishes one reference pipeline (GitHub Actions) as a how-to guide rather than three; the previously contemplated GitLab CI and Jenkins how-tos are dropped from v1.0 scope.
- Adopters on Jenkins or GitLab CI in v1.0 must hand-write their own pipeline against the documented CLI, env-var schema, and network-endpoint list; the platform does not ship or support those pipelines.
- Revisit trigger (architecture-backlog.md § 3.14): a deployment environment requires Jenkins or GitLab CI. At that point the work is authoring a native pipeline against the existing CLI contract, not redesigning the integration surface.
- Pipeline hardening beyond v1.0 (admission-time enforcement of scan-artifact presence, continuous scanning of running images, image signing and attestation) remains in future-enhancements and is not in scope for this ADR.

## References

- architecture-overview.md § 8
- architecture-backlog.md § 2.17, § 3.14, § 7 item 10
- ADR 0011 (CLI-based test orchestration), ADR 0033 (AWS + GitHub initial targets)
