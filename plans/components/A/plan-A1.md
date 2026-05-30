# PLAN A1 — LiteLLM (gateway)

> spec: SPEC-A1 · kind: COMPONENT · tier: T0
> wave: W0 · estimate: XL
> upstream-pieces: [] · downstream-pieces: [A16, A17, A19, A20, B1, B2, B16, B6, B9, B13, B17, B18]

## 1. Implementation Strategy

Install OSS LiteLLM via Helm as a `replicas >= 2` Deployment, then layer the platform's governance surfaces onto it: the callbacks pipeline (registration contract here; logic in B2), the MCP/A2A/RAG registry (populated by B13, shipped as a subchart per ADR 0006), virtual-key/Keycloak-claim auth, OpenAI↔A2A translation, dynamic OPA-gated registration, and provider failover. Because A1 is W0 foundation with no upstreams, it is built first; downstream consumers are unblocked incrementally. Where B2/A7/A18/B13/A20 have not yet landed, A1 stands up **fakes** (a no-op callback, an in-memory registry seed, a stub audit sink, a stub OPA allow/deny) so the gateway is testable in isolation, and swaps them for the real components as they land. The OPA-bridge + audit callbacks are mandatory and ride here if A1 precedes A7/B2 (ADR 0002 / §14.2 B2).

## 2. Ordered Task List

- **TASK-01:** Helm install LiteLLM (`replicas >= 2`), ConfigMap-driven config — produces: gateway Deployment + chart — depends-on: []
- **TASK-02:** Wire Reloader to the ConfigMap for graceful rolling restart (ADR 0035) — produces: reload behavior — depends-on: [TASK-01]
- **TASK-03:** Configure callbacks pipeline registration (4 hook points) + stub callbacks — produces: pipeline contract + fakes — depends-on: [TASK-01]
- **TASK-04:** Virtual-key auth + Keycloak Platform-JWT claim verification — produces: inbound authn — depends-on: [TASK-01]
- **TASK-05:** OpenAI↔A2A translation + stream-chunk preservation — produces: model-facing surface — depends-on: [TASK-04]
- **TASK-06:** A2A endpoint (internal + approved-external; no self-registration) — produces: A2A path — depends-on: [TASK-05]
- **TASK-07:** MCP gateway (tool namespacing, OAuth) against registry — produces: MCP path — depends-on: [TASK-04]
- **TASK-08:** Registry consumption surface for B13 (MCP/A2A/RAG) + seed fake — produces: registry contract — depends-on: [TASK-06, TASK-07]
- **TASK-09:** Dynamic OPA-gated registration on agent startup + deregistration — produces: dynamic-reg path — depends-on: [TASK-08]
- **TASK-10:** LLM provider failover logic (4 cases) + zero-provider admission reject — produces: routing/failover — depends-on: [TASK-04]
- **TASK-11:** MCP health surfacing (metrics/audit/traces + agent error path) — produces: health signals — depends-on: [TASK-07]
- **TASK-12:** Audit emission via adapter (stub until A18) + OPA budget callback (stub until A7) — produces: audit + budget hooks — depends-on: [TASK-03]
- **TASK-13:** CloudEvent emission under `platform.gateway.*` / `platform.observability.*` / `platform.audit.*` — produces: events — depends-on: [TASK-12]
- **TASK-14:** Dry-run decision shape for A20 (`simulated: true`) — produces: simulator hook — depends-on: [TASK-12]
- **TASK-15:** Package B13 kopf operator as subchart (coordinate with B13) — produces: single ArgoCD release — depends-on: [TASK-08]
- **TASK-16:** Admin API `/v1/...` versioning surface — produces: versioned admin API — depends-on: [TASK-01]
- **TASK-17:** §14.1 cross-cutting deliverables (docs, runbook, alerts, dashboard XR, Headlamp integration, HolmesGPT toolset, tutorials) — produces: deliverable set — depends-on: [TASK-13]
- **TASK-18:** 3-layer test suite mapping all ACs — produces: tests — depends-on: [TASK-17]

Critical path: TASK-01 → 04 → 05 → 06 → 08 → 09 → 15 → 18.

## 3. Dependency Map

### 3.1 Upstream pieces that must ship first (HARD)
None (W0 foundation). OSS LiteLLM + k8-platform baseline only.

### 3.2 Downstream pieces blocked on this
A16, A17, A19, A20, B1, B2, B6, B9, B13, B16, B17, B18.

### 3.3 Continuous (non-blocking) inputs
- B13 (kopf operator) — co-developed; subchart packaging needs coordination but A1 ships with a fake registry seed first.
- B2 (callbacks), A7 (OPA), A18 (audit) — A1 uses fakes/stubs until they land; mandatory audit + OPA-bridge callbacks ride A1 if it precedes them.
- B14 test framework, B22 threat model — continuous inputs (not wave-depth-setting).

## 4. Parallelizable Subtasks

- Group α (after TASK-01): TASK-02, TASK-03, TASK-04, TASK-16 independent.
- Group β (after TASK-04): TASK-05→06 (A2A) ∥ TASK-07 (MCP) ∥ TASK-10 (failover).
- Group γ (after TASK-08): TASK-09 (dynamic reg) ∥ TASK-15 (subchart packaging).
- TASK-11, TASK-12→13→14 fan out after their inputs.

## 5. Test Strategy

| AC | Layer | Fixtures / fakes |
|---|---|---|
| AC-A1-01,02,08,09,11 | Chainsaw | kind cluster; fake B13 CRDs; Reloader |
| AC-A1-03,04,05,06,07,10,12,13,14,15,16,17,18 | PyTest | stub OPA (allow/deny), stub audit endpoint, fake registry seed, fake provider endpoints, fake MCP server |
| Headlamp capability view, tutorials | Playwright | seeded CRDs; logged-in admin session |

Fakes needed for not-yet-landed upstreams: stub audit endpoint (A18), stub OPA decision service (A7), fake B13 registry (B13), fake Keycloak JWT issuer (claims schema only).

## 6. PR / Branch Mapping

### 6.1 Stack position — base branch = `wave/0`.
### 6.2 PR — `piece/A1-litellm-gateway` → base `wave/0`; carries spec-A1.md + plan-A1.md.
### 6.3 Merge order — W0 siblings (A2, A3, A4, A6, A7, A9, A11, A13, A14, A15, B22) independent; A1 ↔ B13 coordinate subchart packaging but spec/plan PRs are independent; `wave/0` rolls up to main.

## 7. Effort Estimate

- S: TASK-02, TASK-16. M: TASK-01, TASK-03, TASK-04, TASK-07, TASK-10, TASK-11, TASK-12, TASK-13, TASK-14, TASK-15. L: TASK-05, TASK-06, TASK-08, TASK-09, TASK-17, TASK-18.
- Rollup: **XL** (matches piece-index). Critical path ≈ TASK-01→04→05→06→08→09→15→18.

## 8. Rollback / Reversibility

Back out by reverting the LiteLLM Helm release (single ArgoCD app; subchart removes B13 with it). Because the gateway holds no system-of-record state (CRDs in Git are the source of truth; spend/key state is rebuildable by re-reconcile), rollback loses only in-flight spend counters. **Reverting A1 breaks essentially the entire data plane** (all LLM/MCP/A2A traffic): A16, A17, A19, A20, B6-using agents, B17/B18 — so reversion is a platform-down event, not a routine rollback. Mitigate with blue/green at the Helm-release level rather than delete-and-reapply.
