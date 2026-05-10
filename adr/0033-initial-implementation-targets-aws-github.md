# ADR 0033: Initial implementation targets — AWS (EKS) and GitHub

- **Status**: Accepted
- **Date**: 2026-05-10

## Context

The architecture is written to run on EKS, AKS, or kind, and to integrate with any CI system through the `agent-platform` CLI rather than a hardcoded pipeline runtime (architecture-overview.md § 3). Section 6.11 likewise commits to OIDC-based federation that maps cleanly onto both AWS IRSA and Azure Workload Identity. Building and validating both cloud-side bootstraps in v1.0 would double the install scope (Workstream A) without a current consumer asking for Azure. We therefore need an explicit decision recording which targets v1.0 actually exercises versus which are kept as architecturally-supported paths.

## Decision

For v1.0 the platform targets **AWS (EKS)** as the cloud and **GitHub** (source control + GitHub Actions) as the CI/source surface. The architecture continues to support **Azure (AKS, Azure Workload Identity, Azure Key Vault)** as a first-class option, but no Azure-specific install path, secret-provider wiring, or pipeline reference is built, tested, or shipped in v1.0. Local development and test continue to use **kind**.

## Consequences

- Workstream A install scope is bounded to EKS + GitHub bootstraps for v1.0; Azure bootstrap (AKS provisioning, Workload Identity trust setup, Key Vault wiring for ESO) is deferred and tracked in architecture-backlog.md § 1.17.
- Aligns with ADR 0010: GitHub Actions-only CI is the matching CI decision; the reference pipeline shipped in v1.0 is GitHub-Actions-only and any Jenkins / GitLab CI work is deferred (architecture-backlog.md § 3.14).
- `kind` remains the supported developer and integration-test environment, so contributors do not need an AWS account to develop the platform.
- Cloud portability is protected at the architecture level by ADR 0003 (Envoy egress proxy): egress controls are not tied to a CNI or cloud-specific L7 policy, so the Azure path is not blocked by v1.0 implementation choices.
- Identity federation (ADR 0028) defines a single OIDC-based pattern that maps to both IRSA and Azure Workload Identity, so adding Azure later is a configuration and bootstrap exercise, not an architectural change.
- Cloud-shaped resources (managed Postgres, secret store, object storage) are accessed via Crossplane / ESO abstractions in v1.0 against AWS backends; Azure equivalents will need provider configuration but no new abstraction layer.
- **Revisit triggers**: a deployment environment that requires AKS, or a CI environment that requires Jenkins / GitLab CI, reopens this decision and pulls the deferred Azure bootstrap (§ 1.17) and multi-CI work (§ 3.14) into scope.

## References

- architecture-overview.md § 3 (baseline assumptions), § 6.11 (identity federation)
- architecture-backlog.md § 1.17 (identity federation details, Azure bootstrap deferred), § 3.14 (multi-CI support beyond GitHub Actions)
- ADR 0003 (Envoy egress proxy — cloud/CNI portability)
- ADR 0010 (GitHub Actions only for v1.0)
- ADR 0028 (identity federation)
