# PLAN B17 — Agent profile library

> spec: SPEC-B17 · kind: COMPONENT · tier: T1
> wave: W2 · estimate: M
> upstream-pieces: [B13, A5, A1] · downstream-pieces: [B18]

## 1. Implementation Strategy
B17 is curated content, not code: a small set of GitOps-packaged `CapabilitySet` YAML
artifacts plus a derivation guide. The approach is to (a) fix the four required profile
categories (knowledge-base, code-gen, customer-support, minimal RAG) against the Canon
`CapabilitySet` field set (interface-contract §1.4), (b) author each profile as full-list
overlays so the §6.8 replace-if-there semantics resolve deterministically, (c) wire them to
reconcile through ArgoCD → the B13 kopf operator exactly like any other capability CRD, and
(d) ship a "derive a new profile" guide and overlay-resolution tests. No reconciler logic is
written here — B17 leans entirely on B13 and ADR 0032 mechanics.

## 2. Ordered Task List
- TASK-01: Fix profile names + categories and reconcile them with B18 — produces: name registry
  doc — depends-on: [].
- TASK-02: Author the `knowledge-base-agent` profile referencing `platform-knowledge-base`
  `RAGStore` + read-oriented `opaPolicyRefs` — produces: CapabilitySet YAML — depends-on: [TASK-01].
- TASK-03: Author `code-gen-agent`, `customer-support-agent`, `minimal-rag-agent` profiles as
  full-list overlays — produces: CapabilitySet YAMLs — depends-on: [TASK-01].
- TASK-04: Add API-version pinning + GitOps packaging (ArgoCD app / kustomize) so profiles
  reconcile via B13 — produces: packaging manifests — depends-on: [TASK-02, TASK-03].
- TASK-05: Write the derivation guide (layering order, add/replace resolution, `overrides`
  behavior, SDK-default note) — produces: docs — depends-on: [TASK-02, TASK-03].
- TASK-06: Author Chainsaw apply/resolve tests + PyTest overlay-resolution assertions — produces:
  test suites — depends-on: [TASK-04].

## 3. Dependency Map
### 3.1 Upstream pieces that must ship first (HARD)
- B13 — `CapabilitySet` CRD schema + reconcile-to-LiteLLM behavior; B17 profiles are instances it reconciles.
- A5 — `Agent` CRD `capabilitySetRefs[]` target field the profiles slot into.
- A1 — LiteLLM gateway the capabilities resolve into.

### 3.2 Downstream pieces blocked on this
- B18 — recommended `Agent` compositions consume B17 profiles verbatim.

### 3.3 Continuous (non-blocking) inputs
- B14 (test framework), B22 (threat model / security standards), B16/B3 (OPA policy content
  referenced by `opaPolicyRefs`), A17 (registered `MCPServer`/`EgressTarget` instance names).

## 4. Parallelizable Subtasks
- TASK-02 and TASK-03 (profile authoring) run concurrently once TASK-01 fixes names.
- TASK-05 (docs) runs in parallel with TASK-04/TASK-06 once profiles exist.

## 5. Test Strategy
- AC-B17-01/03/06/07 → Chainsaw: apply the library, assert ≥4 `CapabilitySet`s, assert KB profile
  `ragStores[]` contains `platform-knowledge-base`, assert reconcile via B13 with zero manual
  LiteLLM admin calls, assert apiVersion present. Needs a B13 reconciler fake/stub if B13 not landed.
- AC-B17-02/04 → PyTest: schema/lint check (no out-of-schema field; names resolve to Canon kinds)
  and a two-layer overlay-resolution assertion matching §6.8 replace semantics.
- AC-B17-05 → manual/review gate: a reviewer follows the guide to produce a passing fifth profile.
- AC-B17-08 → joint with B18 test suite (Agent referencing profile resolves as expected).
- AC-B17-09 → lint: doc states `deep-agents` default; CRD has no `sdk` field. Playwright N/A (no UI).

## 6. PR / Branch Mapping
### 6.1 Stack position — base branch = `wave/2` (contains upstream B13/A5/A1 specs).
### 6.2 PR — `piece/B17-agent-profile-library` → base `wave/2`; carries spec-B17 + plan-B17.
### 6.3 Merge order — independent of W2 siblings; rolls up to main with the wave. B18 (W4) rebases on it.

## 7. Effort Estimate
- TASK-01 S · TASK-02 S · TASK-03 M · TASK-04 S · TASK-05 M · TASK-06 M. Rollup: **M**.
- Critical path: TASK-01 → TASK-03 → TASK-04 → TASK-06.

## 8. Rollback / Reversibility
Fully reversible: removing the profile YAMLs from Git causes ArgoCD/B13 to delete the
`CapabilitySet`s. Downstream impact: any B18 `Agent` referencing a deleted profile fails to
resolve its `capabilitySetRefs[]`; mitigate by reverting B18 references first or keeping profiles
until B18 is repointed. No data migration, no schema change — content-only revert.
