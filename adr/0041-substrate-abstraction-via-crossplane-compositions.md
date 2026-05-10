# ADR 0041: Substrate abstraction via Crossplane Compositions

- **Status**: Accepted
- **Date**: 2026-05-10

## Context

ADR 0033 commits the platform to dual-mode hosting: **kind** for dev /
integration (so kind is functionally complete without cloud services) and
**AWS (EKS)** for the production-ready target. The architecture-overview
already names this dual-mode pattern for Postgres (§ 6.3, ADR 0014) and
OpenSearch (§ 6.3, ADR 0009): "in-cluster on kind, AWS-managed via
Crossplane on AWS". The audit pipeline (ADR 0034) extends the same pattern
to S3, and per-agent / per-tenant / per-user databases (ADR 0020) extend
it to MongoDB.

Without a uniform abstraction over these substrate-asymmetric primitives,
every workload that consumes one would need to know its substrate, parallel
manifest sets would proliferate, and the Kargo cross-substrate promotion
fabric (ADR 0040) would break — a manifest that works on a kind dev cluster
would silently fail on an AWS staging cluster, defeating the point of
promoting the *same* declarative artifact across environments.

Crossplane Compositions are the canonical pattern for this: one XRD,
multiple Compositions selecting on a cluster-environment label, all writing
the same connection-secret shape and exposing the same status fields. We
already use Crossplane v2 for cloud-shaped composition (ADR 0006 split with
the kopf LiteLLM operator; ADR 0021 `GrafanaDashboard` XRD as the earliest
example); generalising the pattern is well within Crossplane's intended use.

## Decision

Each substrate-asymmetric primitive is wrapped in an XRD, with **one
Composition per supported substrate** (kind + AWS for v1.0).

**Naming convention.** `X`-prefixed names below refer to the cluster-scoped composite resource (XR); the user-facing namespaced claim drops the prefix (e.g., XR `XPostgres` ↔ claim `Postgres`). Both refer to the same XRD; earlier ADRs that name the claim form (`AuditLog`, `AgentDatabase`, `GrafanaDashboard`) and this ADR's XR-form names refer to the same artifacts.

**Initial primitives wrapped (v1.0):**

- **`XPostgres`** — kind: CloudNativePG `Cluster`. AWS: Crossplane
  `RDSInstance`. (ADR 0014)
- **`XSearchIndex`** for OpenSearch — kind: in-cluster OpenSearch operator.
  AWS: AWS-managed OpenSearch via Crossplane MR. (ADR 0009)
- **`XObjectStore`** for S3-shaped storage — kind: MinIO (or no-op, see
  capability-parity caveat). AWS: Crossplane S3 bucket. (ADR 0034)
- **`XMongoDocStore`** — kind: Bitnami MongoDB. AWS: DocumentDB or
  self-managed. (ADR 0020)
- **`XAuditLog`** — already declared in ADR 0034; explicitly an instance of
  this pattern, composing `XPostgres` + `XObjectStore` + an indexer.
- **`XAgentDatabase`** — already declared in ADR 0020 as `AgentDatabase`;
  per-agent / per-tenant / per-user database provisioning, composes
  `XPostgres` or `XMongoDocStore` depending on declared engine.
- **`XGrafanaDashboard`** — already in ADR 0021; on kind it is plain
  Grafana folders, on AWS the same Grafana but provisioned via Crossplane
  Compositions.

**Connection-secret schema is part of the XRD contract.** Both Compositions
write the same secret shape (host, port, user, password, dbname or the
equivalent fields per primitive). Secret-shape conformance is tested as an
architecture-level invariant, not left to convention.

**XR status fields are substrate-agnostic** (`ready`, `endpoint`, `version`).
Substrate-specific fields (RDS ARN, in-cluster Service paths) are
deliberately absent from the XR's user-visible status. Where absolutely
needed they live in a substrate-specific subfield with a clear
"implementation detail, do not depend on" comment.

**Every cluster carries an environment label** (`platform.io/environment=kind`
or `=aws`). An OPA Gatekeeper admission policy (ADR 0002) rejects any claim
where no Composition matches the cluster's label, preventing silent stuck
reconciliation from typos or missing Compositions.

**CRD/API versioning (ADR 0030) applies to XRDs.** Changing an XRD's claim
shape is a versioning event affecting both Compositions; conversion
webhooks and deprecation windows apply identically.

**Day-2 drift handling** is standard Crossplane behaviour: substrate-side
changes (someone resizing RDS in the AWS console) get reconciled back to
declared state. This is the documented day-2 model.

**Documented exceptions — intentionally NOT wrapped:**

- **Knative event sources** (ADR 0023): SQS exists on AWS, doesn't on kind;
  the shape difference isn't abstractable in a way that preserves
  behaviour. Sources differ per environment by design.
- **Cluster bootstrap** (cluster OIDC issuer, kind kubeadm patch, identity
  federation at install): pre-Crossplane work — Crossplane isn't running
  yet at bootstrap time.
- **Cloud provider stack** (which Crossplane providers are active):
  install-time configuration, not promotion-time.
- **Resource sizing / replicas / region**: normal Kustomize overlay values
  per environment, not architectural.

**Capability-parity caveat.** Substrate parity is not promised across all
primitives. Kind `XObjectStore` may produce "no archive" (the audit
pipeline degrades gracefully on kind per ADR 0034). Each XRD documents its
substrate-specific behavioural differences. The claim *shape* is
consistent; the *runtime behaviour* may differ.

## Alternatives considered

- **Parallel manifest sets per substrate** — what we'd have without this
  ADR. The landmines (drift, every consumer aware of its substrate, broken
  cross-substrate promotion) are exactly what we are trying to avoid.
- **Kustomize-only** — works for sizing / replicas but not for
  cross-resource topology differences (RDS instance vs in-cluster
  StatefulSet are not the same Kubernetes object).
- **Helm chart conditional templates** — works in some places but explodes
  for complex cross-resource composition.

## Consequences

- New work: a kind-side Composition for each XRD (CloudNativePG /
  OpenSearch operator / MinIO / Bitnami MongoDB) — mechanical engineering
  work, but a real budget item alongside the AWS Compositions.
- Connection-secret-shape and status-field discipline become tested
  architectural invariants.
- New admission guardrail (OPA Gatekeeper, ADR 0002): every claim must have
  a matching Composition for the cluster's environment label.
- Kargo (ADR 0040) promotes XRC claims with the same shape across
  substrates, making cross-substrate promotion mechanically uniform.
- The audit pipeline (ADR 0034) and per-agent database provisioning
  (ADR 0020) are natural instances of this pattern, not separate designs.
- An `OidcRoleMapping` unifying CRD remains in `future-enhancements.md`;
  per-service OIDC role-mapping configuration promotes via Kargo in v1.0
  without an XRD.

## References

- [architecture-overview.md](../architecture-overview.md) [§ 6.3](../architecture-overview.md#63-memory-and-data-architecture), [§ 6.12](../architecture-overview.md#612-crd-inventory)
- [ADR 0009](./0009-opensearch-search-vector-store.md) (OpenSearch as search / vector store)
- [ADR 0014](./0014-postgres-primary-opensearch-retrieval.md) (Postgres primary / OpenSearch retrieval)
- [ADR 0020](./0020-initial-mcp-services.md) (Initial MCP services — `XAgentDatabase` pattern)
- [ADR 0021](./0021-grafanadashboard-xrs.md) (`GrafanaDashboard` XRs — earliest example of this pattern)
- [ADR 0023](./0023-environment-specific-knative-sources.md) (Environment-specific Knative sources — documented exception)
- [ADR 0026](./0026-independent-cluster-install-no-federation.md) (Independent-cluster install)
- [ADR 0030](./0030-crd-and-api-versioning-policy.md) (CRD / API versioning policy)
- [ADR 0033](./0033-initial-implementation-targets-aws-github.md) (Initial implementation targets AWS + GitHub)
- [ADR 0034](./0034-audit-pipeline-durable-adapter.md) (Audit pipeline — instance of this pattern)
- [ADR 0040](./0040-kargo-promotion-fabric.md) (Kargo promotion fabric — substrate abstraction makes its cross-substrate promotion clean)
