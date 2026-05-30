# PLAN ADR-0026 — Independent-cluster-install topology with no v1.0 federation [PROPOSED]

> spec: SPEC-ADR-0026 · kind: ADR · tier: T2
> wave: authoring-parallel · estimate: S
> upstream-pieces: [] · downstream-pieces: [A23, A18, A7, B4, A4]

## 1. Implementation Strategy
Enforcement/verification map. ADR 0026 is a topology-bounding decision enforced mostly by *absence*: no shared identity root, no cross-cluster mesh, no shared registry, no cross-cluster A2A. The positive obligations are per-cluster singletons (one Keycloak realm, one OPA bundle, one audit pipeline, own eventing/secrets/cloud bindings) and that multi-environment installs are N independent ArgoCD-reconciled instances. **A23** (Kargo) provides the only inter-cluster contract — promotion, explicitly not federation. Conformance is tested by auditing for the absence of federation artifacts, verifying per-cluster singletons, and confirming Kargo promotion leaves each cluster independent.

## 2. Ordered Task List
- TASK-01: Map each REQ to its enforcing surface (federation-absence audit, per-cluster singletons, A23 promotion) — produces: enforcement matrix — depends-on: []
- TASK-02: Audit for absence of shared identity root / cross-cluster mesh / shared registry / cross-cluster A2A — produces: federation-absence audit — depends-on: [TASK-01]
- TASK-03: Verify each cluster owns one Keycloak realm, one OPA bundle, one audit pipeline, own eventing/secrets/cloud bindings — produces: singleton audit — depends-on: [TASK-01]
- TASK-04: Verify workload identity = Kubernetes ServiceAccounts; no SPIFFE/SPIRE present — produces: identity-primitive check — depends-on: [TASK-02]
- TASK-05: Verify dev/staging/prod realized as N independent ArgoCD-reconciled installs — produces: multi-env layout check — depends-on: [TASK-03]
- TASK-06: Verify Kargo promotion runs cross-cluster with each cluster still independent — produces: A23 promotion conformance — depends-on: [TASK-05]
- TASK-07: Verify HA/cross-region absent from v1.0 scope; federation gated behind backlog 3.13 trigger — produces: scope audit — depends-on: [TASK-01]

## 3. Dependency Map
### 3.1 Upstream pieces that must ship first (HARD)
- None (topology-bounding decision; constrains rather than consumes).
### 3.2 Downstream pieces blocked on this
- A23 (Kargo promotion-not-federation), A18/A7/B4/A4 (per-cluster singletons).
### 3.3 Continuous (non-blocking) inputs
- ADR 0028 (single-cluster identity), ADR 0023 (per-cluster sources), ADR 0034 (per-cluster audit); B22 threat model.

## 4. Parallelizable Subtasks
TASK-02, TASK-03, TASK-07 fan out independently once TASK-01 lands. TASK-04 follows TASK-02; TASK-05→TASK-06 are serial.

## 5. Test Strategy
- AC-01 → PyTest/manifest-lint: no shared identity root, mesh, registry, or cross-cluster A2A artifact.
- AC-02 → Chainsaw: realize dev/staging/prod as three independent ArgoCD installs.
- AC-03 → PyTest: workload identity resolves to SAs; no SPIFFE/SPIRE component.
- AC-04 → PyTest: each cluster has exactly one realm/OPA-bundle/audit-pipeline + own eventing/secrets/cloud bindings.
- AC-05 → Chainsaw: Kargo promotion across the independent installs leaves each independent (no shared runtime state introduced).
- AC-06 → scope-lint: HA/cross-region absent; federation gated behind backlog 3.13.
- Fixtures/fakes: multi-cluster harness (kind clusters) + ArgoCD + Kargo stubs for not-yet-landed A23.

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/authoring`
### 6.2 PR — `piece/ADR-0026-independent-cluster-no-federation` → base `wave/authoring`; carries spec + plan.
### 6.3 Merge order — independent of sibling ADR specs; rolls up to main.

## 7. Effort Estimate
S overall. Per-task: TASK-01 S, 02 S, 03 S, 04 S, 05 M, 06 M, 07 S. Critical path: TASK-01 → 03 → 05 → 06.

## 8. Rollback / Reversibility
Decision record; back out by reverting spec+plan. Reverting reopens the federation design space (per backlog 3.13 trigger); A23 loses its promotion-not-federation framing and per-cluster singleton guarantees lose their conformance contract. No runtime artifact is deleted; the topology is inherently reversible by adding federation later.
