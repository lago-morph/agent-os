# PLAN V6-01 — Gateway architecture `[PROPOSED]`

> spec: SPEC-V6-01 · kind: VIEW · tier: T0
> wave: authoring-parallel · estimate: S
> upstream-pieces: [] · downstream-pieces: []

## 1. Implementation Strategy
This view is realized, not built. The realization map: **A1 (LiteLLM)** lands first in W0 as the foundational chokepoint; **B13 (kopf operator)** lands in W1 as a LiteLLM subchart (ADR 0006) to drive the capability registry; **B2 (custom callbacks)** lands in W1 to wire PII/OPA/audit/Langfuse into the hook chain; **B1 (SSO/auth proxy)** lands in W1 to front the gateway with Keycloak claims. The view holds when the chokepoint, callback, registry-only-via-CRD, and rolling-restart invariants are all enforced end-to-end. Verification is by tracing one request through every layer and by negative tests (bypass attempts, unreconciled capabilities).

## 2. Ordered Task List
- **TASK-01:** Confirm A1 LiteLLM install establishes the single chokepoint with `replicas >= 2` and ADR 0035 deployment shape — produces: chokepoint conformance checklist — depends-on: []
- **TASK-02:** Confirm B13 reconciles all eight capability CRDs into the LiteLLM admin API and ships as a subchart (ADR 0006) — produces: registry-contract checklist — depends-on: [TASK-01]
- **TASK-03:** Confirm B2 callback chain fires PII/OPA/audit/Langfuse at pre/post/success/failure hooks — produces: callback-conformance checklist — depends-on: [TASK-01]
- **TASK-04:** Confirm B1 supplies Keycloak claims to virtual-key auth — produces: auth-binding checklist — depends-on: [TASK-01]
- **TASK-05:** Define end-to-end view verification: bypass audit, failover matrix, dynamic-registration OPA gate, streaming, initial-MCP-set reachability — produces: view acceptance suite mapping (AC-01..11) — depends-on: [TASK-02, TASK-03, TASK-04]

## 3. Dependency Map
### 3.1 Upstream pieces that must ship first (HARD)
- A1 — the LiteLLM proxy (chokepoint, router, MCP/A2A endpoints, translation).
- B13 — capability-CRD reconciliation into the gateway (consumed: registry population).
- B2 — callback chain (consumed: PII/OPA/audit/Langfuse enforcement points).
- B1 — Keycloak claim injection (consumed: virtual-key auth context).
### 3.2 Downstream pieces blocked on this
- None directly (view imposes constraints; A16, A17, A19, A20, B6, B17, B18 consume the gateway per CSV but bind to component specs).
### 3.3 Continuous (non-blocking) inputs
- B14 test framework; B22 threat model (capability-escape / OPA-bypass attack patterns shape REQ-05/06).

## 4. Parallelizable Subtasks
- TASK-03 and TASK-04 run concurrently once TASK-01 is green; TASK-02 is independent of them (only depends on TASK-01). Fan-out group: {TASK-02, TASK-03, TASK-04}.

## 5. Test Strategy
- **Chainsaw (operator/CRD):** B13 reconcile of each capability CRD; unreconciled-capability-unreachable negative test (AC-03); zero-provider admission rejection (AC-07).
- **Playwright (UI/e2e):** LibreChat → Platform-Agent-as-model streaming (AC-04); Headlamp inspection of initial MCP set (AC-11).
- **PyTest (logic):** failover matrix (AC-07); callback ordering and adapter-not-direct-write (AC-02); MCP health surfacing (AC-08); dynamic-registration OPA gate (AC-06).
- Fixtures/fakes: fake LLM provider with toggleable availability; fake MCP backend with health signals; stub OPA returning allow/deny; fake audit endpoint capturing posts.

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/auth` (views authoring band)
### 6.2 PR — `piece/V6-01-gateway-architecture` → base `wave/auth`; carries spec-V6-01.md + plan-V6-01.md
### 6.3 Merge order — independent of sibling view PRs; rolls up to main with the other V6-0x views.

## 7. Effort Estimate
- TASK-01 S · TASK-02 S · TASK-03 S · TASK-04 S · TASK-05 M. Rollup: S (authoring). Critical path: TASK-01 → TASK-02/03/04 → TASK-05.

## 8. Rollback / Reversibility
Reverting the view doc has no runtime effect (authoring artifact). If an invariant is found wrong, amend the SPEC and re-flag `[PROPOSED]`; the realizing component SPECs (A1/B1/B2/B13) carry the enforceable change. No downstream code breaks from reverting the view itself.
