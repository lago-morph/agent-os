# ADR 0009: OpenSearch as the search and vector store

## Status

Accepted

## Context

The platform needs a single store that handles vector search (RAG retrieval over
embeddings), hybrid search (BM25 + dense vectors with reciprocal rank fusion),
full-text search over the audit index, and the Knowledge Base index. Several
components depend on it: Letta uses it for memory retrieval (ADR 0005), the
Knowledge Base / `RAGStore` indexes documents and runbooks into it (ADR 0022),
and audit emission writes structured events to a dedicated index (overview §6.5).

We need the store to integrate with Keycloak — SSO is exclusively Keycloak (no
tool runs its own user directory). We also need vector and hybrid search at
production parity.

The serious candidates were Elasticsearch and OpenSearch. They are functionally
equivalent for our query workloads, but their OSS surfaces differ around
authentication.

## Decision

Use **OpenSearch** as the search and vector store. The OSS Security plugin
ships **OIDC against Keycloak** (and SAML) without an additional license, which
matches our Keycloak-only SSO invariant. Vector and hybrid search are at parity
with Elasticsearch for our use cases.

OpenSearch hosts:

- **RAG indexes** — embeddings and chunks for the Knowledge Base and per-agent
  RAG stores; queried via vector + hybrid search.
- **Audit index** — every audit-relevant event from every component, queried
  through Headlamp / Grafana data source.
- **Letta retrieval index** — vectors and metadata Letta uses to recall memory
  fragments (the durable memory state itself lives in Postgres).

OpenSearch is **retrieval-optimization, not primary storage**. This reinforces
the architecture invariant (overview §6.3, backlog §6):

> Anything in OpenSearch must be reproducible from a primary source — Postgres
> or object storage. Vectors can be re-derived from documents in object
> storage. The audit index can be rebuilt from the immutable archive.
> Checkpoints live in Postgres.

The role split is detailed in ADR 0014. Loss of an OpenSearch index is a
performance incident, never a data-loss incident: rebuild from the primary.

Authentication uses the OSS Security plugin configured against Keycloak via
OIDC. No basic-auth users are created; OpenSearch roles are mapped from
Keycloak groups.

Provisioning is via Crossplane v2 — `OpenSearchCluster` XRs for the install
itself, and index templates / ISM policies as configuration in Git.

## Consequences

- One store covers vector, hybrid, and full-text — no separate vector DB
  (no Pinecone, Weaviate, Qdrant) and no separate search engine.
- Keycloak SSO works out of the box without buying a commercial security tier.
- Rebuild procedures must exist and be tested for every index: KB, audit,
  Letta retrieval. These are part of the per-component backup/restore
  deliverable (overview §14.1).
- We track the OpenSearch fork's vector-search roadmap independently of
  Elastic's; if a future capability appears only upstream in Elasticsearch,
  we revisit.
- Operators learn one query DSL and one set of dashboards.

## Alternatives considered

- **Elasticsearch.** Equivalent search and vector capabilities. Rejected
  because OIDC and SAML in the OSS distribution require a paid tier; using
  Elasticsearch would force either a license purchase or an external auth
  shim (`oauth2-proxy` in front), neither of which we want for a store this
  central. License trajectory is also less predictable.
- **Dedicated vector DB (Pinecone, Weaviate, Qdrant) + separate search
  engine.** Rejected: two stores to operate, two backup stories, two auth
  integrations, and no benefit for our workload sizes. Hybrid search across
  two systems is awkward.
- **Postgres with `pgvector` only.** Rejected as the sole solution: fine for
  small RAG corpora, but full-text and audit search at platform scale want
  a real search engine. Postgres remains the primary store for state and
  checkpoints (ADR 0014).

## Related

- ADR 0014 — Postgres primary, OpenSearch retrieval-only (role split; this
  ADR is the product selection, 0014 is the data-architecture rule).
- ADR 0022 — Knowledge Base / `RAGStore` uses OpenSearch indexes.
- ADR 0005 — Letta uses OpenSearch for memory retrieval.
- Overview §5 (component table), §6.3 (memory and data), §6.5 (observability
  and audit).
- Backlog §2.15 (decision), §6 (invariant on reproducibility from primary
  source).
