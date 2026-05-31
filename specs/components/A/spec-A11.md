# SPEC A11 — OpenSearch

> kind: COMPONENT · workstream: A · tier: T0
> upstream: [] · downstream: [A10, A18] · adrs: [0009, 0014, 0044, 0033, 0034, 0011, 0040, 0031, 0030, 0002] · views: [6.3, 6.4]
> canon-glossary: 6aadcc2a4f383a26 · canon-interface: 54f5ede58e5f8c05

## 1. Purpose & Problem Statement

A11 installs and operates **OpenSearch** as the platform's single **retrieval-optimization tier**: vector + hybrid (lexical + vector) search, RAG indexes including the `platform-knowledge-base` RAGStore, the **advisory audit search index**, Letta-derived retrieval indexes, and non-unit test-result indexes (ADR 0009, ADR 0014). It is dual-mode hosted via the **`SearchIndex` Crossplane XRD** — in-cluster on kind, AWS-managed OpenSearch on AWS — with one Composition per substrate writing a uniform connection secret (ADR 0044).

The defining architectural constraint is the **reproducibility invariant** (ADR 0014, backlog §6): *anything in OpenSearch must be reproducible from a primary source.* OpenSearch is **never a system of record** — Postgres + S3 are the audit system of record (ADR 0034); vectors re-derive from object storage; audit indexes re-derive from S3 (AWS) or Postgres (kind). Every index has a documented rebuild path. A11 lands early (foundation) so HolmesGPT, the Knowledge Base, Letta, the audit pipeline, and the test-result streamer all have a retrieval tier to build against.

A11 is **T0, contract-owning** — it owns the search-tier contract (connection secret, advisory-only role, OIDC-via-OSS-Security-plugin) that Letta (A10) and the audit pipeline (A18) consume directly, and that the SDK `rag.*` API, the Knowledge Base, and Kargo promotion (ADR 0040) build on.

## 2. Scope

### 2.1 In scope

- **OpenSearch** install in both modes via the `SearchIndex` XRD: in-cluster on kind, AWS-managed via Crossplane on AWS.
- Vector + hybrid (BM25 + dense-vector) search at parity from a single backend.
- Hosting the RAG indexes (incl. the `platform-knowledge-base` RAGStore index), the advisory audit index, Letta-derived indexes, and test-result indexes — **as advisory fanout only**.
- The **reproducibility invariant** enforcement posture: every index has a documented rebuild/reindex path from a primary; reindex is a first-class operational procedure (exercised in DR drills, F2).
- **OIDC/SAML via the OpenSearch OSS Security plugin** authenticating directly against Keycloak (intentionally excluded from the oauth2-proxy fronting other UIs).
- **OpenSearch Dashboards** as the operator UI (search/audit/test-result exploration), SSO'd through Keycloak.
- The uniform connection-secret contract (endpoint/host, port, user, password, optional TLS material) written by both Compositions.
- The §14.1 standard deliverable set.

### 2.2 Out of scope (and where it lives instead)

- **The `SearchIndex` XRD + per-substrate Compositions themselves** — authored by **B4** (Crossplane Compositions) per ADR 0044; A11 *consumes/installs through* the XRD and owns the OpenSearch product operation. (The XRD is the contract; B4 owns the composition code.)
- **The audit endpoint, adapter library, OpenSearch indexer service, S3 batch CronJob** — **A18** (audit pipeline); A11 hosts the index A18's indexer writes to, but does not own the indexer.
- **Letta memory backend** — **A10**; A11 hosts Letta's derived retrieval indexes.
- **Knowledge Base RAG indexing pipeline** — **C8** (authored-doc indexing); vendor-doc acquisition is a **separate companion project** (ADR 0024). A11 hosts the index; it does not run the ingestion pipeline.
- **The SDK `rag.*` API** — **B6**; A11 is the backend it terminates at (via LiteLLM call path).
- **Postgres / object storage** (the primaries) — **B4** XRDs (`Postgres`, `ObjectStore`) + their operators (CloudNativePG/RDS, MinIO/S3).
- **Test-results streamer / 3-layer test orchestration** — **B14** / ADR 0011; A11 hosts the advisory test-result index.
- **OpenSearch as an MCP service** (system-mediated + external-credentialed) — that MCP integration is **A17** (ADR 0020); A11 is the platform's own OpenSearch it targets.

## 3. Context & Dependencies

**Upstream consumed:** None at the wave level (W0 foundation). At runtime it sits **behind** the `SearchIndex` XRD; that XRD/Composition is authored by B4, but A11 (and the broader v1.0 substrate set) is foundation — A11 installs the OpenSearch product the XRD composes. The reproducibility invariant means A11's data depends on primaries (Postgres/S3/object storage) for rebuild, but A11 does not block on them to stand up the search tier.

**Downstream consumers (what they consume):**
- **A10 (Letta)** — writes derived retrieval indexes into OpenSearch (via Letta's index adapter), Postgres state is primary.
- **A18 (audit pipeline)** — its OpenSearch indexer fans out `platform.audit.*` events into the advisory audit index hosted here.
- Cross-cutting (non-blocking): the SDK `rag.*` API (B6), the Knowledge Base (C8 indexing), the test-result streamer (B14), HolmesGPT (A14), Kargo promotion (A23) consuming the connection secret.

**ADR decisions honored:**
- **ADR 0009** — OpenSearch is the search + vector store; advisory role, not system of record; OIDC/SAML via OSS Security plugin (excluded from oauth2-proxy fronting); OpenSearch Dashboards is the operator UI; foundation ordering (early).
- **ADR 0014** — Postgres primary / OpenSearch retrieval-only; every index reproducible from a primary; write paths hit a primary first; DR is Postgres-restore + object-storage durability + OpenSearch **reindex** (not OpenSearch backup/restore).
- **ADR 0044** — `SearchIndex` XRD with per-substrate Compositions; uniform connection-secret shape; substrate-agnostic status (`ready`, `endpoint`, `version`); Gatekeeper rejects XRs with no matching Composition.
- **ADR 0033** — dual-mode targets: in-cluster on kind, AWS-managed on AWS.
- **ADR 0034** — OpenSearch is **advisory fanout only** for audit; if down, audit ingestion still succeeds; rebuildable from S3 (AWS) / Postgres (kind).
- **ADR 0011** — non-unit test results stream to OpenSearch as an advisory index.
- **ADR 0040** — the uniform connection secret is consumed by Kargo during environment promotion.
- **ADR 0002** — substrate-claim admission (Gatekeeper) rejects no-matching-Composition claims.
- **ADR 0031 / 0030** — any A11-emitted CloudEvents fall under the taxonomy; versioning policy applies.

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)

A11 is **consumed through** (does not own the reconciler for) the substrate XRD — owner is Crossplane Compositions (B4) per ADR 0044:

| XR / XRD | Scope | Key fields (source-stated) |
|---|---|---|
| `SearchIndex` (XRD) | namespaced | `version`, `nodeCount`, `storage`, `connectionSecretRef`, `substrateClass`. kind: in-cluster OpenSearch; AWS: managed OpenSearch |

XR status (substrate-agnostic, ADR 0044): `ready`, `endpoint`, `version`.

A11-owned platform CRD reconciler: **N/A — A11 installs/operates the OpenSearch product behind the B4-owned `SearchIndex` XRD; it introduces no platform CRD.** (RAG content is modeled by the `RAGStore` CRD, owned by B13, not A11.)

### 4.2 APIs / SDK surfaces

- **OpenSearch HTTP API** — vector + hybrid + BM25 search; the backend the SDK `rag.*` API (B6, via LiteLLM), Letta's adapter (A10), the audit indexer (A18), and the test-result streamer (B14) write/query against. Writers reach it **only through their adapters** — OpenSearch is never addressed as a primary write endpoint (ADR 0009).
- **OpenSearch Dashboards** — operator UI, Keycloak-SSO'd.
- **OSS Security plugin OIDC/SAML** — authenticates against Keycloak directly (no oauth2-proxy in front).
- No bespoke platform HTTP service; A11 is the product, not a wrapper service.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)

Emitted (per-event-type names deferred to B12):
- `platform.observability.*` — index-health / reindex-lag threshold crossings. `[PROPOSED — not in source]` exact event-type names → B12.
- `platform.audit.*` — operator actions on OpenSearch (admin operations) via the audit adapter, where audit-relevant.

Consumed: A11 does not subscribe to events for its core function; the **audit indexer (A18)** consumes `platform.audit.*` and writes the advisory index hosted here — but that indexer is A18, not A11. `[PROPOSED — not in source]` whether A11 itself emits any lifecycle event vs the audit pipeline — treat fanout as A18's concern.

Every event carries `specversion` + `schemaVersion` (ADR 0031/0030).

### 4.4 Data schemas / connection-secret contracts

- **Uniform connection-secret contract** (ADR 0044): both Compositions write the same shape — for OpenSearch, ADR 0009 states **endpoint, user, password, optional TLS material** (mapping onto the canonical host/port/user/password/dbname shape; `dbname` N/A for a search index). This is the contract A10/A18/A23 bind to without knowing the substrate.
- **Reproducibility contract:** every index hosted here (RAG/KB, audit, Letta-derived, test-results, checkpoint-derived) has a **documented rebuild source** — object storage for vectors, S3/Postgres for audit, Postgres for Letta/checkpoints, primary record for test-results. A11 holds **no system-of-record data**.
- `[PROPOSED — not in source]` exact per-index mapping/schema definitions (vector dims, analyzer config) — design-time per consumer, not specified at architecture level.

## 5. OSS-vs-Custom Decision

- **OpenSearch:** OSS. **Decision: config** — install in-cluster on kind and use AWS-managed OpenSearch on AWS, both fronted by the `SearchIndex` XRD (B4). No fork.
- **ADR 0009 linkage:** OpenSearch over Elasticsearch because the OSS Security plugin includes OIDC/SAML (removes a separate auth proxy) and vector/hybrid search is at parity; Elasticsearch licensing/Security-tier gating offered no offsetting benefit.
- **Dual-mode** is implemented via the substrate-abstraction pattern (ADR 0044) — Composition code owned by B4; A11 owns operating the product and its reindex/DR procedures.
- Custom code in A11 is limited to install/config, the reindex runbooks, dashboards, and the standard deliverable set. No custom search service is built.

## 6. Functional Requirements

- **REQ-A11-01:** A11 SHALL provision OpenSearch in-cluster on kind and AWS-managed on AWS, fronted by the `SearchIndex` XRD with per-substrate Compositions (ADR 0044/0033).
- **REQ-A11-02:** OpenSearch SHALL serve dense-vector retrieval, BM25 lexical search, and hybrid combinations from a single backend (ADR 0009).
- **REQ-A11-03:** A11 SHALL be **advisory only** — never a system of record; every index SHALL have a documented rebuild path from a primary (object storage / S3 / Postgres) (ADR 0014).
- **REQ-A11-04:** If OpenSearch is unavailable, **audit ingestion SHALL still succeed** (the audit system of record is Postgres + S3); recovery SHALL be a reindex, not a restore (ADR 0034/0014).
- **REQ-A11-05:** Writers SHALL reach OpenSearch only through their component adapters (audit adapter, RAG adapter, Letta index adapter, test-results streamer); OpenSearch SHALL NOT be addressed as a primary write endpoint (ADR 0009).
- **REQ-A11-06:** OpenSearch SHALL authenticate via the OSS Security plugin's OIDC/SAML directly against Keycloak, and SHALL be excluded from the oauth2-proxy fronting applied to other UIs (ADR 0009).
- **REQ-A11-07:** OpenSearch Dashboards SHALL be the operator UI, SSO'd through Keycloak.
- **REQ-A11-08:** Both Compositions SHALL write the **uniform connection secret** (endpoint/host, port, user, password, optional TLS) so consumers do not branch on substrate (ADR 0044/0009).
- **REQ-A11-09:** The XR status surface SHALL be substrate-agnostic (`ready`, `endpoint`, `version`); substrate-specific fields SHALL be absent (ADR 0044).
- **REQ-A11-10:** Reindex paths SHALL be first-class operational procedures, exercised in DR drills (F2) (ADR 0014).
- **REQ-A11-11:** A claim for `SearchIndex` with no matching Composition for the cluster's `platform.io/environment` label SHALL be rejected at admission (ADR 0044/0002).
- **REQ-A11-12:** Audit-relevant operator actions on OpenSearch SHALL emit via the platform audit adapter; A11 SHALL NOT write its own audit records directly to any store (ADR 0034).

## 7. Non-Functional Requirements

- **Security / multi-tenancy (§6.9):** OSS Security plugin enforces Keycloak-claim-based access; index/document-level access aligns to tenant namespaces; RBAC-as-floor / OPA-as-restrictor for any platform-mediated access (ADR 0018). Secrets via ESO / connection secret.
- **Durability / DR:** A11 is **not** the durability boundary — DR posture focuses on Postgres restore + object-storage durability; OpenSearch recovery is reindex from primaries (ADR 0014). Migration off OpenSearch is a reindex exercise, not a data migration (bounded risk).
- **Observability (§6.5):** OTel + metrics (index health, query latency, reindex lag); per-component Grafana dashboard via `GrafanaDashboard` XR. OpenSearch is itself a query/dashboard target for audit + test results.
- **Scale:** sized for v1.0 vector + hybrid + audit-fanout volumes; `nodeCount`/`storage` set per substrate via the XRD.
- **Versioning (ADR 0030/0044):** `SearchIndex` claim shape versioned with conversion webhooks + deprecation windows (owned by B4); `version` field tracks the OpenSearch version.

## 8. Cross-Cutting Deliverable Checklist (§14.1)

| Deliverable | Status |
|---|---|
| Helm values / manifests in Git | applicable — in-cluster chart; AWS path via `SearchIndex` Composition (B4-authored) |
| Per-product docs (10.5) | applicable |
| Operator runbook (10.7) | applicable — reindex procedures, OIDC config, dual-mode operation |
| Backup / restore | applicable-with-caveat — **reindex from primaries**, not OpenSearch backup/restore (ADR 0014); DR drill in F2 |
| Alert rules | applicable — cluster red/yellow, reindex lag, storage pressure, query-latency breach |
| Grafana dashboard (`GrafanaDashboard` XR) | applicable |
| Headlamp plugin | applicable — index/cluster-health visibility (A11 ensures integration in our env) |
| OPA/Rego integration | applicable — substrate-claim admission (`SearchIndex` no-matching-Composition reject); access-policy helpers (Rego in B16) |
| Audit emission (ADR 0034) | applicable — operator actions; note A11 also *hosts* the advisory audit index (written by A18's indexer) |
| Knative trigger flow | applicable — reindex-lag / index-health observability flow under `platform.observability.*` |
| HolmesGPT toolset | applicable — index-health query, reindex-status, search-diagnostics tools |
| 3-layer tests | applicable — Chainsaw (`SearchIndex` claim + admission), PyTest (vector/hybrid query, advisory-fanout-down behavior), Playwright (Dashboards/Headlamp) |
| Tutorials & how-tos | applicable — "query the KB", "trigger a reindex", "connect Dashboards" |

## 9. Acceptance Criteria

- **AC-A11-01** (REQ-A11-01): An `SearchIndex` claim provisions in-cluster OpenSearch on kind and AWS-managed OpenSearch on AWS; the XR schema is identical across substrates. *(Chainsaw, two substrates)*
- **AC-A11-02** (REQ-A11-02): The same backend answers a dense-vector query, a BM25 query, and a hybrid query for the same corpus. *(PyTest)*
- **AC-A11-03** (REQ-A11-03): For each index class, a documented rebuild path exists and a reindex reproduces the index from its primary. *(PyTest + doc check)*
- **AC-A11-04** (REQ-A11-04): With OpenSearch stopped, an audit write still succeeds at the system of record (Postgres/S3); after restart a reindex restores the advisory index. *(PyTest)*
- **AC-A11-05** (REQ-A11-05): A direct primary-write attempt against OpenSearch (bypassing an adapter) is identified as a violation in test; supported writers go through adapters. *(PyTest)*
- **AC-A11-06** (REQ-A11-06): OpenSearch authenticates a Keycloak OIDC user via the OSS Security plugin; OpenSearch is not behind oauth2-proxy. *(PyTest/Chainsaw)*
- **AC-A11-07** (REQ-A11-07): OpenSearch Dashboards loads with Keycloak SSO. *(Playwright)*
- **AC-A11-08** (REQ-A11-08): The connection secret written by each Composition has identical keys (endpoint/host, port, user, password, optional TLS); a consumer binds to the same keys on both substrates. *(Chainsaw)*
- **AC-A11-09** (REQ-A11-09): The XR status exposes only `ready`, `endpoint`, `version`; no substrate-specific fields (e.g. RDS ARN, Service path) appear. *(Chainsaw)*
- **AC-A11-10** (REQ-A11-10): A DR-drill reindex procedure runs end to end and is documented as a first-class runbook. *(doc check + PyTest)*
- **AC-A11-11** (REQ-A11-11): On a `kind`-labeled cluster, an `SearchIndex` claim whose only Composition targets `aws` is rejected at admission. *(Chainsaw)*
- **AC-A11-12** (REQ-A11-12): An audit-relevant operator action emits via the audit adapter; no direct store write from A11. *(PyTest)*

## 10. Risks & Open Questions

- **R1 (med):** Boundary with B4 — A11 operates OpenSearch but the `SearchIndex` XRD/Composition is B4-authored. A drift between A11's install expectations and B4's Composition breaks dual-mode. *Reconciliation:* A11 and B4 jointly own the `SearchIndex` field/secret contract; A11 tests both substrates. Blast radius med.
- **R2 (med):** Capability-parity caveat — AWS-managed vs in-cluster OpenSearch may differ (plugin availability, version). *Open question:* is the OSS Security plugin OIDC path identical on AWS-managed OpenSearch? Confirm during implementation; document parity gaps explicitly (ADR 0044 mandate).
- **R3 (low):** A11 hosts the advisory audit index but A18 owns the indexer; ownership of index lifecycle (mapping, rollover) must be explicit to avoid a gap. *Reconciliation:* A18 owns the indexer + index schema; A11 owns the cluster + reindex capability.
- **R4 (low):** Per-index mapping/schema (vector dims, analyzers) is `[PROPOSED — not in source]` — design-time per consumer (B6/C8/A18/B14), not architecture-level.
- **OQ1:** Whether A11 emits its own lifecycle CloudEvents vs leaving all event emission to consumers is `[PROPOSED — not in source]`; recommend `platform.observability.*` for index-health only.

## 11. References

- architecture-overview.md §6.3 Memory and data architecture (line ~294 — storage roles, reproducibility rule of thumb, dual-mode hosting, audit pipeline), §6.4 The Knowledge Base as a separate primitive (~349), §6.5 (~370, observability — advisory audit index), §6.13 (~981, versioning).
- ADR 0009 (OpenSearch search/vector store), ADR 0014 (Postgres primary / OpenSearch retrieval-only), ADR 0044 (substrate abstraction — `SearchIndex`), ADR 0033 (dual-mode targets), ADR 0034 (audit pipeline — advisory fanout), ADR 0011 (three-layer testing — test-result fanout), ADR 0040 (Kargo — connection-secret consumer), ADR 0002 (substrate-claim admission), ADR 0031 (CloudEvent taxonomy), ADR 0030 (versioning), ADR 0024 (vendor-doc companion project).
- Related pieces: A10 (Letta), A18 (audit pipeline), B4 (`SearchIndex` Composition), B6 (SDK `rag.*`), C8 (KB indexing), B14 (test-results streamer), A17 (OpenSearch-as-MCP), A23 (Kargo).
