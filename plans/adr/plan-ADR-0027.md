# PLAN ADR-0027 — Threat-model scope: v1.0 stance and B22 prerequisite [PROPOSED]

> spec: SPEC-ADR-0027 · kind: ADR · tier: T1
> wave: authoring-parallel · estimate: M
> upstream-pieces: [B22] · downstream-pieces: [B16, B3, A7, A6, A18, A20]

## 1. Implementation Strategy
Enforcement/verification map. ADR 0027 is enforced by the existence and fan-out of **B22** (the adversarial threat-model design spec) and by a process gate: every component completes a security review against B22's standards before it is implementation-complete. B22's output flows into **B16/B3** (OPA policy targets), **A7** (Gatekeeper), **A6** (sandbox + Envoy egress), **A18** (the audit/observability signals B22's per-attack mapping requires), and **A20** (policy simulator validating against B22 targets). Conformance is verified by checking B22 ships before the first A/B implementation wave, covers the §6.6 minimum set, updates per-component deliverable lists and B16 targets, refines (not replaces) the ADR 0002/0003/0018 layers, and that no component is marked complete without a passing B22-standard security review.

## 2. Ordered Task List
- TASK-01: Map each REQ to its enforcing surface (B22 publication, fan-out into deliverables/B16, per-component security-review gate) — produces: enforcement matrix — depends-on: []
- TASK-02: Verify B22 publishes before the first A/B implementation wave merges — produces: sequencing check — depends-on: [TASK-01]
- TASK-03: Verify B22 covers the §6.6 minimum (adversary classes, assets, trust boundaries, 8 attack patterns) — produces: coverage audit — depends-on: [TASK-02]
- TASK-04: Verify B22 output updates Workstream-A deliverable lists and B16 OPA policy targets — produces: fan-out audit — depends-on: [TASK-03]
- TASK-05: Verify each component's §8 checklist includes a B22-standard security review as a completion gate — produces: gate audit — depends-on: [TASK-01]
- TASK-06: Verify B22 controls extend (not replace) the ADR 0002/0003/0018 layers — produces: layering check — depends-on: [TASK-03]
- TASK-07: Verify observability alert rules / dashboard panels trace to B22's per-attack mapping — produces: A18/A20 signal trace — depends-on: [TASK-04]

## 3. Dependency Map
### 3.1 Upstream pieces that must ship first (HARD)
- B22 — the threat-model design spec is the prerequisite this ADR makes load-bearing.
### 3.2 Downstream pieces blocked on this
- B16/B3 (OPA policy targets), A7/A6 (hardening), A18/A20 (signals/simulation) — and every component's security-complete state.
### 3.3 Continuous (non-blocking) inputs
- ADR 0002/0003/0018 (defense-in-depth layers), ADR 0034 (audit substrate for signals).

## 4. Parallelizable Subtasks
TASK-03 and TASK-05 fan out once TASK-01/02 land. TASK-04→TASK-07 and TASK-06 run independently after TASK-03.

## 5. Test Strategy
- AC-01 → doc-lint: v1.0 controls documented as optimized against unintentional access excess.
- AC-02 → sequencing/CI check: B22 merged before first A/B implementation wave.
- AC-03 → doc-lint: deliverable lists + B16 targets reflect B22 standards.
- AC-04 → process gate: no component marked complete without a passing B22-standard security review.
- AC-05 → coverage audit: all 8 §6.6 attack patterns + adversary/asset/boundary present in B22.
- AC-06 → traceability: B22 controls map onto (extend) ADR 0002/0003/0018 layers.
- AC-07 → trace: security alert rules / dashboard panels back to B22 per-attack mapping.
- Fixtures/fakes: a B22 standards stub to validate fan-out wiring before B22 fully lands; this is a design-conformance audit, mostly doc/process-lint rather than runtime tests.

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/authoring`
### 6.2 PR — `piece/ADR-0027-threat-model-scope-b22` → base `wave/authoring`; carries spec + plan.
### 6.3 Merge order — independent of sibling ADR specs; rolls up to main.

## 7. Effort Estimate
M overall. Per-task: TASK-01 S, 02 S, 03 M, 04 M, 05 M, 06 S, 07 M. Critical path: TASK-01 → 02 → 03 → 04 → 07.

## 8. Rollback / Reversibility
Decision record; back out by reverting spec+plan. If reverted, the platform loses the pre-implementation security-standards gate; each component would invent its own threat assumptions and end-to-end validation would be lost. No runtime artifact is deleted, but the security-complete contract across both workstreams is voided — high blast radius, so reversal is strongly discouraged once components depend on B22.
