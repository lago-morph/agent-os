# PLAN F4 — Security review

> spec: SPEC-F4 · kind: COMPONENT · tier: T2
> wave: authoring-parallel (consumes A/B; runs at end of v1.0) · estimate: M
> upstream-pieces: [B22, B16, A6, A7, B13, A18, A17] · downstream-pieces: []

## 1. Implementation Strategy

F4 verifies the shipped platform against B22's adversarial threat-model outputs (ADR 0027) and runs the operational security drills v1.0 names. The approach: treat B22 (security standards, mandatory test cases, dashboard signals, OPA targets) as the acceptance contract, then exercise each control — rotation, secret-handling, sandbox/capability escape, bundle coverage, audit integrity, claim enforcement — in a controlled throwaway cluster, producing a findings register routed to owning components and F6. No new build; manual bundle review (automated conflict analysis is deferred, future §13). Critical path: B22 contract ingestion → control drills (parallel) → coverage matrix + findings register → pen-test scope.

## 2. Ordered Task List

- **TASK-01:** Ingest B22 — extract mandatory test cases, security standards, dashboard signals, OPA policy targets; build the F4 verification checklist — produces: B22-derived checklist — depends-on: [].
- **TASK-02:** Credential rotation drill — `VirtualKey` `ttl`/reissue + ESO-managed secrets; assert new works, old invalidated, no outage — produces: rotation drill record — depends-on: [TASK-01].
- **TASK-03:** Secret-handling audit — trace AWS SM → ESO → workload; scan Git/logs/audit/traces for leakage (zero material) — produces: secret-handling audit — depends-on: [TASK-01].
- **TASK-04:** Sandbox + capability escape testing — adversarial probes vs gVisor/Kata + Envoy allowlist; capability-not-in-set call; assert deny + detect (`platform.security.*`) + audit — produces: escape-test record — depends-on: [TASK-01].
- **TASK-05:** OPA bundle review — map B16 bundle to B22 OPA targets (admission per CRD, gateway runtime, egress, RBAC-floor/OPA-restrictor, Headlamp gating); flag gaps/unreachable rules; record bundle version — produces: coverage matrix — depends-on: [TASK-01].
- **TASK-06:** Audit-integrity + emission-completeness — verify §6.6 emission points emit; attempt audit tamper (resisted, ADR 0034) — produces: audit-integrity record — depends-on: [TASK-01].
- **TASK-07:** Claim enforcement + DoS-class probe — forged/over-broad Keycloak claim rejected (ADR 0029); budget/capability-exhaustion degrades gracefully (coord. F5) — produces: claim + DoS record — depends-on: [TASK-01].
- **TASK-08:** Pen-test scope document — §6.6 trust boundaries, B22 adversary classes, in/out-of-scope, rules of engagement (ADR 0033 deployment) — produces: pen-test scope — depends-on: [TASK-01].
- **TASK-09:** Findings register — prioritized findings + residual risk, each assigned to owning component; route to F6 — depends-on: [TASK-02, TASK-03, TASK-04, TASK-05, TASK-06, TASK-07].
- **TASK-10:** Runbooks (credential rotation / secret-leak / sandbox-escape response) feeding F6; how-tos; verify B22 mandatory dashboard signals exist — depends-on: [TASK-09].
- **TASK-11:** 3-layer tests (Chainsaw admission-/capability-deny; PyTest rotation + leak-scan; Playwright claim/Headlamp gating) mapping ACs — depends-on: [TASK-02, TASK-04, TASK-05, TASK-07].

## 3. Dependency Map

### 3.1 Upstream that must ship first (HARD)
- **B22** — threat-model design spec = F4's acceptance contract (standards, test cases, dashboard signals, OPA targets).
- **B16** — OPA bundle under review. **A6** — sandbox/Envoy under attack. **A7** — OPA/Gatekeeper probed.
- **B13** — `VirtualKey`/capability CRDs for rotation + capability-escape. **A18** — audit integrity. **A17** — MCP-service credentials audited.

### 3.2 Downstream blocked on this
- None (terminal). F6 consumes findings + security runbooks; owning components consume assigned findings.

### 3.3 Continuous (non-blocking) inputs
- **F5** DoS/load baselines (TASK-07). **B1/Keycloak** claim issuance. **B3** OPA framework. **B14** harness. **A14/HolmesGPT** as detection-signal consumer.

## 4. Parallelizable Subtasks
- After TASK-01: TASK-02..08 are a wide independent fan-out (rotation / secret-audit / escape / bundle / audit-integrity / claim-DoS / pen-scope) — different control surfaces.
- TASK-10 (runbooks/dashboard verify) and TASK-11 (tests) parallelize after the findings register.

## 5. Test Strategy
- **Chainsaw (operator/CRD):** AC-F4-04 (capability-escape denied + audited), AC-F4-03 (sandbox-escape denied + `platform.security.*`), admission-deny per CRD (bundle coverage spot-checks).
- **PyTest (logic):** AC-F4-01 (rotation propagation + old-invalidation), AC-F4-02 (zero secret material across Git/logs/audit/traces), AC-F4-05 (coverage matrix completeness), AC-F4-06 (emission points + tamper detection), AC-F4-08 (findings register assigns owners).
- **Playwright (UI/e2e):** AC-F4-09 (forged/over-broad claim rejected at Headlamp/LibreChat surfaces), Headlamp action-gating checks.
- **Fixtures/fakes:** throwaway cluster for adversarial probes (NFR containment); forged-claim generator; secret-leak scanner; B22-checklist fixture; DoS load generator (shared with F5). Pen-test *execution* is out of scope — scope doc only (OQ1).

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/F` (B22, B16, A6, A7, B13, A18, A17 merged to main).
### 6.2 PR — `piece/F4-security-review` → base `wave/F`; carries SPEC-F4 + PLAN-F4.
### 6.3 Merge order — independent of F-siblings; findings assigned out to owning components as their own follow-up PRs.

## 7. Effort Estimate
- B22 ingestion + drills (TASK-01..07): M (adversarial probes + secret audit are the bulk).
- Pen-scope + findings + runbooks + tests (TASK-08..11): M.
- **Rollup: M** (matches CSV). **Critical path:** TASK-01 → 04 (escape) → 09 (findings) → 10.

## 8. Rollback / Reversibility
F4 is review/drills — it makes no lasting platform change; rollback is tearing down the throwaway cluster and discarding probe artifacts (the findings register, coverage matrix, pen-test scope persist as artifacts). **No downstream break** on revert (F4 is terminal); reverting only removes the security-review evidence, re-opening the production-readiness gate. Remediation PRs spawned from findings are owned by their components and roll back independently.
