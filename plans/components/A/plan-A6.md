# PLAN A6 — agent-sandbox + Envoy egress proxy

> spec: SPEC-A6 · kind: COMPONENT · tier: T0
> wave: W0 · estimate: XL
> upstream-pieces: [] · downstream-pieces: [A5, A20, B16, B7, B20]

## 1. Implementation Strategy

Build the two coupled perimeter controls in parallel: (1) install the agent-sandbox operator and stand up the `Sandbox`/`SandboxTemplate` reconcile, gVisor/Kata runtime, warm-pool, hibernation, and skill-mount mechanics; (2) install Envoy via Helm as the single non-LiteLLM egress chokepoint, configured per agent class from `EgressTarget`-derived config (produced by B13), with the OPA hook, mTLS, audit-on-every-connection, and the dry-run surface. Because A6 is W0 foundation with no upstreams it is built first; ARK (A5), the SDK (B7), and PV access (B20) consume its runtime contract later. Where B13 (EgressTarget reconcile), A7 (OPA), and A18 (audit) have not landed, A6 uses a hand-seeded allowlist fixture, a stub OPA allow/deny endpoint, and a stub audit sink, swapping them as they land. The hardest correctness gate is proving **no egress bypass** on each substrate.

## 2. Ordered Task List

- **TASK-01:** Install agent-sandbox operator; reconcile `Sandbox`/`SandboxTemplate` — produces: sandbox operator — depends-on: []
- **TASK-02:** gVisor/Kata runtime wiring per `runtime` field — produces: kernel isolation — depends-on: [TASK-01]
- **TASK-03:** Warm-pool + hibernation + resourceLimits — produces: pod lifecycle — depends-on: [TASK-01]
- **TASK-04:** Skill init-container → skill-volume mount mechanics — produces: skill delivery — depends-on: [TASK-01]
- **TASK-05:** Install Envoy via Helm, per-agent-class config scaffold — produces: egress proxy — depends-on: []
- **TASK-06:** FQDN allowlist enforcement from EgressTarget-derived config (seed fixture until B13) — produces: allowlist — depends-on: [TASK-05]
- **TASK-07:** OPA runtime decision call at egress hook (stub OPA until A7) — produces: egress policy hook — depends-on: [TASK-06]
- **TASK-08:** mTLS + audit-on-every-connection via adapter (stub until A18) — produces: secure+audited egress — depends-on: [TASK-07]
- **TASK-09:** "No-bypass" network enforcement (pod can only egress via Envoy) — produces: perimeter guarantee — depends-on: [TASK-05]
- **TASK-10:** Sandbox lifecycle audit (create/destroy/hibernate) — produces: lifecycle audit — depends-on: [TASK-03, TASK-08]
- **TASK-11:** Envoy egress dry-run surface for A20 (`simulated: true`) — produces: simulator hook — depends-on: [TASK-07]
- **TASK-12:** `LogLevel` runtime toggle for Envoy (ADR 0035) — produces: dynamic toggle — depends-on: [TASK-05]
- **TASK-13:** `platform.security.*` escape/bypass signal emission — produces: security events — depends-on: [TASK-08, TASK-09]
- **TASK-14:** §14.1 deliverables (docs, runbook, alerts, dashboard XR, Headlamp integration, HolmesGPT toolset, tutorials) — produces: deliverable set — depends-on: [TASK-10, TASK-13]
- **TASK-15:** 3-layer test suite mapping all ACs (incl. two-substrate runs) — produces: tests — depends-on: [TASK-14]

Critical path: TASK-05 → 06 → 07 → 08 → 09 → 13 → 15 (egress perimeter); sandbox track (TASK-01→02/03/04→10) runs in parallel.

## 3. Dependency Map

### 3.1 Upstream pieces that must ship first (HARD)
None (W0 foundation). agent-sandbox + Envoy OSS + k8-platform baseline.

### 3.2 Downstream pieces blocked on this
A5 (ARK runs into Sandboxes), A20 (egress dry-run), B16 (egress Rego target), B7 (harness runtime), B20 (PV mount into sandbox).

### 3.3 Continuous (non-blocking) inputs
- B13 (kopf operator — `EgressTarget`→Envoy config) — co-developed; A6 uses a seeded allowlist fixture first.
- A7 (OPA) / B16 (egress Rego) — stub OPA until landed.
- A18 (audit endpoint) — stub until landed.
- B14 test framework, B22 threat model — continuous inputs (sandbox-escape / exfiltration attack patterns feed AC design).

## 4. Parallelizable Subtasks

- **Sandbox track** (TASK-01 → 02 ∥ 03 ∥ 04) and **Egress track** (TASK-05 → 06 → 07 → 08, with 09/12 forking off TASK-05) run fully in parallel — two independent fan-out groups.
- TASK-11 (dry-run) ∥ TASK-12 (LogLevel) after their inputs.
- TASK-10 (lifecycle audit) joins both tracks before TASK-14.

## 5. Test Strategy

| AC | Layer | Fixtures / fakes |
|---|---|---|
| AC-A6-01,02,03,04,10,12,13 | Chainsaw | kind + managed-K8s; gVisor/Kata runtime classes; Reloader/LogLevel |
| AC-A6-05,06,07,08,09,11,14 | PyTest | seeded EgressTarget allowlist (until B13); stub OPA; stub audit endpoint; bypass-probe harness |
| Headlamp sandbox/egress view | Playwright | seeded Sandbox + EgressTarget; admin session |

Fakes for not-yet-landed upstreams: B13-produced Envoy config (seed fixture), stub OPA decision endpoint (A7), stub audit endpoint (A18). The bypass-probe harness must run per substrate to satisfy AC-A6-05/13.

## 6. PR / Branch Mapping

### 6.1 Stack position — base branch = `wave/0`.
### 6.2 PR — `piece/A6-sandbox-envoy-egress` → base `wave/0`; carries spec-A6.md + plan-A6.md.
### 6.3 Merge order — W0 siblings independent; A6 ↔ B13 coordinate the EgressTarget→Envoy config contract but spec/plan PRs are independent; `wave/0` rolls up to main.

## 7. Effort Estimate

- S: TASK-12. M: TASK-01, TASK-02, TASK-03, TASK-04, TASK-05, TASK-06, TASK-10, TASK-11, TASK-13. L: TASK-07, TASK-08, TASK-09, TASK-14, TASK-15.
- Rollup: **XL** (matches piece-index — two coupled subsystems + two-substrate proof). Critical path ≈ TASK-05→06→07→08→09→13→15.

## 8. Rollback / Reversibility

Back out by reverting the agent-sandbox and Envoy Helm releases. A6 holds **no system-of-record state** (sandboxes ephemeral, config in Git), so rollback is clean. **Reverting A6 removes two of the four defense-in-depth layers** (kernel isolation + egress perimeter) and breaks the agent runtime: ARK (A5) has nowhere to run agents, harnesses (B7) lose their runtime, PV access (B20) loses its mount target, and the policy simulator (A20) loses the egress layer. Critically, reverting Envoy while agents still run would **remove the egress perimeter entirely** — so reversion must drain/stop agent workloads first, not just delete the proxy. Treat as a controlled maintenance action, not a routine rollback.
