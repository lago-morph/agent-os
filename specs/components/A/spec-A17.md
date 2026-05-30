# SPEC A17 — Initial MCP services integration

> kind: COMPONENT · workstream: A · tier: T0
> upstream: [A1, A7, B13] · downstream: [] · adrs: [0009, 0013, 0014, 0020, 0021, 0030, 0031, 0033, 0034, 0038, 0040, 0041] · views: [6.1, 6.8]
> canon-glossary: 6aadcc2a4f38 · canon-interface: 54f5ede58e5f

## 1. Purpose & Problem Statement

A17 lands the platform's **fixed initial set of MCP services** so that Platform Agents have a concrete, governed, day-one tool surface rather than an ad-hoc per-team onboarding flow. Per ADR 0020 the set is: **GitHub**, **Google Drive**, **Context7**, **OpenSearch**, **Postgres**, **MongoDB**, and **generic web-search + web-scrape** servers. Each is registered as a namespaced `MCPServer` CRD (reconciled by the B13 kopf operator into the LiteLLM registry), has its secrets sourced via **ESO**, and is reachable by a Platform Agent only through **CapabilitySet inclusion + OPA** gating.

The problem A17 solves is twofold. First, it makes the abstract "approved MCP server" primitive (§6.8) real by exercising every credential mode the architecture commits to: **system-credential**, **user-credential** (LiteLLM-brokered OAuth), and **system-mediated** (OpenSearch against the platform's own cluster). Second, it proves the per-agent/tenant/user runtime-database provisioning story by fronting Postgres and MongoDB with the `XAgentDatabase` XRD (ADR 0020/0041) — the first place a Crossplane XR mints *runtime* per-agent state. A17 carries no bypass path: adding a service means landing a new `MCPServer` CRD plus its ESO/OPA wiring, never a gateway-side toggle.

## 2. Scope

### 2.1 In scope
- **`MCPServer` CRD instances** (one or more per service) for the seven services, with the source-stated fields `endpoint`, `authMode`, `credentialsRef`, `tags`, `scopes`, `visibility`.
- **ESO wiring** per service: `ExternalSecret`/`SecretStore` plumbing that materializes each service's `credentialsRef` secret from the configured secret provider (AWS SM / Vault / AKV per §6.6).
- **CapabilitySet wiring**: registering each service so it can be referenced by `mcpServers[]` in a `CapabilitySet`; the resolved-view is read in Headlamp.
- **OPA integration (Rego)** contributed to the policy library for: MCP-server-access decisions (especially web-search/scrape query/domain/target gating), database-assignment gating for Postgres/MongoDB, and admission of the A17 `MCPServer` CRDs.
- **GitHub & Google Drive**: both **system-credential** and **user-credential** modes; LiteLLM brokers the user OAuth flow for user-credential mode.
- **Context7**: install + register; documentation/library lookup; system-credential.
- **OpenSearch MCP**: **system-mediated** mode against the platform OpenSearch (reusing `XSearchIndex`, ADR 0041) **plus** an agent/tenant/user-credentialed mode for externally operated OpenSearch.
- **Postgres & MongoDB MCP**: agent/tenant/user-scoped DB access fronted by the **`XAgentDatabase`** XRD (composing `XPostgres` / `XMongoDocStore`); RBAC + OPA decide which agent resolves to which database.
- **Generic web-search + web-scrape** in-cluster MCP servers (build-new or adopt OSS); access control enforced at the MCP-server-access layer, not internally.
- Standard §14.1 deliverables: Helm/manifests, per-product docs, runbooks, alerts, Grafana dashboard (XR), Headlamp plugin contribution, audit emission (rides the LiteLLM path), Knative trigger-flow design, HolmesGPT toolset, 3-layer tests, tutorials/how-tos.

### 2.2 Out of scope (and where it lives instead)
- **LiteLLM gateway + MCP broker path itself** → **A1**. A17 rides A1's MCP callback chain; no separate emission/health point.
- **The `MCPServer` reconciler** (CRD → LiteLLM registry) → **B13** kopf operator. A17 authors CRD *instances*, not the reconciler.
- **The Crossplane Composition machinery + the `XAgentDatabase` / `XPostgres` / `XMongoDocStore` / `XSearchIndex` XRD definitions** → **B4** (Crossplane Compositions). A17 consumes/claims them.
- **OPA engine + Gatekeeper install** → **A7**; **OPA framework** → B3; **initial OPA content** → B16. A17 contributes its own service-specific Rego.
- **Audit endpoint + adapter library** → **A18**. A17 emits via the adapter on the LiteLLM path.
- **Per-service auth-flow / scope / secret-shape / revocation / budget / per-XRD-field detail** → **deferred** per architecture-backlog §1.11; A17 implements within whatever those settle to and tags gaps `[PROPOSED]`.
- **OpenSearch install** → **A11**; **Postgres-on-kind (CloudNativePG) / Mongo (Bitnami) / object store** substrate installs → B4/substrate XRDs.
- **OpenAPI→MCP synthesis** (`SyntheticMCPServer`) → **A12** (not in A17's set).
- **Commercial scrape fallback** (Firecrawl, Google Search API) → future-enhancements (explicitly NOT v1.0).

## 3. Context & Dependencies

**Upstream consumed (HARD):**
- **A1 (LiteLLM)** — the single broker for MCP credentials and OAuth flows; A17's services route through the gateway's MCP path and share its callback chain for audit, OPA, and Langfuse. Consumes A1's user-credential OAuth brokering and dynamic-registration boundary.
- **A7 (OPA / Gatekeeper)** — the policy engine that gates MCP-server-access, database assignment, and admission of A17's CRDs.
- **B13 (kopf operator)** — reconciles each `MCPServer` CRD into the LiteLLM registry and emits `platform.capability.changed`.

**Effective build-order dependencies (named here, not in the CSV upstream list):**
- **B4 (Crossplane Compositions)** — owns `XAgentDatabase`, `XPostgres`, `XMongoDocStore`, `XSearchIndex`. A17 claims these; they must exist for Postgres/MongoDB/OpenSearch system-mediated modes. `[PROPOSED — not in source]` that A17 declares a hard dependency on B4 (the CSV lists only A1/A7/B13; B4 ownership of these XRDs is stated in waves.md and the interface contract).
- **A11 (OpenSearch)** — the platform OpenSearch the system-mediated OpenSearch MCP targets.
- **ESO** (installed via A15-adjacent baseline / §6.6) — secret materialization.

**Downstream consumers:** none in the CSV. De-facto consumers: early Platform Agents (Interactive Access Agent A16, HolmesGPT A14, Coach B10) compose against this capability surface via CapabilitySet.

**ADRs honored:**
- **ADR 0020** — the exact seven-service set, three credential modes, `XAgentDatabase` fronting for DBs, MCP-server-access as the enforcement point for web-search/scrape, Firecrawl removed, per-service detail deferred.
- **ADR 0013** — capability-CRD model; access via CapabilitySet inclusion; `platform.capability.changed` emission on reconcile.
- **ADR 0009 / 0033** — OpenSearch dual-hosting (in-cluster on kind, AWS-managed on AWS) underpinning system-mediated mode.
- **ADR 0014** — Postgres primary; DB assignment gated at admission + request time, not hard-coded in the `MCPServer` spec.
- **ADR 0021 / 0041** — Crossplane XR composition + substrate abstraction; `XAgentDatabase` composes `XPostgres` / `XMongoDocStore`; same claim shape every substrate; connection-secret contract.
- **ADR 0038** — the web-search/scrape OPA bundle is a primary policy-simulator target before rollout (false-positives degrade capability, false-negatives leak egress).
- **ADR 0040** — Kargo promotes `XAgentDatabase` claims uniformly across substrates.
- **ADR 0034** — audit emission via the adapter, riding the LiteLLM callback path.
- **ADR 0030 / 0031** — CRD versioning; CloudEvent taxonomy.

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
**`MCPServer`** (namespaced; owner B13; consumed here as instances). Source-stated fields (interface-contract §1.4): `endpoint`, `authMode` (system/user-cred), `credentialsRef`, `tags`, `scopes`, `visibility`. A17 authors instances; it does **not** invent fields.
- `[PROPOSED — not in source]` A **system-mediated** authMode value (distinct from system-cred / user-cred) is required by OpenSearch per ADR 0020 but is not enumerated in interface-contract §1.4's `authMode` (which lists only system/user-cred). Tagged for B13 to settle the enum.
- `[PROPOSED — not in source]` per-service `tags`/`scopes` values (e.g. a `web-search`/`web-scrape` tag, GitHub repo scopes) — concrete values deferred per backlog §1.11.

**`XAgentDatabase`** (XRD; claim form `AgentDatabase`; owner B4; consumed here). Source-stated fields: `engine` (postgres/mongodb), `scope` (agent/tenant/user), `ownerRef`, `credentialsSecretRef`. Composes `XPostgres` or `XMongoDocStore`. A17 emits claims; field shape beyond these is `[PROPOSED — not in source]` / deferred to B4 + backlog §1.11.

**`XSearchIndex`** (XRD; owner B4) — reused by the OpenSearch system-mediated MCP mode. Source-stated: `version`, `nodeCount`, `storage`, `connectionSecretRef`, `substrateClass`.

CRD versioning per ADR 0030: A17's `MCPServer` instances pin the `MCPServer` CRD major served by B13; conversion-webhook/deprecation lifecycle is B13-owned.

### 4.2 APIs / SDK surfaces
- A17 exposes **no new platform API**. MCP tool invocation is the standard `§7.3 MCP tool invocation` path through LiteLLM. The web-search/web-scrape servers expose an **MCP tool interface** only (tool names/argument schemas `[PROPOSED — not in source]`, deferred per backlog §1.11).
- **User-credential OAuth** (GitHub, Google Drive) is brokered by **LiteLLM** (A1); A17 supplies the per-service OAuth client config materialized via ESO. Exact broker mechanics, requested scopes, and revocation flows are deferred (backlog §1.11) and tagged `[PROPOSED]` where A17 must choose.
- HTTP APIs of any custom A17 service (web-search/scrape) follow URL-path versioning `/v1/...` (interface-contract §3.3).

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
- **`platform.capability.*`** — capability-registry changes (MCPServer add/update/delete) are emitted by **B13** on reconcile, including `platform.capability.changed`. A17 is the cause, B13 the emitter.
- **`platform.audit.*`** — every MCP request/response (incl. these services) is emitted on the **LiteLLM callback** path (§6.6: "A17 MCP services ride this path; no separate emission point") into the A18 adapter.
- **`platform.policy.*`** — OPA decisions at the MCP-server-access layer (web-search/scrape allow/deny, DB-assignment) are emitted by the OPA decision path; dry-run/simulator runs (ADR 0038) are audited here.
- **`platform.gateway.*`** — MCP health changes are LiteLLM gateway events (A1), not A17-specific.
- Per-event-type names/schemas live in **B12**'s registry — `[PROPOSED — not in source]` for any concrete A17 event name; A17 only commits to namespaces.

### 4.4 Data schemas / connection-secret contracts
- **Connection secret** (ADR 0041 uniform contract) written by `XAgentDatabase` → `XPostgres`/`XMongoDocStore` Compositions: `host`, `port`, `user`, `password`, `dbname` (or per-primitive equivalent). A17 consumes this secret to wire the Postgres/MongoDB MCP servers; it does not redefine the shape.
- **ESO-materialized credential secrets** for GitHub/Google Drive/Context7/external-OpenSearch: shape is per-service and **deferred per backlog §1.11** — `[PROPOSED — not in source]` for exact keys.
- XR status fields are substrate-agnostic: `ready`, `endpoint`, `version`.

## 5. OSS-vs-Custom Decision
- **GitHub / Google Drive / Context7 MCP servers** — **adopt OSS / vendor MCP servers + config + wrap**. No pinned versions in source → `[PROPOSED — not in source]` to pin at implementation. Wrapping = ESO secret injection + `MCPServer` CRD registration + OPA gating; no fork.
- **OpenSearch / Postgres / MongoDB MCP** — **config + wrap** around the platform's own substrate primitives via `XAgentDatabase`/`XSearchIndex`; DB MCP servers themselves may be OSS or thin custom (`[PROPOSED]`, deferred).
- **Web-search + web-scrape** — ADR 0020 explicitly allows **build-new OR adopt OSS**; Firecrawl is **removed**. Decision deferred to implementation; A17 commits only that the endpoints stay simple internally and rely on MCP-server-access OPA for control.
- Rationale: LiteLLM is the single credential/OAuth broker (ADR 0020) so per-service code stays thin; substrate abstraction (ADR 0041) keeps DB provisioning uniform across kind/AWS.

## 6. Functional Requirements
- **REQ-A17-01:** Each of the seven services (GitHub, Google Drive, Context7, OpenSearch, Postgres, MongoDB, web-search, web-scrape) MUST be registered as a namespaced `MCPServer` CRD using only the source-stated fields (`endpoint`, `authMode`, `credentialsRef`, `tags`, `scopes`, `visibility`).
- **REQ-A17-02:** GitHub MUST be provided in **both** system-credential and user-credential modes; the user-credential mode MUST broker the user's OAuth flow through LiteLLM (A1), not through a per-service auth scheme.
- **REQ-A17-03:** Google Drive MUST be provided in **both** system-credential and user-credential modes, following the same LiteLLM-brokered OAuth pattern as GitHub.
- **REQ-A17-04:** Context7 MUST be installed and registered as an `MCPServer` providing documentation/library lookup (system-credential).
- **REQ-A17-05:** The OpenSearch MCP server MUST provide a **system-mediated** mode against the platform's own OpenSearch (reusing `XSearchIndex`) AND an **agent/tenant/user-credentialed** mode for externally operated OpenSearch instances.
- **REQ-A17-06:** Postgres and MongoDB MCP access MUST resolve to databases provisioned by the **`XAgentDatabase`** XRD (`engine` postgres/mongodb), which composes `XPostgres`/`XMongoDocStore`; the same claim shape MUST work on kind and AWS.
- **REQ-A17-07:** Which agent/tenant/user resolves to which provisioned database MUST be decided by **RBAC + OPA** at admission and at request time, and MUST NOT be hard-coded into the `MCPServer` spec (ADR 0014).
- **REQ-A17-08:** Generic web-search and web-scrape MCP servers MUST be deployed in-cluster with **no internal RBAC/OPA stack**; access control (allow/deny by query, domain, or scrape target) MUST be enforced at the **MCP-server-access OPA layer**.
- **REQ-A17-09:** Every service's secret MUST be materialized via **ESO** into the secret referenced by the `MCPServer` `credentialsRef`; no secret may be embedded in the CRD or image.
- **REQ-A17-10:** Each service MUST be reachable by a Platform Agent **only** via CapabilitySet inclusion (`mcpServers[]`) + OPA — there MUST be no gateway-side bypass toggle; adding/removing a service is a CRD + ESO/OPA change.
- **REQ-A17-11:** A17 MUST contribute Rego to the OPA policy library for: MCP-server-access (web-search/scrape gating), database-assignment, and admission of A17 `MCPServer` CRDs.
- **REQ-A17-12:** The web-search/scrape OPA bundle MUST expose the structured **dry-run** mode (ADR 0038) so the policy simulator (A20) can evaluate it before rollout.
- **REQ-A17-13:** All MCP requests/responses for these services MUST emit audit on the **LiteLLM callback path** via the A18 adapter — no separate audit emission point.
- **REQ-A17-14:** Capability-registry changes for A17's `MCPServer` CRDs MUST surface as `platform.capability.*` (incl. `platform.capability.changed`) via B13.
- **REQ-A17-15:** A17 MUST function on both substrates: on **kind** Postgres/Mongo/OpenSearch resolve to in-cluster CloudNativePG/Bitnami/in-cluster OpenSearch; on **AWS** to RDS/DocumentDB-or-self-managed/AWS-managed OpenSearch — via the substrate Compositions, no caller change.
- **REQ-A17-16:** **Firecrawl MUST NOT be deployed** in v1.0; the generic web-search/scrape services fill its role.

## 7. Non-Functional Requirements
- **Security:** No secret in CRD/image (ESO only); unapproved MCP servers unreachable (REQ-A17-10); web-search/scrape egress controlled at the access layer; user-credential OAuth tokens brokered/stored by LiteLLM, not by A17. Egress from any A17 service to external destinations rides the Envoy egress proxy (ADR 0003) and is FQDN-allowlisted.
- **Multi-tenancy (§6.9):** All `MCPServer` CRDs and `XAgentDatabase` claims are **namespaced**; DB assignment is per agent/tenant/user via `scope`; OPA reads `platform_namespaces`/`capability_set_refs` to gate access. No cross-namespace DB reuse without policy.
- **Observability (§6.5):** Per-service Grafana dashboard (Crossplane `GrafanaDashboard` XR); MCP call telemetry via LiteLLM/Langfuse; audit via A18. Health uses only what the MCP protocol exposes (no synthetic health layer; backlog §1.9).
- **Scale:** Web-search/scrape servers stateless/horizontally scalable; `XAgentDatabase` provisioning is per-principal and must not create unbounded DBs without OPA/budget gating (`[PROPOSED]` quota at OPA, deferred).
- **Versioning (ADR 0030):** `MCPServer` instances pin the B13-served CRD major; XRD claim shapes follow ADR 0041 conversion-webhook lifecycle.

## 8. Cross-Cutting Deliverable Checklist
- Helm/manifests — **applicable** (MCPServer CRDs, ESO ExternalSecrets, web-search/scrape Deployments, XAgentDatabase claims).
- Per-product docs (10.5) — **applicable** (one per service: setup, modes, scopes).
- Runbook (10.7) — **applicable** (credential rotation, OAuth re-consent, DB-provisioning failure, scrape IP-block).
- Backup/restore — **applicable** (DB stores via XAgentDatabase; covered by substrate primitives).
- Alerts — **applicable** (`MCPServer` not-ready, ESO sync failure, web-search/scrape error rate, DB-provisioning failure). `[PROPOSED — not in source]` thresholds.
- Grafana dashboard (Crossplane XR) — **applicable** (per-service MCP usage + DB-provisioning dashboard).
- Headlamp plugin — **applicable** (contributes to the per-agent capability resolved-view + MCPServer inspection; framework is A9/A22).
- OPA/Rego integration — **applicable** (MCP-server-access, DB-assignment, CRD admission).
- Audit emission (ADR 0034) — **applicable, via LiteLLM path** (no separate emission point).
- Knative trigger flow — **applicable** — design the capability-change consumption flow; **`[PROPOSED — not in source]`** any new A17-specific trigger (no v1.0 trigger flow is mandated for A17 beyond capability-change surfacing).
- HolmesGPT toolset — **applicable** (web-search/scrape and GitHub/Drive lookups are natural HolmesGPT tools; gated by its CapabilitySet).
- 3-layer tests — **applicable** (Chainsaw: MCPServer + XAgentDatabase reconcile/admission; PyTest: web-search/scrape + OPA gating logic; Playwright: Headlamp capability view).
- Tutorials & how-tos — **applicable** ("add an MCP service to an agent's CapabilitySet"; "connect an agent to a provisioned Postgres").

## 9. Acceptance Criteria
- **AC-A17-01:** All seven services exist as namespaced `MCPServer` CRDs whose fields are a subset of {`endpoint`,`authMode`,`credentialsRef`,`tags`,`scopes`,`visibility`}; a schema check finds no invented fields. (→ REQ-A17-01, -16)
- **AC-A17-02:** An agent with GitHub user-credential mode triggers a LiteLLM-brokered OAuth flow and obtains user-scoped access; system-credential mode uses the platform identity. (→ REQ-A17-02)
- **AC-A17-03:** Google Drive exhibits both modes equivalently. (→ REQ-A17-03)
- **AC-A17-04:** Context7 is registered and answers a documentation/library lookup through the MCP path. (→ REQ-A17-04)
- **AC-A17-05:** The OpenSearch MCP serves a query in system-mediated mode against the platform OpenSearch AND, with credentials supplied, against an external instance. (→ REQ-A17-05)
- **AC-A17-06:** A `Postgres`/`AgentDatabase` claim provisions a database, role, and grants; the same claim YAML applies unchanged on kind and AWS. (→ REQ-A17-06, -15)
- **AC-A17-07:** An agent not authorized by RBAC+OPA for a given database is denied at request time even though the `MCPServer` endpoint is reachable. (→ REQ-A17-07)
- **AC-A17-08:** A web-scrape request to a policy-blocked domain is denied at the MCP-server-access OPA layer; an allowed domain succeeds. (→ REQ-A17-08)
- **AC-A17-09:** Removing a service's ESO `ExternalSecret` causes its `credentialsRef` secret to disappear and the service to fail closed; no credential is found in any CRD or image. (→ REQ-A17-09)
- **AC-A17-10:** Deleting an `MCPServer` CRD makes the service unreachable from agents with no gateway-side override available. (→ REQ-A17-10)
- **AC-A17-11:** A17's Rego bundles for MCP-server-access, DB-assignment, and CRD admission load into OPA and pass their unit tests. (→ REQ-A17-11)
- **AC-A17-12:** The policy simulator (A20) evaluates the web-search/scrape bundle in dry-run and returns a per-layer decision with `simulated: true` and no enforcement effect. (→ REQ-A17-12)
- **AC-A17-13:** An MCP call to any A17 service produces an audit event on the LiteLLM callback path captured by the A18 adapter; no A17 component writes audit directly. (→ REQ-A17-13)
- **AC-A17-14:** Creating/updating/deleting an A17 `MCPServer` CRD produces a `platform.capability.*` event (incl. `platform.capability.changed`). (→ REQ-A17-14)
- **AC-A17-15:** The full A17 set installs and functions on a kind cluster with no cloud-managed services. (→ REQ-A17-15)
- **AC-A17-16:** No Firecrawl workload is present in any A17 manifest or running install. (→ REQ-A17-16)

## 10. Risks & Open Questions
- **R1 (high):** `authMode` enum lacks a **system-mediated** value in interface-contract §1.4, yet OpenSearch requires it. `[PROPOSED — not in source]`. Mitigation: coordinate the enum with B13 before OpenSearch lands; blast radius high (CRD contract).
- **R2 (med):** Per-service OAuth scopes, secret shapes, revocation, and budget enforcement are **deferred** (backlog §1.11); A17 must pick provisional values tagged `[PROPOSED]`, risking churn when F1/B-track settle them.
- **R3 (med):** Web-scrape **IP-block / rate-limit** risk is acknowledged in ADR 0020; the commercial fallback is deferred to future-enhancements, so v1.0 scraping may degrade. Blast radius med (capability, not security).
- **R4 (med):** `XAgentDatabase` is the **first runtime-state XR** — unbounded per-principal DB creation could exhaust the substrate without quota/budget gating (not yet specified). `[PROPOSED]` OPA/budget cap.
- **R5 (low):** `XObjectStore`/archive capability-parity caveat does not affect A17 directly but the kind OpenSearch path may differ in behavior from AWS-managed; tests must cover both.
- **OQ1:** Are web-search/scrape built-new or OSS-adopted? Deferred; affects supply-chain (B22) review surface.
- **OQ2:** Do user-credential tokens persist in LiteLLM across sessions or re-consent each run? Deferred to A1/backlog §1.11.
- **OQ3:** Is B4 a hard build-order upstream for A17 despite absence from the CSV `upstream` cell? `[PROPOSED]` yes (XRD ownership).

## 11. References
- architecture-overview.md §6.1 (gateway / MCP broker), §6.8 (line 632; capability registries, CapabilitySet layering, `platform.capability.changed`), §6.6 (line 517; "A17 MCP services ride this path"), §7.3 (line 1157; MCP tool invocation), §14.1 (line 1683; A17 deliverable).
- ADR 0020 (initial MCP services set — the seven services, three credential modes, XAgentDatabase, MCP-server-access, Firecrawl removed). ADR 0013 (capability CRD / CapabilitySet). ADR 0009 / 0033 (OpenSearch dual-hosting). ADR 0014 (Postgres primary / DB assignment gating). ADR 0021 / 0041 (XR composition / substrate abstraction / connection secret). ADR 0038 (policy-simulator target for search/scrape bundle). ADR 0040 (Kargo promotes claims). ADR 0034 (audit). ADR 0030 / 0031 (versioning / event taxonomy).
- architecture-backlog.md §1.9 (MCP health deferred), §1.11 (A17 per-service detail deferred).
- Related pieces: A1 (LiteLLM), A7/B3/B16 (OPA), B13 (kopf reconciler), B4 (Crossplane XRDs), A11 (OpenSearch), A18 (audit), A20 (simulator).
