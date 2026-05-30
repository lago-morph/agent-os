# PLAN A12 — OpenAPI→MCP converter

> spec: SPEC-A12 · kind: COMPONENT · tier: T1
> wave: consumer (real-build after A/B foundations) · estimate: M
> upstream-pieces: [] · downstream-pieces: []

## 1. Implementation Strategy
Select and evaluate an OSS OpenAPI→MCP converter, then wrap and harden it into a stateless platform
service: validate input specs (fail-closed), consume credentials by reference (ESO/`authConfigRef`),
synthesize an MCP server, and register it **through B13** (never the LiteLLM admin API directly) so
it inherits standard CapabilitySet/OPA/audit governance. Honor the `SyntheticMCPServer` XR contract
(B4) by populating the `mcpServerRef` back-link. Although the CSV marks A12 `consumer` with empty
build edges, it has real functional dependencies (B4, B13, A1, A7) — build against contract-faithful
mocks per the §10 mock-out process and re-run the same tests when the real pieces land.

## 2. Ordered Task List
- **TASK-01:** Evaluate + select OSS converter base; pin version — produces: ADR-style selection note + pinned dep — depends-on: [].
- **TASK-02:** Converter service skeleton + URL-path-versioned (`/v1/...`) HTTP API — produces: service scaffold — depends-on: [TASK-01].
- **TASK-03:** OpenAPI spec validation + fail-closed hardening — produces: validation layer — depends-on: [TASK-02].
- **TASK-04:** Synthesize MCP server + register via B13 `MCPServer` path — produces: synthesis+registration — depends-on: [TASK-03].
- **TASK-05:** `SyntheticMCPServer` XR contract honoring (consume refs, set `mcpServerRef`) — produces: XR integration — depends-on: [TASK-04].
- **TASK-06:** Credential-by-reference handling (`authConfigRef`/ESO) — produces: cred wiring — depends-on: [TASK-04].
- **TASK-07:** `platform.capability.*` events + audit emission + `LogLevel` honoring — produces: events/audit/toggle — depends-on: [TASK-04].
- **TASK-08:** `GrafanaDashboard` XR + alerts + OPA admission Rego — produces: dashboard/alerts/Rego — depends-on: [TASK-02].
- **TASK-09:** Three-layer tests + docs/runbook + the synthetic-MCP tutorial — produces: tests + docs — depends-on: [TASK-05, TASK-06, TASK-07, TASK-08].

## 3. Dependency Map
### 3.1 Upstream pieces that must ship first (HARD)
- None per CSV. Functional (mock until landed): B4 (`SyntheticMCPServer` XR/Composition),
  B13 (`MCPServer` reconciliation into LiteLLM), A1 (LiteLLM registry), A7 (OPA admission/runtime).
### 3.2 Downstream pieces blocked on this
- None per CSV. (Synthetic servers consumed by agents via CapabilitySet at runtime, not a build edge.)
### 3.3 Continuous (non-blocking) inputs
- B14 test framework; B22 threat-model standards; A17 as the curated-service consumer pattern.

## 4. Parallelizable Subtasks
- After TASK-02: TASK-08 runs parallel to the TASK-03→04 spine.
- After TASK-04: TASK-05, TASK-06, TASK-07 fan out independently.

## 5. Test Strategy
- **Chainsaw:** AC-A12-02 (XR reconcile + back-link), AC-A12-07 (LogLevel), AC-A12-09 (XR/alert), Chainsaw half of AC-A12-01.
- **Playwright:** none primary (no A12-owned UI; capability inspector is B5). 
- **PyTest:** AC-A12-01 (end-to-end synthesize+reachable), AC-A12-03 (no direct admin write), AC-A12-04 (fail-closed), AC-A12-05 (cred-by-ref), AC-A12-06 (events+audit), AC-A12-08 (URL versioning), AC-A12-10 (tutorial e2e).
- **Fixtures/fakes:** fake B13 `MCPServer` reconciler asserting it (not A12) writes LiteLLM; mock
  LiteLLM registry; stub OPA admission; sample valid + malformed OpenAPI specs; ESO secret-ref stub.

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/consumer` (rebased on the A/B foundation waves it mocks).
### 6.2 PR — `piece/A12-openapi-to-mcp` → base `wave/consumer`; carries spec-A12.md + plan-A12.md.
### 6.3 Merge order — independent of other consumer-wave siblings; real-build after B4/B13/A1/A7 land.

## 7. Effort Estimate
- TASK-01 S · TASK-02 M · TASK-03 M · TASK-04 M · TASK-05 S · TASK-06 S · TASK-07 M · TASK-08 S · TASK-09 M.
- Rollup: **M**. Critical path: TASK-01 → TASK-02 → TASK-03 → TASK-04 → TASK-09.

## 8. Rollback / Reversibility
Remove the converter service and any `SyntheticMCPServer` XRs. Synthetic MCP servers it produced are
plain `MCPServer` CRDs; deleting A12 does not strand them, but they can no longer be regenerated from
specs. No downstream build piece breaks (empty downstream); agents lose only the ability to mint new
synthetic MCP capabilities from OpenAPI.
