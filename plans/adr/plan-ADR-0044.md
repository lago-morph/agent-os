> spec: SPEC-ADR-0044 · kind: ADR · tier: T0
> wave: authoring-parallel (auth) · estimate: S
> upstream-pieces: [A7, A11] · downstream-pieces: [B4, A18, A21, A23, B19, B11]

# Plan: ADR-0044 — Uniform Substrate XRD Contract (Crossplane v2)

## 1. Strategy

ADR 0044 supersedes ADR 0041 and is an authoring-only ADR: it fixes a contract, it does not ship
code. The work is to write the ADR text in the Crossplane v2 model, thread it into the glossary
and interface contract, and ensure the downstream substrate specs (A18, A21, A23, B19, B11)
inherit the contract by reference. No runtime artifacts are produced by this plan; enforcement
lives in the downstream pieces and in the OPA Gatekeeper admission policy (B4).

The contract is deliberately small: one XRD per substrate exposing a namespace-scoped XR (no
claim layer, no `X` prefix), one Composition per backing implementation, a uniform
connection-secret shape, a substrate-agnostic status surface, and an admission guardrail.
Everything else is delegated to the per-substrate specs.

## 2. Task List

- **T1** — Author the ADR text (purpose, scope, contract, REQs, ACs) in the v2 model; state that
  it supersedes ADR 0041.
- **T2** — Define the XRD contract: `spec.names` for a namespace-scoped XR; assert
  `spec.claimNames` does not exist in v2.
- **T3** — Specify the one-Composition-per-implementation rule (kind + AWS).
- **T4** — Specify the uniform connection-secret shape + key set, written to the XR's namespace.
- **T5** — Specify the substrate-agnostic status surface (`Ready`/`Synced` on the XR).
- **T6** — Specify the OPA Gatekeeper admission guardrail (reject XRs with no Composition).
- **T7** — Thread XRD versioning per ADR 0030.
- **T8** — Update glossary + interface contract to the v2 contract (drop `X` prefix / claim
  language; thread D-07: add `SearchIndex`, `MongoDocStore`, `ObjectStore` as v2 composite types).

## 3. Dependency Map

- T1 precedes all other tasks.
- T2–T7 are independent once T1 lands.
- T8 depends on T2–T7 (it documents the settled contract).

## 4. Parallelizable Subtasks

- T2, T3, T4, T5, T6, T7 can be authored in parallel.
- T8 is the join point.

## 5. Test Strategy

- **AC-01 test** — Provision each substrate with no backing-implementation reference.
- **AC-02 test** — Swap a Composition; assert no consumer manifest changes.
- **AC-03 test** — Assert identical connection-secret key set across Compositions.
- **AC-04 test** — Assert `Ready`/`Synced` on every XR.
- **AC-05 test** — Assert an XR with no matching Composition is rejected at admission.
- **AC-06 test** — Honored when `Postgres` exists as a namespaced XR users create directly.
- **AC-07 test** — Assert XRD version changes follow ADR 0030.

## 6. PR / Branch Mapping

- Single PR: the ADR text + glossary/interface-contract updates.

## 7. Effort Estimate

- **S** — authoring-only; no runtime code.

## 8. Rollback

- Revert the ADR text + glossary/interface-contract edits. No runtime impact. ADR 0041 remains
  available (superseded) for reference.
