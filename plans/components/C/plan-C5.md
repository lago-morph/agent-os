# PLAN C5 — Diataxis explanation

> spec: SPEC-C5 · kind: COMPONENT · tier: T1
> wave: consumer (authoring-parallel; anchored to views + ADRs, re-aligned on their revision) · estimate: M
> upstream-pieces: [C1] · downstream-pieces: [C8]

## 1. Implementation Strategy

Author the §10.4 explanation set as docs-as-code into C1's explanation navigation slot, one article
per §10.4 item, each anchored to its architecture view (§6.x) and ADR(s). Drive each article from
the cited source so it restates Canon rationale faithfully rather than re-deriving it; mark anything
beyond source `[PROPOSED — not in source]`. Use a uniform structure (tension → decision → trade-offs
→ consequences) and cross-link to the corresponding C4 reference and C3 how-to for the "why → what/
how" path. Articles fan out independently once C1's skeleton and conventions exist.

## 2. Ordered Task List

- TASK-01: Adopt C1 conventions + explanation section scaffold + shared article template (tension/decision/trade-offs/consequences) + view/ADR-citation lint — produces: article template + citation lint — depends-on: [C1]
- TASK-02: Why-this-architecture — design principles (§6.1) — produces: article — depends-on: [TASK-01]
- TASK-03: How-sandboxing-works — defense in depth (ADR 0003; §6.2/§6.6) — produces: article — depends-on: [TASK-01]
- TASK-04: How-the-gateway-controls-cost (LiteLLM + `BudgetPolicy`) — produces: article — depends-on: [TASK-01]
- TASK-05: How-memory-works-across-agents — namespaces/sharing/lifecycle (ADR 0025; §6.3) — produces: article — depends-on: [TASK-01]
- TASK-06: Trade-offs-between-memory-backends (Letta, ADR 0005) — produces: article — depends-on: [TASK-01]
- TASK-07: Why-ARK-as-the-agent-operator (ADR 0001) — produces: article — depends-on: [TASK-01]
- TASK-08: Why-NATS-JetStream-as-the-broker (ADR 0004; §6.7) — produces: article — depends-on: [TASK-01]
- TASK-09: Why-Envoy-egress-rather-than-CNI-L7-policy (ADR 0003; §6.6) — produces: article — depends-on: [TASK-01]
- TASK-10: Why-a-Python-kopf-operator-rather-than-a-Crossplane-provider-for-LiteLLM (ADR 0006) — produces: article — depends-on: [TASK-01]
- TASK-11: Multi-tenancy-story (namespaces + RBAC-floor/OPA-restrictor, ADR 0016/0018; §6.9) — produces: article — depends-on: [TASK-01]
- TASK-12: CapabilitySet-layering-semantics (ADR 0013/0032; §6.8) — produces: article — depends-on: [TASK-01]
- TASK-13: Platform-self-management-with-HolmesGPT (ADR 0012; §6.10) — produces: article — depends-on: [TASK-01]
- TASK-14: Canon-name lint + citation-presence check + cross-link wiring to C4/C3 + C8 corpus-shape conformance — produces: clean, anchored, cross-linked re-indexable corpus — depends-on: [TASK-02..13]

## 3. Dependency Map

### 3.1 Upstream pieces that must ship first (HARD)
- C1 — portal infrastructure, explanation nav slot, authoring conventions, contribution-workflow check.

### 3.2 Downstream pieces blocked on this
- C8 — indexes the explanation corpus into `platform-knowledge-base`.

### 3.3 Continuous (non-blocking) inputs
- Architecture views §6.1–§6.10 and the ADR set — the source material; articles re-aligned when
  views/ADRs are revised.
- C3 (how-to) + C4 (reference) — cross-link targets (AC-C5-07); resolved as they land in-wave.
- The components whose design each article explains (A1/A5/A6/A4/A10/A14/B13/OPA) — conceptual
  subjects, not code dependencies.

## 4. Parallelizable Subtasks

- After TASK-01: TASK-02 through TASK-13 (the twelve articles) are independent and fan out concurrently.
- TASK-14 converges the fan-out (lint + citation check + cross-links + C8 conformance).

## 5. Test Strategy

- Chainsaw: N/A — C5 defines no CRD.
- Playwright: AC-C5-01 (each article renders/navigable), AC-C5-07 (cross-links to C4/C3 resolve),
  AC-C5-06 (docs-check pass on an explanation PR).
- PyTest: AC-C5-02 (every article cites its §6 view + ADR(s); no rationale absent from source),
  AC-C5-03 (structure present), AC-C5-04 (Canon-name lint), AC-C5-05 (multi-tenancy + layering
  articles match ADR 0016/0018/0013/0032). C8 corpus-shape conformance via fake indexer until C8.

## 6. PR / Branch Mapping

### 6.1 Stack position — base branch = `wave/consumer` (contains spec-C1 skeleton + conventions)
### 6.2 PR — `piece/C5-diataxis-explanation` → base `wave/consumer`; carries spec-C5 + plan-C5
### 6.3 Merge order — after C1; independent of C2/C3/C4 siblings (cross-links resolve in-wave); rolls up to main

## 7. Effort Estimate

- TASK-01 S · TASK-02..13 each S/M (twelve articles) · TASK-14 M
- Rollup: M (narrative content anchored to existing views/ADRs; low integration complexity).
- Critical path: TASK-01 → (any article) → TASK-14.

## 8. Rollback / Reversibility

Revert the explanation Markdown via GitOps; fully reversible — authored content, no runtime state,
no CRD. If reverted, the platform loses its authored conceptual layer and C8 loses explanation
content from `platform-knowledge-base`; the underlying views/ADRs are unaffected (C5 restates, never
amends).
