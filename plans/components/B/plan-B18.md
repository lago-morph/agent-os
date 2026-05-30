# PLAN B18 — Recommended agent compositions

> spec: SPEC-B18 · kind: COMPONENT · tier: T2
> wave: W4 · estimate: M
> upstream-pieces: [B17, A5, A1, B7] · downstream-pieces: []

## 1. Implementation Strategy
B18 is reference content: complete `Agent` CRDs assembled from B17 profiles. The approach is to
import the final B17 profile names, author one `Agent` CRD per common shape using only Canon §1.2
fields, document the resolved capability view for each, and validate that each reconciles through
ARK and resolves deterministically under the §6.8 overlay model. No new CRDs, primitives, or
reconciler logic — B18 sits entirely on top of B17 (profiles), A5 (ARK `Agent` CRD), and B7
(SDK values). Because it is W4, it lands after its upstreams and can pin to real shipped names.

## 2. Ordered Task List
- TASK-01: Import final B17 profile names + the shipped `sandboxTemplateRef`/`modelRef`/`image`
  names — produces: reference table — depends-on: [].
- TASK-02: Author the four reference `Agent` CRDs (kb/chat, code-gen, customer-support, minimal
  RAG) with `capabilitySetRefs[]`, `overrides`, `sdk` — produces: Agent YAMLs — depends-on: [TASK-01].
- TASK-03: Compute + document the resolved capability view per composition — produces: docs —
  depends-on: [TASK-02].
- TASK-04: GitOps-package the compositions for ArgoCD → ARK reconcile + pin Agent apiVersion —
  produces: packaging manifests — depends-on: [TASK-02].
- TASK-05: Author Chainsaw apply/resolve tests + PyTest resolved-capability assertions — produces:
  test suites — depends-on: [TASK-04].
- TASK-06: Write the "assemble an agent from profiles" how-to + one A2A-exposing variant — produces:
  docs + example — depends-on: [TASK-02, TASK-03].

## 3. Dependency Map
### 3.1 Upstream pieces that must ship first (HARD)
- B17 — `CapabilitySet` profile names referenced verbatim in `capabilitySetRefs[]`.
- A5 — `Agent` CRD definition + ARK reconcile.
- A1 — LiteLLM gateway capabilities resolve into.
- B7 — supported `sdk` values (`langgraph`, `deep-agents`).

### 3.2 Downstream pieces blocked on this
- None (B18 is a leaf; consumed by docs/training C2/C3/E2 as content, non-blocking).

### 3.3 Continuous (non-blocking) inputs
- B14 (test framework), B22 (security standards), B13 + ADR 0032 (overlay resolver for resolved-view
  validation), A17 (registered capability instance names), A6 (`SandboxTemplate` names).

## 4. Parallelizable Subtasks
- The four `Agent` CRDs in TASK-02 are independent and authored concurrently.
- TASK-03 (resolved-view docs) and TASK-06 (how-to) run in parallel once CRDs exist.

## 5. Test Strategy
- AC-B18-01/05/06 → Chainsaw: apply compositions, assert four schema-valid `Agent`s, reconcile via
  ARK with zero manual config, apiVersion present, `exposes` version string present on the A2A variant.
- AC-B18-02/03/04/07 → PyTest: lint for profile-bypassing inline lists, `sdk` enum + §1.2-only field
  check, resolved-capability computation vs documented view, and KB composition resolving to
  `platform-knowledge-base`. Needs a fake overlay resolver if B13/ADR 0032 not yet landed.
- Playwright N/A (no UI). Joint check with B17 AC-B17-08 (profile resolves inside an Agent).

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/4` (contains B17, A5, A1, B7).
### 6.2 PR — `piece/B18-recommended-agent-compositions` → base `wave/4`; carries spec-B18 + plan-B18.
### 6.3 Merge order — independent of W4 siblings; must rebase on the merged B17 profile names before merge.

## 7. Effort Estimate
- TASK-01 S · TASK-02 M · TASK-03 S · TASK-04 S · TASK-05 M · TASK-06 S. Rollup: **M**.
- Critical path: TASK-01 → TASK-02 → TASK-04 → TASK-05.

## 8. Rollback / Reversibility
Fully reversible: removing the `Agent` CRD YAMLs from Git causes ArgoCD/ARK to delete the example
agents. No downstream piece hard-depends on B18, so revert is low blast-radius (loss of reference
examples only). No schema or data migration involved — content-only revert.
