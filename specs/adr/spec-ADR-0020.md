# SPEC ADR-0020 — Initial MCP services set [PROPOSED]

> kind: ADR · workstream: — · tier: T0
> upstream: [A17, A1, B13] · downstream: [A7, B4, A11, A16, A14, B10] · adrs: [0020] · views: [6.1, 6.8]
> canon-glossary: 6aadcc2a4f38 · canon-interface: 54f5ede58e5f

## 1. Purpose & Problem Statement
ADR 0020 is a settled T0 decision: the v1.0 initial MCP services integration (component **A17**) is a fixed set — **GitHub**, **Google Drive**, **Context7**, **OpenSearch**, **Postgres**, **MongoDB**, and generic **web search + web scrape** — each registered as a namespaced `MCPServer` CRD reconciled by the kopf operator (B13) into LiteLLM, with secrets via ESO and access gated by CapabilitySet inclusion plus OPA. **Firecrawl is removed** from the set (replaced by the generic web services). This SPEC states the constraints, credential-mode contracts, XRD wiring, and conformance the decision imposes. It does not re-argue the service selection.

The problem the decision solves: a fixed initial set lets ESO plumbing, OPA admission rules, CapabilitySet wiring, audit tagging, and Headlamp inspection have concrete exercised integrations from day one, and lets early Platform Agents compose against a known surface.

## 2. Scope
### 2.1 In scope
- The fixed v1.0 MCP set, each as a namespaced `MCPServer` CRD reconciled by B13 into the LiteLLM registry.
- Three credential modes as first-class: system-credential, user-credential (LiteLLM-brokered OAuth), system-mediated (OpenSearch against the platform's own instance).
- Per-agent/tenant/user database provisioning fronted by the **`XAgentDatabase`** XRD, composing `XPostgres` / `XMongoDocStore` per substrate (ADR 0041); RBAC + OPA decide assignment.
- OpenSearch MCP system-mediated mode reusing `XSearchIndex`.
- Generic web-search/web-scrape servers with policy enforced at the MCP-server-access layer (OPA), endpoints kept simple internally.
- ESO-sourced secrets; no gateway-side bypass — adding a service means landing a new `MCPServer` CRD + ESO/OPA wiring.
- Firecrawl removed; commercial fallback deferred.

### 2.2 Out of scope (and where it lives instead)
- A17 implementation (install, registration, wiring) — component **A17** SPEC.
- Per-service auth flows, scopes, secret shapes, revocation, budget enforcement, per-XRD field schemas, search/scrape OPA bundle contents — design-time (architecture-backlog §1.11).
- MCP-server health-check details — deferred (architecture-backlog §1.9).
- `XAgentDatabase` / `XPostgres` / `XMongoDocStore` / `XSearchIndex` Composition builds — component **B4** / ADR 0041.
- kopf operator — component **B13** / ADR 0013; LiteLLM broker — **A1**.
- Commercial fallback (Firecrawl, Google Search API) — future-enhancements.

## 3. Context & Dependencies
Upstream consumed: **A17** (initial MCP services integration) implements the set; **A1** LiteLLM (single broker for MCP credentials/OAuth, shared callback chain); **B13** kopf operator (reconciles `MCPServer` CRDs). Downstream consumers / conformers: **A7** OPA (admission + access gating), **B4** Crossplane (`XAgentDatabase`/`XPostgres`/`XMongoDocStore`/`XSearchIndex`), **A11** OpenSearch (system-mediated target), **A16** Interactive Access Agent, **A14** HolmesGPT, **B10** Coach (compose against the set via CapabilitySet).

ADR decisions honored:
- **ADR 0020** (this) — the fixed initial set; three credential modes; `XAgentDatabase` fronting Postgres/MongoDB; web services policed at the access layer; Firecrawl removed.
- **ADR 0013** — each service is an `MCPServer` CRD; access via CapabilitySet inclusion + OPA; B13 reconciles; no bypass.
- **ADR 0009** — OpenSearch hosting (in-cluster kind / AWS-managed).
- **ADR 0014** — RBAC + OPA decide database assignment at admission/request time, not hard-coded in the MCPServer spec.
- **ADR 0018** — RBAC-floor / OPA-restrictor governs database assignment and search/scrape access.
- **ADR 0021** — `XAgentDatabase` follows the Crossplane XR composition pattern.
- **ADR 0033** — set sized for the v1.0 initial implementation targets; dual-hosting story.
- **ADR 0038** — search/scrape OPA bundle is a primary policy-simulator target before rollout.
- **ADR 0040 / 0041** — Kargo promotes `XAgentDatabase`/`XSearchIndex` claims uniformly; substrate-abstraction pattern composes `XPostgres`/`XMongoDocStore`.

## 4. Interfaces & Contracts
### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
- `MCPServer` (namespaced, B13) — one per service: `endpoint`, `authMode` (system / user-cred / system-mediated), `credentialsRef` (ESO), `tags`, `scopes`, `visibility`.
- `XAgentDatabase` (XRD, namespaced claim `AgentDatabase`, B4) — `engine` (postgres/mongodb), `scope` (agent/tenant/user), `ownerRef`, `credentialsSecretRef`. Composes `XPostgres` (CloudNativePG kind / RDS AWS) or `XMongoDocStore` (Bitnami kind / DocumentDB-or-self-managed AWS).
- `XSearchIndex` (XRD, B4) — reused by OpenSearch MCP system-mediated mode; `connectionSecretRef`, `substrateClass`.
- `CapabilitySet` (B13) — includes the `MCPServer` refs that grant agents access.

### 4.2 APIs / SDK surfaces
- LiteLLM is the single broker for MCP credentials and OAuth flows; all services route through the gateway's MCP path sharing the audit/OPA/Langfuse callback chain.
- Web search + web scrape expose generic MCP servers; OPA at the MCP-server-access layer blocks disallowed queries/domains/scrape targets.

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
- MCPServer registry add/update/delete under `platform.capability.*`; MCP health changes / routing under `platform.gateway.*`; access decisions under `platform.policy.*`; MCP calls emit audit under `platform.audit.*`. No new top-level namespace.

### 4.4 Data schemas / connection-secret contracts
- `XAgentDatabase`-provisioned databases write the uniform connection-secret shape (`host`, `port`, `user`, `password`, `dbname`) per ADR 0041, identical across substrates.
- ESO supplies MCP-service credentials referenced by `MCPServer.credentialsRef`; per-service secret shapes are design-time — `[PROPOSED — not in source]` (architecture-backlog §1.11).

## 5. OSS-vs-Custom Decision
N/A — ADR. (Enforcement note: GitHub / Google Drive / **Context7** MCP servers installed+registered; OpenSearch / Postgres / MongoDB fronted by platform XRDs; web search/scrape implemented by us or adopted from OSS — config/wrap + build-new, no fork of the gateway. Rejected: **Firecrawl** is removed from the initial set, replaced by the generic web services; commercial fallback deferred to future-enhancements.)

## 6. Functional Requirements
- REQ-ADR-0020-01: The v1.0 initial MCP set MUST be exactly {GitHub, Google Drive, Context7, OpenSearch, Postgres, MongoDB, web-search, web-scrape}, each registered as a namespaced `MCPServer` CRD reconciled by B13 into the LiteLLM registry; Firecrawl MUST NOT be in the set.
- REQ-ADR-0020-02: GitHub and Google Drive MUST ship both system-credential mode (platform-managed shared identity) and user-credential mode (LiteLLM brokers the user's own OAuth); this two-mode shape sets the pattern any later delegated-auth MCP service follows.
- REQ-ADR-0020-03: OpenSearch MUST ship a system-mediated mode against the platform's own OpenSearch (reusing `XSearchIndex`) plus an agent/tenant/user-credentialed mode for externally operated instances.
- REQ-ADR-0020-04: Postgres and MongoDB per-agent/tenant/user database provisioning MUST be wrapped by the `XAgentDatabase` XRD that creates the database, role, and grants, composing `XPostgres` or `XMongoDocStore` per substrate (kind/AWS) with one uniform claim shape.
- REQ-ADR-0020-05: RBAC + OPA MUST decide which agent/tenant/user resolves to which provisioned database at admission and request time; assignment MUST NOT be hard-coded in the `MCPServer` spec.
- REQ-ADR-0020-06: Generic web-search / web-scrape endpoints MUST stay simple internally; access control MUST be enforced at the MCP-server-access layer where OPA can block specific queries, domains, or scrape targets.
- REQ-ADR-0020-07: LiteLLM MUST be the single broker for MCP credentials and OAuth flows; all services MUST share the gateway's audit/OPA/Langfuse callback chain.
- REQ-ADR-0020-08: MCP-service secrets MUST be sourced via ESO and referenced by `MCPServer.credentialsRef`.
- REQ-ADR-0020-09: There MUST be no gateway-side bypass — adding another v1.0 service MUST mean landing a new `MCPServer` CRD plus ESO/OPA wiring; unapproved MCP servers MUST be unreachable from a Platform Agent.
- REQ-ADR-0020-10: Early Platform Agents (Interactive Access Agent, HolmesGPT, Coach) MUST compose against this set via CapabilitySet inclusion (ADR 0013).

## 7. Non-Functional Requirements
- Security/tenancy: three credential modes are first-class and load-bearing; database assignment and web access are RBAC-floored / OPA-restricted (ADR 0018); the search/scrape OPA bundle is a primary policy-simulator target (ADR 0038) since false-positives degrade capability and false-negatives leak egress.
- Observability (§6.5): all MCP calls share the LiteLLM callback chain for audit, OPA, and Langfuse instrumentation; MCP health uses only what the protocol exposes (no synthetic health layer; surfacing deferred, architecture-backlog §1.9).
- Scale/substrate: `XAgentDatabase` is the first XR pattern minting *runtime* per-agent state; Kargo promotes these claims uniformly across substrates (ADR 0040).
- Versioning (ADR 0030): `MCPServer` (B13-owned) and the XRDs (B4-owned) versioned with conversion webhooks.

## 8. Cross-Cutting Deliverable Checklist
N/A — ADR (verification map in the PLAN). The §14.1 set is owned by enforcing components (A17, A1, B13, B4, A7).

## 9. Acceptance Criteria
- AC-ADR-0020-01: Honored when all eight services exist as reconciled `MCPServer` CRDs in the LiteLLM registry and no Firecrawl `MCPServer` is present. (REQ-01)
- AC-ADR-0020-02: Honored when GitHub/Google Drive each operate in both system-credential and user-credential (brokered OAuth) modes. (REQ-02)
- AC-ADR-0020-03: Honored when the OpenSearch MCP reaches the platform's own instance in system-mediated mode (via `XSearchIndex`) and an external instance in credentialed mode. (REQ-03)
- AC-ADR-0020-04: Honored when an `XAgentDatabase` claim provisions a per-scope database+role+grants composing `XPostgres`/`XMongoDocStore`, with the same claim shape on kind and AWS. (REQ-04)
- AC-ADR-0020-05: Honored when database assignment is denied/permitted by RBAC+OPA at admission/request, not by a static MCPServer field. (REQ-05)
- AC-ADR-0020-06: Honored when OPA at the access layer blocks a disallowed search query / scrape target while the endpoint itself stays policy-free. (REQ-06)
- AC-ADR-0020-07: Honored when every MCP call routes through LiteLLM and produces audit/OPA/Langfuse records via the shared callback chain. (REQ-07)
- AC-ADR-0020-08: Honored when each `MCPServer.credentialsRef` resolves to an ESO-managed secret. (REQ-08)
- AC-ADR-0020-09: Honored when an unapproved MCP server is unreachable from a Platform Agent and a new service requires a new `MCPServer` CRD + wiring, not a gateway toggle. (REQ-09)
- AC-ADR-0020-10: Honored when the Interactive Access Agent / HolmesGPT / Coach reach a service only when their CapabilitySet includes it. (REQ-10)

## 10. Risks & Open Questions
- OQ-1 (high): Per-service auth flows, scopes, secret shapes, revocation, budget enforcement, per-XRD field schemas, and search/scrape OPA bundle contents are all design-time (architecture-backlog §1.11); many ACs are partial until A17/B4 design lands. `[PROPOSED]`
- R-1 (med): Self-hosted scraping can be blocked by upstream sites (IP-block/rate-limit); the commercial-fallback path is deferred to future-enhancements, not landed.
- OQ-2 (low): MCP health-check surfacing is deferred (architecture-backlog §1.9); LiteLLM uses only protocol-native signals. `[PROPOSED]`
- R-2 (low): `XAgentDatabase` is the first XR minting runtime per-agent state — the precedent matters for later per-agent resources; blast radius bounded by the substrate-abstraction contract (ADR 0041).

## 11. References
- ADR 0020 (`adr/0020-initial-mcp-services.md`) — the decision enforced here.
- architecture-overview.md §6.1 (gateway), §6.8 (capability registries), §14.1 (A17); architecture-backlog.md §1.9, §1.11.
- Enforcing components: A17 (initial MCP services, owner), A1 (LiteLLM broker), B13 (kopf operator), B4 (Crossplane XRDs), A7 (OPA), A11 (OpenSearch).
- Related ADRs: 0009, 0013, 0014, 0018, 0021, 0033, 0038, 0040, 0041.
