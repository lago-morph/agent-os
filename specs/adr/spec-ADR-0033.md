# SPEC ADR-0033 — Initial implementation targets — AWS (EKS) and GitHub `[PROPOSED]`

> kind: ADR · workstream: — · tier: T1
> upstream: [B4] · downstream: [A18; A21; A23; B4; B15] · adrs: [0033] · views: [6.11]
> canon-glossary: FROZEN · canon-interface: FROZEN

## 1. Purpose & Problem Statement

ADR 0033 is a settled decision: v1.0 exercises **AWS (EKS)** as the cloud target and
**GitHub** (source control + GitHub Actions) as the CI/source surface, with **kind** as the
developer and integration-test environment. Azure (AKS, Workload Identity, Key Vault) stays
architecturally supported but is not built, tested, or shipped in v1.0. This SPEC states what
honoring that scoping requires of install, CI, identity-bootstrap, and substrate-consuming
components — it does not re-argue the target selection.

The problem the decision solves: an unbounded target matrix would double Workstream A install
scope with no current Azure consumer, and stateful primitives (Postgres, OpenSearch) need a
single dual-mode hosting story (in-cluster on kind, managed on AWS) that components consume
without knowing where the backend runs. The decision pins the exercised side of every split.

## 2. Scope

### 2.1 In scope
- The v1.0 target set: AWS (EKS) cloud + GitHub CI/source + kind dev/test, as a conformance
  boundary on install and CI artifacts.
- Dual-mode hosting of stateful primitives via the substrate-abstraction pattern (ADR 0044),
  not parallel manifest sets — components consume a Crossplane v2 XR, not RDS / StatefulSet specifics.
- The kind cluster-OIDC bootstrap as a v1.0 deliverable (so the kind path exercises ADR 0028 federation).
- The forward-looking AKS opt-in flag requirement recorded for the future Azure bootstrap.

### 2.2 Out of scope (and where it lives instead)
- The substrate-abstraction mechanism itself (XRDs, Compositions, connection-secret contract)
  — owned by ADR 0044 / component B4.
- The GitHub Actions reference pipeline build — owned by component B15 (CI/CD reference pipeline)
  per ADR 0010.
- Azure (AKS) bootstrap, Key Vault wiring, multi-CI support — deferred (architecture-backlog
  §1.17, §3.14); not a v1.0 artifact.
- Identity-federation trust-model design — owned by ADR 0028.

## 3. Context & Dependencies

Upstream consumed: B4 (Crossplane v2 Compositions) supplies the dual-mode hosting mechanism that
keeps components substrate-agnostic. Downstream consumers: A18 (audit pipeline — needs Postgres +
S3 backends on AWS, Postgres-only on kind), A21 (tenant onboarding), A23 (Kargo — promotes uniform
XR schemas across substrates), B15 (GitHub-Actions-only reference pipeline).

ADR decisions honored:
- **ADR 0033** (this): AWS+GitHub+kind is the v1.0 target set; Azure not built; kind OIDC bootstrap ships.
- **ADR 0044**: dual-mode hosting is realized via one XRD + one Composition per substrate, not parallel manifests.
- **ADR 0010**: GitHub Actions is the v1.0 CI; Jenkins/GitLab deferred.
- **ADR 0028**: a single OIDC federation pattern maps to AWS IRSA, future Azure Workload Identity, and the kind cluster-OIDC issuer.
- **ADR 0003**: Envoy egress proxy keeps egress control CNI/cloud-agnostic, protecting the future Azure path.
- **ADR 0009 / ADR 0014**: OpenSearch and Postgres each run dual-mode (in-cluster on kind, managed on AWS).

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs
This ADR imposes no new CRD/XRD. It consumes the substrate XRDs defined by ADR 0044 (`Postgres`,
`SearchIndex`, `ObjectStore`, `MongoDocStore`) which carry `substrateClass` and a
`connectionSecretRef`. The cluster substrate-selection label is `platform.io/environment` with
values `kind` | `aws`.

### 4.2 APIs / SDK surfaces
The `agent-platform` CLI (B9) is the CI-agnostic surface through which GitHub Actions drives the
platform (ADR 0010). The kind OIDC bootstrap exposes a static discovery-doc / JWKS host (an
in-cluster Service backed by a ConfigMap) reachable by Keycloak. Specific bootstrap-utility
flags (`--service-account-issuer`, `--service-account-jwks-uri`, `--service-account-signing-key-file`)
are stated in source; the utility's command surface beyond these is **not specified in source**.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
N/A — this ADR emits no CloudEvent of its own. Tenant/identity bootstrap events emitted by
realizing components fall under `platform.tenant.*` / `platform.security.*` per their own SPECs.

### 4.4 Data schemas / connection-secret contracts
Components contract against the ADR 0044 uniform connection-secret shape (`host`, `port`, `user`,
`password`, `dbname` or per-primitive equivalent) and substrate-agnostic status (`ready`,
`endpoint`, `version`), regardless of whether the backend is AWS-managed or in-cluster on kind.

## 5. OSS-vs-Custom Decision
N/A — ADR (target-selection scoping, no component build). Realizing components (B4, B15, the
kind OIDC bootstrap utility) make their own OSS-vs-custom calls in their own SPECs.

## 6. Functional Requirements

- REQ-ADR-0033-01: The v1.0 install path MUST provision and validate AWS (EKS) and kind targets only; no Azure-specific install path, secret-provider wiring, or pipeline reference is built, tested, or shipped in v1.0.
- REQ-ADR-0033-02: The v1.0 CI/source surface MUST be GitHub (source control + GitHub Actions); no Jenkins / GitLab CI reference is shipped (ADR 0010).
- REQ-ADR-0033-03: Stateful primitives (Postgres, OpenSearch, S3-shaped storage, Mongo) MUST run in-cluster on kind and as AWS-managed services in production, provisioned through the same XRD schema (ADR 0044) — not via parallel manifest sets per substrate.
- REQ-ADR-0033-04: Components MUST contract against the Crossplane v2 XR (connection-secret + substrate-agnostic status), NOT against RDS / in-cluster StatefulSet specifics.
- REQ-ADR-0033-05: A kind cluster-OIDC bootstrap MUST ship in v1.0: a kubeadm-patch utility setting the three service-account issuer/JWKS/signing-key flags, plus a static discovery-doc / JWKS host Keycloak can reach, so the kind path exercises the ADR 0028 federation chain.
- REQ-ADR-0033-06: CI MUST run component tests against BOTH the kind in-cluster mode and the AWS-managed mode of dual-mode primitives.
- REQ-ADR-0033-07: The future AKS bootstrap (when targeted) MUST provision clusters with `--enable-oidc-issuer --enable-workload-identity`; this requirement is recorded now so the deferred Azure path (§1.17) does not silently break the ADR 0028 federation. `[PROPOSED — not in source]` enforcement mechanism (the flag check is design-time for the future bootstrap, not a v1.0 artifact).

## 7. Non-Functional Requirements
- Security/identity: dev (kind), EKS (IRSA), and future AKS (Workload Identity) converge on one OIDC trust model (ADR 0028); kind OIDC bootstrap closes the IRSA-vs-kind drift gap.
- Portability: ADR 0003 (Envoy egress) keeps egress control off CNI/cloud-specific L7 policy, so the Azure path is not foreclosed by v1.0 choices.
- Multi-tenancy (§6.9): unchanged by substrate; tenancy is namespace-based regardless of target.
- Versioning (ADR 0030): substrate XR schema changes are versioning events on both Compositions (ADR 0044).
- Scale: dev loop is self-contained on kind (no AWS account required), keeping the contributor path cheap.

## 8. Cross-Cutting Deliverable Checklist
N/A — ADR. The realizing components (B4 substrate Compositions, B15 GitHub Actions pipeline, the
kind OIDC bootstrap utility, A18/A21/A23 substrate consumers) carry the §14.1 deliverables in
their own SPECs; conformance to this scoping is verified per §9.

## 9. Acceptance Criteria

- AC-ADR-0033-01: Honored when the v1.0 install/CI artifact set contains AWS (EKS) + GitHub Actions + kind paths and zero Azure-specific install / secret-provider / pipeline artifacts. (→ REQ-01, REQ-02)
- AC-ADR-0033-02: Honored when a dual-mode primitive is provisioned from one XR schema on a kind cluster and on an AWS cluster, and the consuming component is shown to read only the connection-secret + substrate-agnostic status in both. (→ REQ-03, REQ-04)
- AC-ADR-0033-03: Honored when, on a kind cluster, a projected service-account token is validated by Keycloak through the bootstrapped discovery-doc / JWKS host (federation chain exercised end-to-end). (→ REQ-05)
- AC-ADR-0033-04: Honored when CI is shown to run the same component test suite against both kind in-cluster and AWS-managed modes of a dual-mode primitive. (→ REQ-06)
- AC-ADR-0033-05: Honored when the future-AKS provisioning requirement (`--enable-oidc-issuer --enable-workload-identity`) is recorded as a gating precondition in the deferred Azure-bootstrap design note. (→ REQ-07)

## 10. Risks & Open Questions
- (med) Capability-parity is not promised across substrates (ADR 0044) — a test that passes on AWS-managed OpenSearch may exercise behavior kind cannot (e.g. object-store archive lifecycle); REQ-06 dual-mode CI surfaces the gap rather than hiding it.
- (low) `[PROPOSED]` — the exact CI mechanism enforcing "no Azure artifact shipped" (manifest lint vs. release-gate check) is not specified in source; flagged for B15 design.
- Open: the kind OIDC bootstrap utility's full command surface beyond the three named flags is not specified in source; defers to the bootstrap component's SPEC.

## 11. References
- ADR 0033 (this decision). Enforcing/realizing components: B4 (substrate Compositions), B15 (GitHub Actions pipeline), the kind OIDC bootstrap utility, A18/A21/A23 (substrate consumers).
- architecture-overview.md §3 (baseline assumptions), §6.11 (identity federation). architecture-backlog.md §1.17 (Azure bootstrap deferred), §3.14 (multi-CI deferred). ADR 0003, 0009, 0010, 0014, 0028, 0034, 0040, 0044.
