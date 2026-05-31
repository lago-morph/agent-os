# PLAN C4 — Diataxis reference

> spec: SPEC-C4 · kind: COMPONENT · tier: T1
> wave: consumer (authoring-parallel; finalized flow-last, partials encouraged earlier) · estimate: L
> upstream-pieces: [C1] · downstream-pieces: [C8]

## 1. Implementation Strategy

Author the §10.3 reference set as docs-as-code into C1's reference navigation slot, one reference
document per §10.3 surface, covering both LangGraph and Deep Agents (ADR 0019). Drive accuracy from
`_meta/interface-contract.md`: reproduce source-stated fields verbatim and record everything not
source-stated as deferred to its owning component (never invent). Build an automated interface-
contract conformance check that fails on any divergent or silently-invented CRD/XRD field, event
namespace, connection-secret field, or CLI subcommand. Contribute partial reference from earlier
components, then finalize each multi-component doc with the last component in its flow (§10).

## 2. Ordered Task List

- TASK-01: Adopt C1 conventions + reference section scaffold + shared reference template + interface-contract conformance harness — produces: reference template + conformance check — depends-on: [C1]
- TASK-02: ARK CRD reference (`Agent`/`AgentRun`/`Team`/`Tool`/`Memory`/`Evaluation`/`Query`) — produces: reference doc — depends-on: [TASK-01]
- TASK-03: Sandbox CRD reference (`Sandbox`/`SandboxTemplate`) — produces: reference doc — depends-on: [TASK-01]
- TASK-04: Capability CRD reference (`MCPServer`/`A2APeer`/`RAGStore`/`EgressTarget`/`Skill`/`CapabilitySet`/`VirtualKey`/`BudgetPolicy`) — produces: reference doc — depends-on: [TASK-01]
- TASK-05: Crossplane XR catalog (MemoryStore/AgentEnvironment/SyntheticMCPServer/GrafanaDashboard/AuditLog/TenantOnboarding/AgentDatabase/Postgres/SearchIndex/ObjectStore/MongoDocStore) — produces: reference doc — depends-on: [TASK-01]
- TASK-06: LiteLLM gateway + admin API reference — produces: reference doc — depends-on: [TASK-01]
- TASK-07: Platform SDK reference (Python/TypeScript; `memory.*`/`rag.*`/OTel/A2A helpers) — produces: reference doc — depends-on: [TASK-01]
- TASK-08: SDK surface reference — LangGraph + Deep Agents (ADR 0019) — produces: reference doc — depends-on: [TASK-01]
- TASK-09: OPA policy library reference — produces: reference doc — depends-on: [TASK-01]
- TASK-10: CloudEvent schema registry reference (ten `platform.*` namespaces + versioning rule) — produces: reference doc — depends-on: [TASK-01]
- TASK-11: Audit event schema reference (ADR 0034) — produces: reference doc — depends-on: [TASK-01]
- TASK-12: Metrics catalog + trace span attribute reference — produces: reference docs — depends-on: [TASK-01]
- TASK-13: Headlamp extension points + CI CLI command reference (`agent-platform`) — produces: reference docs — depends-on: [TASK-01]
- TASK-14: LibreChat config reference (locked-down profile, ADR 0007) + Langfuse dataset/evaluator schemas — produces: reference docs — depends-on: [TASK-01]
- TASK-15: Run conformance check + cross-doc consistency + C8 corpus-shape conformance — produces: clean, contract-conformant re-indexable corpus — depends-on: [TASK-02..14]

## 3. Dependency Map

### 3.1 Upstream pieces that must ship first (HARD)
- C1 — portal infrastructure, reference nav slot, authoring conventions, contribution-workflow check.

### 3.2 Downstream pieces blocked on this
- C8 — indexes the reference corpus into `platform-knowledge-base`.

### 3.3 Continuous (non-blocking) inputs
- A5, A6, B13, B4, A1, B6, B7, B16/B3, B12, A18, A13, A9, B9, A8, A2 — the surface-owning
  components whose interfaces C4 documents; partial reference contributed early, finalized flow-last.
- `_meta/interface-contract.md` — the conformance source of truth; the check tracks it continuously.
- B14 test framework — runs the conformance/lint checks; fakes until landed.

## 4. Parallelizable Subtasks

- After TASK-01: TASK-02 through TASK-14 (the reference documents) are independent and fan out concurrently.
- TASK-15 converges the fan-out (conformance + consistency + C8 conformance).

## 5. Test Strategy

- Chainsaw: N/A — C4 defines no CRD; CRD shape conformance is owned by the reconciler components.
- Playwright: AC-C4-01 (each reference doc renders/navigable), AC-C4-07 (two version builds via
  selector), AC-C4-08 (docs-check pass on a reference PR).
- PyTest: AC-C4-02 (both SDK surfaces present), AC-C4-03/04 (interface-contract conformance: zero
  divergent or silently-invented names; non-Canon elements tagged/deferred), AC-C4-05 (ten
  namespaces + versioning rule), AC-C4-06 (canonical connection-secret + status fields),
  AC-C4-09 (flow-last finalization present). C8 corpus-shape conformance via fake indexer until C8.

## 6. PR / Branch Mapping

### 6.1 Stack position — base branch = `wave/consumer` (contains spec-C1 skeleton + conventions)
### 6.2 PR — `piece/C4-diataxis-reference` → base `wave/consumer`; carries spec-C4 + plan-C4
### 6.3 Merge order — after C1; independent of C2/C3/C5 siblings; reference docs finalize flow-last; rolls up to main

## 7. Effort Estimate

- TASK-01 M (incl. conformance harness) · TASK-02..14 each S/M · TASK-15 M
- Rollup: L (broadest surface area of the four quadrants; high drift-management cost).
- Critical path: TASK-01 (conformance harness) → (any reference doc) → TASK-15.

## 8. Rollback / Reversibility

Revert the reference Markdown via GitOps; fully reversible — authored content, no runtime state, no
CRD. If reverted, developers and Platform Agents lose the lookup surface and C8 loses reference
content from `platform-knowledge-base`; no reconciler behavior changes (C4 documents, never defines).
