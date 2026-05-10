# ADR 0033: Initial implementation targets — AWS (EKS) and GitHub

- **Status**: Accepted
- **Date**: 2026-05-10

## Context

The architecture is written to run on EKS, AKS, or kind, and to integrate with any CI system through the `agent-platform` CLI rather than a hardcoded pipeline runtime (architecture-overview.md § 3). Section 6.11 likewise commits to OIDC-based federation that maps cleanly onto both AWS IRSA and Azure Workload Identity. Building and validating both cloud-side bootstraps in v1.0 would double the install scope (Workstream A) without a current consumer asking for Azure. We therefore need an explicit decision recording which targets v1.0 actually exercises versus which are kept as architecturally-supported paths.

Stateful primitives (Postgres, OpenSearch) likewise have a dual story: they must be runnable in-cluster for developer and integration-test loops, and as managed cloud services in production. The decision below pins down which side of that split is exercised in v1.0 and how components stay agnostic.

## Decision

For v1.0 the platform targets **AWS (EKS)** as the cloud and **GitHub** (source control + GitHub Actions) as the CI/source surface. The architecture continues to support **Azure (AKS, Azure Workload Identity, Azure Key Vault)** as a first-class option, but no Azure-specific install path, secret-provider wiring, or pipeline reference is built, tested, or shipped in v1.0. Local development and test continue to use **kind**.

**Stateful primitives run in dual-mode hosting.**

- Postgres runs in-cluster on kind (StatefulSet via the platform's Postgres operator) for dev and integration testing, and on **AWS RDS** in production.
- OpenSearch runs in-cluster on kind (StatefulSet) for dev and integration testing, and on **AWS-managed OpenSearch** in production.
- Both modes are provisioned through the same Crossplane MR/XRD shape, so callers consume a connection (endpoint + credentials handle published into the namespace) without knowing where the backend is hosted.
- This keeps the dev loop self-contained (no AWS account required) and the production path on managed services, without forking component code or test fixtures.
- The dual-mode hosting above is implemented via the substrate-abstraction pattern formalized in [ADR 0041](./0041-substrate-abstraction-via-crossplane-compositions.md) (substrate abstraction via Crossplane Compositions) — *not* via parallel manifest sets per substrate. [Kargo](./0040-kargo-promotion-fabric.md) promotes claim shapes that are uniform across substrates; per-environment differences live inside the matching Composition. AKS support, while architecturally committed, is implemented (when targeted) by adding a third Composition per relevant XRD rather than re-architecting.

**kind cluster OIDC bootstrap is a v1.0 deliverable.** Developer and integration-test workloads need cluster-OIDC → Keycloak federation (ADR 0028) to exercise the same identity path as EKS. The bootstrap consists of:

- a small kubeadm-patch utility that sets `--service-account-issuer`, `--service-account-jwks-uri`, and `--service-account-signing-key-file` on the kind control plane, and
- a static discovery-doc / JWKS host (a tiny in-cluster Service backed by ConfigMap) that Keycloak can reach to verify projected service-account tokens.

Without this, the kind path cannot validate the federation pattern and IRSA-vs-kind drift would only be discovered on EKS.

**AKS opt-in flags (forward-looking).** When AKS is exercised in a future release, the cluster must be provisioned with `--enable-oidc-issuer --enable-workload-identity`. These are opt-in flags on `az aks create` and are *not* on by default; provisioning AKS without them silently breaks the ADR 0028 federation path. Recording it here so the future Azure bootstrap (§ 1.17) does not rediscover this the hard way.

## Consequences

- Workstream A install scope is bounded to EKS + GitHub bootstraps for v1.0; Azure bootstrap (AKS provisioning with the OIDC/workload-identity flags above, Key Vault wiring for ESO) is deferred and tracked in architecture-backlog.md § 1.17.
- Aligns with ADR 0010: GitHub Actions-only CI is the matching CI decision; the reference pipeline shipped in v1.0 is GitHub-Actions-only and any Jenkins / GitLab CI work is deferred (architecture-backlog.md § 3.14).
- `kind` remains the supported developer and integration-test environment, so contributors do not need an AWS account to develop the platform. The kind OIDC bootstrap (above) is what makes that promise real for identity-federated workloads.
- Cloud portability is protected at the architecture level by ADR 0003 (Envoy egress proxy): egress controls are not tied to a CNI or cloud-specific L7 policy, so the Azure path is not blocked by v1.0 implementation choices.
- Identity federation (ADR 0028) defines a single OIDC-based pattern that maps to IRSA, Azure Workload Identity, *and* the kind cluster-OIDC issuer, so the dev, EKS, and future AKS paths all converge on one trust model.
- Cloud-shaped resources (managed Postgres, OpenSearch, secret store, object storage) are accessed via Crossplane / ESO abstractions in v1.0 against AWS backends and in-cluster equivalents on kind; Azure equivalents will need provider configuration but no new abstraction layer.
- Dual-mode Postgres/OpenSearch means component contracts are defined against the Crossplane claim, not against RDS or in-cluster StatefulSet specifics; tests must run against both modes in CI.
- **Revisit triggers**: a deployment environment that requires AKS, or a CI environment that requires Jenkins / GitLab CI, reopens this decision and pulls the deferred Azure bootstrap (§ 1.17) and multi-CI work (§ 3.14) into scope.

## References

- [architecture-overview.md](../architecture-overview.md) [§ 3](../architecture-overview.md#3-baseline-assumptions) (baseline assumptions), [§ 6.11](../architecture-overview.md#611-identity-federation) (identity federation)
- [architecture-backlog.md](../architecture-backlog.md) [§ 1.17](../architecture-backlog.md#117-identity-federation-details) (identity federation details, Azure bootstrap deferred), [§ 3.14](../architecture-backlog.md#314-multi-ci-support-beyond-github-actions) (multi-CI support beyond GitHub Actions)
- [ADR 0003](./0003-envoy-egress-proxy.md) (Envoy egress proxy — cloud/CNI portability)
- [ADR 0009](./0009-opensearch-search-vector-store.md) (OpenSearch dual-mode: in-cluster on kind, managed on AWS)
- [ADR 0010](./0010-github-actions-only-v1.md) (GitHub Actions only for v1.0)
- [ADR 0014](./0014-postgres-primary-opensearch-retrieval.md) (Postgres dual-mode: in-cluster on kind, RDS on AWS)
- [ADR 0028](./0028-identity-federation.md) (identity federation; kind cluster-OIDC issuer support)
- [ADR 0034](./0034-audit-pipeline-durable-adapter.md) (audit pipeline — depends on Postgres + S3 backends established here)
- [ADR 0036](./0036-mattermost-chat-integration.md) (Mattermost integration — part of v1.0 chat surface on this target)
- [ADR 0040](./0040-kargo-promotion-fabric.md) (Kargo promotion — uniform claim shapes across substrates)
- [ADR 0041](./0041-substrate-abstraction-via-crossplane-compositions.md) (substrate abstraction via Crossplane Compositions — dual-mode implementation pattern)
