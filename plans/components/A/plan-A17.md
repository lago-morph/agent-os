# PLAN A17 — Initial MCP services integration

> spec: SPEC-A17 · kind: COMPONENT · tier: T0
> wave: W2 · estimate: XL
> upstream-pieces: [A1, A7, B13] · downstream-pieces: []

## 1. Implementation Strategy

Land the seven MCP services as namespaced `MCPServer` CRD instances plus their ESO/OPA/CapabilitySet wiring, exercising all three credential modes (system, user, system-mediated) so the patterns are proven from day one. Build credential-mode-first: nail GitHub system+user-credential modes against the LiteLLM OAuth broker (the load-bearing delegated-auth pattern every later service follows), then replicate to Google Drive, add Context7 (simplest), then the substrate-backed services (OpenSearch system-mediated via `SearchIndex`, Postgres/MongoDB via `AgentDatabase`), then the generic web-search/scrape servers with MCP-server-access OPA as the only control point. Mock B4's XRDs (`AgentDatabase`/`Postgres`/`MongoDocStore`/`SearchIndex`) and B13's reconciler until they land. Critical path runs through the user-credential OAuth pattern and the `AgentDatabase` runtime-provisioning proof (the first runtime-state XR). No bypass path by construction: everything is a CRD + ESO + OPA change.

## 2. Ordered Task List

- **TASK-01:** Author the `MCPServer` CRD-instance manifest pattern (source-stated fields only) + Helm packaging — produces: MCPServer instance template — depends-on: [] (B13 CRD schema available/mocked).
- **TASK-02:** ESO `SecretStore`/`ExternalSecret` wiring per service materializing `credentialsRef` — produces: ESO secret plumbing — depends-on: [TASK-01].
- **TASK-03:** GitHub MCP — system-credential + user-credential modes via LiteLLM OAuth broker — produces: GitHub MCPServer(s) — depends-on: [TASK-01, TASK-02]. (Mock A1 OAuth broker until wired.)
- **TASK-04:** Google Drive MCP — system + user-credential, mirroring GitHub — produces: Drive MCPServer(s) — depends-on: [TASK-03].
- **TASK-05:** Context7 MCP — install + register (system-credential) — produces: Context7 MCPServer — depends-on: [TASK-01, TASK-02].
- **TASK-06:** OpenSearch MCP — system-mediated (reuse `SearchIndex`) + external-credentialed mode — produces: OpenSearch MCPServer(s) — depends-on: [TASK-01, TASK-02]. (Resolve the `[PROPOSED]` system-mediated `authMode` enum with B13.)
- **TASK-07:** `AgentDatabase` claims for Postgres + MongoDB (compose `Postgres`/`MongoDocStore`); wire MCP servers to the connection secret — produces: DB MCPServers + AgentDatabase XRs — depends-on: [TASK-01, TASK-02]. (Mock B4 XRDs until landed.)
- **TASK-08:** Web-search + web-scrape in-cluster MCP servers (build-new or OSS-adopt; no internal RBAC/OPA) — produces: 2 generic MCP servers — depends-on: [TASK-01].
- **TASK-09:** OPA Rego — MCP-server-access (search/scrape query/domain/target gating), DB-assignment, A17 CRD admission; expose dry-run (ADR 0038) — produces: A17 Rego bundles — depends-on: [TASK-07, TASK-08] (B3 contract).
- **TASK-10:** CapabilitySet wiring + Headlamp per-agent resolved-view contribution — produces: CapabilitySet references + Headlamp view — depends-on: [TASK-03..08].
- **TASK-11:** Cross-cutting — `GrafanaDashboard` XR, alerts, runbooks, HolmesGPT toolset (search/scrape/GitHub/Drive tools) — depends-on: [TASK-09, TASK-10].
- **TASK-12:** 3-layer tests mapping all ACs — depends-on: [TASK-09, TASK-10].
- **TASK-13:** Tutorials & how-tos ("add an MCP service to a CapabilitySet"; "connect an agent to a provisioned Postgres") — depends-on: [TASK-11].

## 3. Dependency Map

### 3.1 Upstream that must ship first (HARD)
- **A1 (LiteLLM)** — MCP broker path + user-credential OAuth brokering; single credential/OAuth broker.
- **A7 (OPA/Gatekeeper)** — engine for MCP-server-access, DB-assignment, CRD admission.
- **B13 (kopf operator)** — reconciles `MCPServer` CRDs into LiteLLM; emits `platform.capability.changed`.
- **B4 (Crossplane Compositions)** — owns `AgentDatabase`/`Postgres`/`MongoDocStore`/`SearchIndex`. `[PROPOSED]` hard dep (not in CSV upstream). Mockable until landed.

### 3.2 Downstream blocked on this
- None in CSV. De-facto: A16/A14/B10 compose against A17's capability surface via CapabilitySet.

### 3.3 Continuous (non-blocking) inputs
- **A11** OpenSearch (system-mediated target). **A18** audit adapter (audit rides LiteLLM path). **B3/B16** OPA framework/content. **B14** test framework. **B22** threat model (MCP egress + supply-chain for OSS adoptions).

## 4. Parallelizable Subtasks
- After TASK-02: TASK-03 (GitHub), TASK-05 (Context7), TASK-06 (OpenSearch), TASK-07 (DBs), TASK-08 (search/scrape) are independent fan-out groups, one per service cluster. TASK-04 follows TASK-03 (shares the OAuth pattern).
- TASK-11 (cross-cutting) and TASK-13 (docs) parallelize once their services exist.

## 5. Test Strategy
- **Chainsaw (operator/CRD):** AC-A17-01 (no invented fields), -06 (AgentDatabase reconcile, kind+AWS), -10 (delete→unreachable), -14 (capability event), -15 (kind install), -16 (no Firecrawl).
- **PyTest (logic):** AC-A17-02/-03 (OAuth modes), -05 (system-mediated vs external), -07 (RBAC+OPA DB gating), -08 (search/scrape OPA allow/deny), -09 (ESO fail-closed), -11 (Rego unit tests), -12 (simulator dry-run), -13 (audit on LiteLLM path).
- **Playwright (UI/e2e):** AC-A17-04 (Context7 lookup e2e), Headlamp capability resolved-view.
- **Fakes/fixtures:** mock B4 XRDs (AgentDatabase/Postgres/Mongo/SearchIndex), mock B13 reconciler + A1 OAuth broker, fake external OpenSearch. Re-run against real surfaces as they land. AWS-only DB/archive modes need a cloud integration fixture (kind cannot exercise them).

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/2` (contains A1, A7, B13 specs A17 builds on).
### 6.2 PR — `piece/A17-initial-mcp-services` → base `wave/2`; carries SPEC-A17 + PLAN-A17.
### 6.3 Merge order — independent of W2 siblings; wave/2 rolls up to main. B4 dependency tracked as a cross-wave (W1) prerequisite.

## 7. Effort Estimate
- Credential modes (TASK-01..06): L (OAuth broker integration + system-mediated enum are the bulk).
- Substrate DBs + generic servers (TASK-07, -08): M (parallel; gated on B4 mocks).
- OPA + CapabilitySet + cross-cutting + tests + docs (TASK-09..13): M.
- **Rollup: XL** (matches CSV; seven services × three modes × dual substrate). **Critical path:** TASK-01 → 02 → 03 (user-cred OAuth) → 07 (`AgentDatabase` runtime proof) → 09 → 12.

## 8. Rollback / Reversibility
Back out by deleting the A17 `MCPServer` CRDs (+ ESO ExternalSecrets); agents immediately lose reachability (no gateway-side state to clean up — the bypass-free design makes rollback clean). `AgentDatabase` claims must be deleted to deprovision databases (destructive to any agent data in them — drain first). **Downstream break:** any agent whose CapabilitySet referenced an A17 service loses that capability (perimeter checks fail closed; no security exposure). OSS-adopted server images are removed with their Deployments.
