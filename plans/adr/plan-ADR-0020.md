# PLAN ADR-0020 — Initial MCP services set [PROPOSED]

> spec: SPEC-ADR-0020 · kind: ADR · tier: T0
> wave: authoring-parallel · estimate: M
> upstream-pieces: [A17, A1, B13] · downstream-pieces: [A7, B4, A11, A16, A14, B10]

## 1. Implementation Strategy
Enforcement/verification map. ADR 0020 is enforced primarily by **A17** (registers the fixed eight-service set as `MCPServer` CRDs, wires ESO secrets, exercises the three credential modes), over **A1** (LiteLLM as the single MCP/OAuth broker + shared callback chain), **B13** (kopf operator reconciling the `MCPServer` CRDs), **A7** (OPA admission + access-layer gating for web search/scrape and database assignment), and **B4** (`XAgentDatabase`/`XPostgres`/`XMongoDocStore`/`XSearchIndex` Compositions). **A11** is the system-mediated OpenSearch target; **A16/A14/B10** compose against the set via CapabilitySet. Conformance is tested by proving all eight services registered (no Firecrawl), the three credential modes, RBAC+OPA database assignment, access-layer policy on web services, the uniform `XAgentDatabase` claim across substrates, and no gateway-side bypass.

## 2. Ordered Task List
- TASK-01: Map each REQ to its enforcing component piece ID — produces: enforcement matrix — depends-on: []
- TASK-02: Verify A17 registers all eight services as `MCPServer` CRDs and excludes Firecrawl — produces: A17 conformance checklist — depends-on: [TASK-01]
- TASK-03: Verify the three credential modes (system / user-OAuth / system-mediated) at A17/A1 — produces: credential-mode trace — depends-on: [TASK-02]
- TASK-04: Verify `XAgentDatabase` composes `XPostgres`/`XMongoDocStore` per substrate with one claim shape (ADR 0041) — produces: B4 XRD trace — depends-on: [TASK-01]
- TASK-05: Verify RBAC+OPA database assignment at admission/request time (not static MCPServer field) — produces: A7 gating note — depends-on: [TASK-04]
- TASK-06: Verify web-search/scrape access-layer OPA policy + simple endpoints (ADR 0038 simulator target) — produces: access-layer conformance — depends-on: [TASK-02]
- TASK-07: Verify ESO secret sourcing + LiteLLM single-broker callback chain — produces: secret/broker trace — depends-on: [TASK-02]
- TASK-08: Verify no gateway-side bypass + CapabilitySet-gated agent access (A16/A14/B10) — produces: no-bypass audit — depends-on: [TASK-02]

## 3. Dependency Map
### 3.1 Upstream pieces that must ship first (HARD)
- A17 (initial MCP services), A1 (LiteLLM broker), B13 (kopf operator reconciling `MCPServer`).
### 3.2 Downstream pieces blocked on this
- A7 (gating), B4 (XRDs), A11 (system-mediated target), A16/A14/B10 (composing agents).
### 3.3 Continuous (non-blocking) inputs
- ADR 0041 (substrate abstraction), ADR 0040 (Kargo promotes claims), ADR 0038 (policy simulator), B22 threat model.

## 4. Parallelizable Subtasks
TASK-04, TASK-06, TASK-07, TASK-08 fan out once TASK-02 lands. TASK-03 follows TASK-02; TASK-05 follows TASK-04.

## 5. Test Strategy
- AC-01 → Chainsaw: all eight services as reconciled `MCPServer` CRDs; no Firecrawl present.
- AC-02 → PyTest+Playwright: GitHub/Google Drive in both system- and user-credential (brokered OAuth) modes.
- AC-03 → Chainsaw: OpenSearch MCP reaches platform instance (system-mediated via `XSearchIndex`) and external (credentialed).
- AC-04 → Chainsaw: `XAgentDatabase` claim provisions db+role+grants composing `XPostgres`/`XMongoDocStore`; same shape kind/AWS.
- AC-05 → PyTest: database assignment denied/permitted by RBAC+OPA, not a static field.
- AC-06 → PyTest: OPA at access layer blocks a disallowed query/scrape target; endpoint stays policy-free.
- AC-07 → Chainsaw: every MCP call routes through LiteLLM, producing audit/OPA/Langfuse records.
- AC-08 → PyTest: each `MCPServer.credentialsRef` resolves to an ESO secret.
- AC-09 → Chainsaw: unapproved MCP server unreachable; new service requires CRD+wiring, not a toggle.
- AC-10 → Chainsaw: Interactive Access Agent/HolmesGPT/Coach reach a service only when CapabilitySet includes it.
- Fixtures/fakes: stub ESO secrets, OAuth broker, CloudNativePG/Bitnami/OpenSearch substrates for not-yet-landed B4.

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/authoring`
### 6.2 PR — `piece/ADR-0020-initial-mcp-services` → base `wave/authoring`; carries spec + plan.
### 6.3 Merge order — independent of sibling ADR specs; rolls up to main. Coordinate with ADR 0041 (substrate) + ADR 0013 (capability) specs.

## 7. Effort Estimate
M overall (T0, eight services + three credential modes + XRD). Per-task: TASK-01 S, 02 M, 03 M, 04 M, 05 S, 06 M, 07 S, 08 S. Critical path: TASK-01 → 02 → 04 → 05.

## 8. Rollback / Reversibility
Decision record; back out by reverting spec+plan. If reverted, A17 loses the fixed-set + credential-mode contract and B4 loses the `XAgentDatabase` runtime-state precedent; per-service detail remains design-time (architecture-backlog §1.11) regardless. No runtime artifact is deleted. Removing a service means deleting its `MCPServer` CRD + wiring; agents lose it via CapabilitySet re-resolution.
