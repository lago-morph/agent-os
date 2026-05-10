# ADR 0009: OpenSearch as the search and vector store

- **Status**: Accepted
- **Date**: 2026-05-10

## Context

The platform needs a single retrieval-optimization tier for vector search, hybrid (lexical + vector) search, the `platform-knowledge-base` RAGStore, and searchable audit indexes (architecture-overview.md §6.3, §6.4, §6.5). The same store is also where Letta-maintained retrieval indexes land and where LiteLLM callbacks and Envoy egress fan out audit events for query and dashboards. The choice was narrowed to two options in architecture-backlog.md §2.15, and the architecture treats "anything in OpenSearch must be reproducible from a primary source (Postgres, object storage, or external system)" as an invariant (architecture-backlog.md §6).

## Decision

The platform adopts **OpenSearch** as the search and vector store (component **A11**, architecture-overview.md §5, §14.1). It hosts vector + hybrid search, RAG indexes (including the Knowledge Base), and an advisory audit search index, and is consumed by the SDK's `rag.*` API, by Letta, and by query/dashboard surfaces fed from the platform audit adapter (ADR 0034).

### Dual-mode hosting

Per ADR 0033 (dual-mode initial targets), OpenSearch runs in two shapes:

- **Dev / integration (kind):** an in-cluster OpenSearch deployment, provisioned alongside other foundation components.
- **AWS deployments:** the AWS-managed OpenSearch service, provisioned through Crossplane (MR/XRD) so the cluster manifests look identical to the in-cluster case from the operator's perspective.

Operators do not write audit events, KB indexes, or test-result documents directly into either deployment. All writes go through component adapters (the audit adapter, the RAG adapter, the test-results streamer); OpenSearch is never addressed as a primary write endpoint.

## Alternatives considered

- **Elasticsearch** — Rejected. OpenSearch's OSS Security plugin already includes OIDC and SAML, removing the need for a separate auth proxy in front of search; vector and hybrid search capabilities are at parity with Elasticsearch for the platform's use cases (architecture-backlog.md §2.15). Elasticsearch's licensing and Security-tier gating offered no offsetting benefit.

## Consequences

- **Advisory role, not system of record.** OpenSearch is an asynchronous fanout
  for query and dashboards. The system of record for audit is Postgres + S3 via
  the platform audit adapter (ADR 0034); OpenSearch indexes are rebuildable from
  those primary sources at any time. The reproducibility invariant in
  architecture-backlog.md §6 makes this explicit: anything in OpenSearch must be
  reproducible from a primary source — vectors and embeddings re-derived from
  documents in object storage, audit indexes re-derived from Postgres + the S3
  archive, and any checkpoint-derived indexes re-derived from Postgres
  (architecture-overview.md §6.3). Reindex paths are first-class operational
  procedures and are exercised in DR drills (architecture-backlog.md F2).
- **Audit pipeline.** Audit events are written to Postgres + S3 by the platform
  audit adapter (ADR 0034); the adapter then fans out to OpenSearch
  asynchronously for searchable retention and dashboards. A lag or outage on the
  OpenSearch side does not block the system of record, and recovery is a
  reindex from S3/Postgres rather than a restore.
- **Test results fanout.** Per the ADR 0011 update, non-unit test results
  (integration, e2e, performance) stream to OpenSearch under the same
  advisory-fanout model: the test-results component writes its primary record
  through its adapter and fans out to OpenSearch for query and dashboards.
- **OIDC/SAML via the OSS Security plugin.** OpenSearch authenticates against
  Keycloak directly using its OSS Security plugin, so it is intentionally
  excluded from the oauth2-proxy / auth-proxy fronting that other UIs receive
  (architecture-overview.md §6.6, auth-proxy table). Service accounts elsewhere
  in the platform reach it via API.
- **Vector + hybrid search at parity.** A single backend serves dense-vector
  retrieval, BM25 lexical search, and hybrid combinations, so the SDK's `rag.*`
  API and the Knowledge Base do not need a second store for either modality
  (architecture-overview.md §6.4).
- **OpenSearch Dashboards** is the operator UI for search, audit exploration,
  and test-result exploration, SSO'd through Keycloak alongside the other admin
  UIs (architecture-overview.md §6.1 interface inventory).
- **Foundational ordering.** A11 is a foundation component scheduled early so
  HolmesGPT, the Knowledge Base, Letta, the audit pipeline, and the test-result
  streamer all have a retrieval tier to build against (architecture-backlog.md
  §14 sequencing).
- **Migration cost.** Moving off OpenSearch later would require re-pointing the
  SDK's `rag.*` API, Letta's index adapter, the audit adapter's fanout target,
  and the test-result streamer; because OpenSearch is advisory, the data itself
  can always be rebuilt from primary sources, so migration risk is bounded.

## References

- architecture-overview.md § 6.3, § 6.4, § 6.5
- architecture-backlog.md § 2.15, § 6
- ADR 0011 (test execution and results) — non-unit results fan out here
- ADR 0014 (Postgres primary, OpenSearch retrieval-only) — directly related
- ADR 0033 (dual-mode initial targets) — kind + AWS-managed hosting model
- ADR 0034 (platform audit pipeline) — Postgres + S3 system of record, OpenSearch fanout
