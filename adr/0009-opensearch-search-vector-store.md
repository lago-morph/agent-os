# ADR 0009: OpenSearch as the search and vector store

- **Status**: Accepted
- **Date**: 2026-05-10

## Context

The platform needs a single retrieval-optimization tier for vector search, hybrid (lexical + vector) search, the `platform-knowledge-base` RAGStore, and short-term searchable audit indexes (architecture-overview.md §6.3, §6.4, §6.5). The same store is also where Letta-maintained retrieval indexes land and where LiteLLM callbacks and Envoy egress write audit events. The choice was narrowed to two options in architecture-backlog.md §2.15, and the architecture treats "anything in OpenSearch must be reproducible from a primary source (Postgres, object storage, or external system)" as an invariant (architecture-backlog.md §6).

## Decision

The platform adopts **OpenSearch** as the search and vector store (component **A11**, architecture-overview.md §5, §14.1). It is provisioned either as a self-hosted install or a managed OpenSearch service, with index templates and ingest pipelines managed as configuration. It hosts vector + hybrid search, RAG indexes (including the Knowledge Base), and the audit search index, and is consumed by the SDK's `rag.*` API, by Letta, and by the audit-emission pipeline.

## Alternatives considered

- **Elasticsearch** — Rejected. OpenSearch's OSS Security plugin already includes OIDC and SAML, removing the need for a separate auth proxy in front of search; vector and hybrid search capabilities are at parity with Elasticsearch for the platform's use cases (architecture-backlog.md §2.15). Elasticsearch's licensing and Security-tier gating offered no offsetting benefit.

## Consequences

- **Reproducibility invariant.** Anything in OpenSearch must be reproducible from
  a primary source — vectors and embeddings re-derived from documents in object
  storage, audit indexes re-derived from the object-storage archive, and any
  checkpoint-derived indexes re-derived from Postgres (architecture-overview.md
  §6.3). OpenSearch is never a system of record; reindex paths are first-class
  operational procedures and are exercised in DR drills (architecture-backlog.md
  F2).
- **OIDC/SAML via the OSS Security plugin.** OpenSearch authenticates against
  Keycloak directly using its OSS Security plugin, so it is intentionally
  excluded from the oauth2-proxy / auth-proxy fronting that other UIs receive
  (architecture-overview.md §6.6, auth-proxy table). Service accounts elsewhere
  in the platform reach it via API.
- **Vector + hybrid search at parity.** A single backend serves dense-vector
  retrieval, BM25 lexical search, and hybrid combinations, so the SDK's `rag.*`
  API and the Knowledge Base do not need a second store for either modality
  (architecture-overview.md §6.4).
- **Audit dual-write.** Audit events flow into OpenSearch for short-term
  searchable retention and are also archived to object storage; the policy
  schedule and lifecycle rules are deferred to Workstream F
  (architecture-overview.md §6.5, architecture-backlog.md §1.13).
- **OpenSearch Dashboards** is the operator UI for search and audit exploration,
  SSO'd through Keycloak alongside the other admin UIs
  (architecture-overview.md §6.1 interface inventory).
- **Foundational ordering.** A11 is a foundation component scheduled early so
  HolmesGPT, the Knowledge Base, Letta, and the audit pipeline have a retrieval
  tier to build against (architecture-backlog.md §14 sequencing).
- **Migration cost.** Moving off OpenSearch later would require re-pointing the
  SDK's `rag.*` API, Letta's index adapter, and the audit-emission callbacks;
  the reproducibility invariant means the data itself can always be rebuilt
  from primary sources, so migration risk is bounded.

## References

- architecture-overview.md § 6.3, § 6.4, § 6.5
- architecture-backlog.md § 2.15, 6
- ADR 0014 (Postgres primary, OpenSearch retrieval-only) — directly related
