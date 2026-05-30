# PLAN ADR-0003 — Envoy egress proxy as CNI-agnostic egress control [PROPOSED]

> spec: SPEC-ADR-0003 · kind: ADR · tier: T1
> wave: authoring-parallel · estimate: S
> upstream-pieces: [] · downstream-pieces: [A5;A20;B7;B16;B20]

## 1. Implementation Strategy
Enforcement/verification map. ADR 0003 is enforced by component A6 (Envoy egress install + per-agent-class config as the only non-LiteLLM outbound chokepoint), with the OPA hook supplied by A7, egress audit by A18, and `EgressTarget` reconciliation by B13. Conformance is proven by: (a) a "no other egress path" network test (LiteLLM-or-Envoy only); (b) allowlist enforcement driven from `EgressTarget`/CapabilitySet; (c) per-connection OPA + audit assertions; (d) cross-substrate parity (kind vs EKS with a different CNI). No new build work belongs to this ADR — it is a constraint map over A6/B13/A7/A18.

## 2. Ordered Task List
- TASK-01: Map each REQ to the enforcing component piece — produces: enforcement matrix — depends-on: []
- TASK-02: Specify the "LiteLLM-or-Envoy only" egress-path test (no other external route) — produces: network conformance test (A6) — depends-on: [TASK-01]
- TASK-03: Specify allowlist-from-`EgressTarget` enforcement + Git→ArgoCD-only config drift check — produces: Chainsaw suite (A6/B13) — depends-on: [TASK-01]
- TASK-04: Specify per-connection OPA-query + single-audit-event assertions — produces: e2e + trace check (A7/A18) — depends-on: [TASK-01]
- TASK-05: Specify cross-substrate parity (kind + EKS-different-CNI) and cross-tenant A2A backstop — produces: multi-cluster e2e — depends-on: [TASK-01]

## 3. Dependency Map
### 3.1 Upstream that must ship first (HARD)
- None — A6 is a foundation component; reads `EgressTarget`/CapabilitySet as data.
### 3.2 Downstream blocked on this
- A5, A20, B7, B16, B20.
### 3.3 Continuous (non-blocking) inputs
- A7 OPA decision API; A18 audit adapter; B13 `EgressTarget` reconciler; B14 test framework; B22 threat model.

## 4. Parallelizable Subtasks
TASK-02, TASK-03, TASK-04, TASK-05 fan out independently after TASK-01.

## 5. Test Strategy
- AC-01/02/06/07 → Chainsaw (egress block/allow from CRD state; config-drift revert; cross-tenant backstop).
- AC-03/04 → PyTest (OPA-query trace assertion; one-audit-event-per-connection via adapter).
- AC-05 → Chainsaw multi-cluster (kind + EKS with different CNI run the same suite).
Fixtures: stub OPA decision (allow/deny) and fake audit adapter until A7/A18 land; fake `EgressTarget` set.

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/authoring`
### 6.2 PR — `piece/ADR-0003-envoy-egress` → base `wave/authoring`; carries spec + plan.
### 6.3 Merge order — independent of sibling ADR PRs; rolls up to main.

## 7. Effort Estimate
TASK-01 S · TASK-02 S · TASK-03 S · TASK-04 S · TASK-05 M (multi-cluster). Rollup: S. Critical path: TASK-01 → TASK-05.

## 8. Rollback / Reversibility
Backing out means re-opening the egress-control choice (Cilium L7) and accepting CNI coupling / per-environment policy forks. Downstream breakage: the "LiteLLM-or-Envoy only" invariant and every agent-class egress allowlist. Low reversibility once agents rely on declarative `EgressTarget` egress.
