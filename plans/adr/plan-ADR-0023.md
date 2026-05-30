# PLAN ADR-0023 — Knative broker is in the architecture; sources are environment-specific by design [PROPOSED]

> spec: SPEC-ADR-0023 · kind: ADR · tier: T1
> wave: authoring-parallel · estimate: S
> upstream-pieces: [A4] · downstream-pieces: [B8, B12, A19, A14]

## 1. Implementation Strategy
Enforcement/verification map. ADR 0023 is enforced primarily by **A4** (broker/Triggers/channels ship as one portable shape in the platform manifests) and the install/overlay layout (sources live only in environment-specific overlays). **B8** (adapters) and **B12** (schema registry) prove portability: the same test suite runs unchanged across kind/EKS/AKS. The "cloud-source consumers document a kind-equivalent or are cloud-only" obligation is enforced in each such component's deliverables. Conformance is tested by asserting no source manifests in platform manifests, sources only in overlays, identical adapter/registry behavior across environments, and the AlertManager → HolmesGPT flow succeeding on kind with no cloud source.

## 2. Ordered Task List
- TASK-01: Map each REQ to its enforcing piece (A4 manifests, overlay layout, B8/B12, cloud-source components) — produces: enforcement matrix — depends-on: []
- TASK-02: Verify A4 ships broker/Triggers/channels/taxonomy as one shape with zero per-env forks — produces: A4 conformance check — depends-on: [TASK-01]
- TASK-03: Verify source resources exist only in environment overlays (kind/EKS/AKS) — produces: overlay-layout audit — depends-on: [TASK-01]
- TASK-04: Define portability conformance test (same B8/B12 suite across environments) — produces: Chainsaw+PyTest set — depends-on: [TASK-02]
- TASK-05: Audit cloud-source-dependent components for kind-equivalent path or cloud-only marker — produces: cloud-source coverage audit — depends-on: [TASK-01]
- TASK-06: Verify AlertManager → HolmesGPT flow on kind with no cloud source — produces: Chainsaw flow test — depends-on: [TASK-02]
- TASK-07: Verify install docs enumerate per-environment source choices — produces: doc-lint — depends-on: [TASK-03]

## 3. Dependency Map
### 3.1 Upstream pieces that must ship first (HARD)
- A4 — broker/Triggers/channels/taxonomy as the portable contract.
### 3.2 Downstream pieces blocked on this
- B8 (adapters), B12 (schema registry) inherit the portability contract.
### 3.3 Continuous (non-blocking) inputs
- A19 (Mattermost adapter), A14 (HolmesGPT); B22 threat model for source-side trust boundaries.

## 4. Parallelizable Subtasks
TASK-04, TASK-05, TASK-06 fan out once TASK-02 lands. TASK-03 and TASK-07 run parallel to TASK-02.

## 5. Test Strategy
- AC-01/AC-02 → PyTest manifest-lint (no sources in platform manifests; sources in overlays) + Chainsaw across environments.
- AC-03 → Chainsaw: emit to broker, assert Trigger fires regardless of source.
- AC-04 → Chainsaw+PyTest: same B8/B12 suite on kind/EKS/AKS broker contracts.
- AC-05 → PyTest doc-lint over cloud-source-dependent component deliverables.
- AC-06 → PyTest doc-lint over install docs.
- AC-07 → Chainsaw: AlertManager → HolmesGPT on kind, no cloud source.
- Fixtures/fakes: stub broker + AlertManager emitter for not-yet-landed A4/A14; webhook/synthetic source fake for kind.

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/authoring`
### 6.2 PR — `piece/ADR-0023-env-specific-knative-sources` → base `wave/authoring`; carries spec + plan.
### 6.3 Merge order — independent of sibling ADR specs; rolls up to main.

## 7. Effort Estimate
S overall. Per-task: TASK-01 S, 02 S, 03 S, 04 M, 05 S, 06 M, 07 S. Critical path: TASK-01 → 02 → 04.

## 8. Rollback / Reversibility
Decision record; back out by reverting spec+plan. If reverted, A4/B8/B12 lose the portability conformance contract and cloud-source components lose the kind-equivalent obligation; no runtime artifact is deleted.
