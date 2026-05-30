# PLAN ADR-0007 — LibreChat (configured) as the chat frontend [PROPOSED]

> spec: SPEC-ADR-0007 · kind: ADR · tier: T1
> wave: authoring-parallel · estimate: S
> upstream-pieces: [A1;A16] · downstream-pieces: [B1]

## 1. Implementation Strategy
Enforcement/verification map. ADR 0007 is enforced by component A8 (LibreChat installed unmodified + the locked `librechat.yaml`, upload→`Memory` wiring, Keycloak SSO), with claims-scoped visibility realized by B1 and the upload OPA policy by A7/B16. Conformance is proven by: (a) an unmodified-image + config-only check; (b) UI surface assertions (no agents/plugins/MCP); (c) JWT-claim-scoped endpoint visibility e2e; (d) the three-outcome upload-policy test; (e) a default-permissive (not RBAC-gated) upload assertion. No new product build belongs to this ADR — it is a constraint map over A8/A1/A16/B1.

## 2. Ordered Task List
- TASK-01: Map each REQ to the enforcing component piece — produces: enforcement matrix — depends-on: []
- TASK-02: Specify unmodified-image + `librechat.yaml`-only check and disabled-surface UI assertions — produces: Playwright + image check (A8) — depends-on: [TASK-01]
- TASK-03: Specify JWT-claim-scoped endpoint-visibility e2e — produces: Playwright (A8/B1) — depends-on: [TASK-01]
- TASK-04: Specify upload→`Memory` wiring + three-outcome OPA upload-policy test (incl. scan-then-allow→approval) — produces: e2e + PyTest (A8/A7/B16) — depends-on: [TASK-01]
- TASK-05: Specify default-permissive (no RBAC allowlist) upload assertion + Interactive-Access-Agent default-endpoint check — produces: Playwright + PyTest (A8/A16) — depends-on: [TASK-01]

## 3. Dependency Map
### 3.1 Upstream that must ship first (HARD)
- A1 — LiteLLM endpoints LibreChat picks from.
- A16 — Interactive Access Agent default endpoint.
### 3.2 Downstream blocked on this
- B1 (SSO/auth proxy layer).
### 3.3 Continuous (non-blocking) inputs
- A7/B16 upload OPA bundle; A18 scan/audit; ADR 0029 claim schema; ADR 0038 simulator; ADR 0039 Headlamp editor; B14 test framework.

## 4. Parallelizable Subtasks
TASK-02, TASK-03, TASK-04, TASK-05 fan out independently after TASK-01.

## 5. Test Strategy
- AC-01/02/07 → Playwright + image check (config-only deploy; disabled surfaces; default endpoint).
- AC-03 → Playwright (claims-scoped endpoint visibility).
- AC-04/05 → e2e + PyTest (upload→`Memory`; forbidden/allowed-after-scanning outcomes).
- AC-06 → PyTest (default-permissive policy; no user allowlist).
Fixtures: stub Keycloak JWTs with varied `capability_set_refs`; fake `Memory` store until A10/ADR 0025; stub scanner.

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/authoring`
### 6.2 PR — `piece/ADR-0007-librechat` → base `wave/authoring`; carries spec + plan.
### 6.3 Merge order — independent of sibling ADR PRs; rolls up to main.

## 7. Effort Estimate
TASK-01..05 each S. Rollup: S. Critical path: TASK-01 → TASK-04.

## 8. Rollback / Reversibility
Backing out means re-opening the chat-UI choice (Open WebUI / custom). Because LibreChat is unmodified and config-driven, reverting the config is cheap; reverting the product is moderate (re-wire upload→`Memory` and SSO). Downstream breakage limited to B1 and end-user chat access. Moderate reversibility.
