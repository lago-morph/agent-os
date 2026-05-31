# ADR 0042: Spec/plan corpus structure and the frozen-Canon convention

- **Status**: Accepted
- **Date**: 2026-05-30

## Context

The architecture in this repository decomposes the platform into ~119 discrete
"pieces" — 65 components (Workstreams A–F), 13 architectural views (§ 6.1–6.13),
and 41 ADRs. Producing a detailed implementation spec and plan for every piece is
inherently a large parallel-authoring effort: dozens of agents writing dozens of
files that all reference the same shared vocabulary — CRD/XRD names (`MCPServer`,
`A2APeer`, `CapabilitySet`, `XPostgres`, …), the CloudEvent taxonomy, SDK surfaces,
and glossary terms drawn from the overview (§ 6.12, § 15) and the ADRs.

Two problems must be solved structurally, not by hope:

1. **Where does the spec/plan corpus live**, so it is durable, navigable, and
   extensible by future work (including the deferred Operational/NFR layer and any
   re-runs)?
2. **How do independent parallel authors avoid drifting** on shared names —
   spelling the same CRD three ways, inventing fields, misciting ADR numbers — when
   no single agent sees all the others' output?

The first parallel spec/plan run (PR #8) established a working answer to both. This
ADR records it as the binding convention so subsequent spec work extends the same
structure rather than reinventing it.

## Decision

**Adopt a fixed corpus layout and a mandatory frozen-Canon step for parallel
authoring.**

Corpus layout (additive; source architecture docs are never modified by spec work):

```
specs/   components/{A..F}/spec-<ID>.md   views/spec-V6-NN.md   adr/spec-ADR-NNNN.md
plans/   components/{A..F}/plan-<ID>.md   views/plan-V6-NN.md   adr/plan-ADR-NNNN.md
_meta/   glossary.md  interface-contract.md  piece-index.csv  waves.md
         templates/{spec,plan}.md
```

- Every piece produces exactly two files — one `spec-<ID>.md` and one
  `plan-<ID>.md` — that travel together. File names are keyed to stable piece IDs
  drawn from `_meta/piece-index.csv`.
- Specs and plans follow the fixed-heading templates in `_meta/templates/`, so the
  corpus is uniform and machine-checkable (every spec has numbered `REQ-*`/`AC-*`
  with each AC mapping a REQ; every plan has a dependency map and PR/branch mapping).

The **frozen Canon** under `_meta/` is the anti-drift contract:

- `glossary.md` — canonical casing + meaning for every proper noun an author could
  misspell. FROZEN for the run.
- `interface-contract.md` — the cross-piece registry: every CRD/XRD → owning
  component, the CloudEvent taxonomy, SDK surfaces, connection-secret contract,
  audit-adapter interface. Fields are recorded only as the source states them; gaps
  are marked "not specified in source", never invented.
- `piece-index.csv` — one row per piece (id, kind, tier, wave, upstream, downstream,
  estimate).
- `waves.md` — the dependency-derived build/authoring layering.

The Canon is extracted **once, serially, before any authoring fan-out**, spot-checked
by a human or orchestrator, and then treated as immutable for the run. Every authoring
agent's brief begins with "read the Canon; use ONLY its names." Changing the Canon
mid-run is a deliberate revision, not a silent edit, because downstream files are built
on it.

## Alternatives considered

- **Let each author read the raw architecture docs and derive vocabulary itself.**
  Rejected: every author re-reading a 150 KB overview both wastes budget and produces
  divergent readings — the exact drift the Canon prevents.
- **One large file per workstream instead of per-piece files.** Rejected: per-piece
  files maximize the number of independently breakable-out units (a stated goal) and
  keep merge conflicts near zero since no two pieces share a file.
- **No fixed templates.** Rejected: uniform headings are what make the corpus
  machine-checkable for completeness and consistency.

## Consequences

- Future spec work — including the deferred Operational/NFR layer (`_meta/
  pending-operational-nfr-layer.md`) and any re-runs or back-fills — extends this
  same structure rather than inventing a new one.
- The Canon is the single load-bearing artifact; an error in it propagates to every
  downstream file, so the spot-check before freezing is mandatory.
- The corpus is greppable for completeness (`piece-index.csv` as manifest) and for
  gaps (the `[PROPOSED — not in source]` marker; see ADR 0043).
- The `canon-freeze-before-fanout` and `coverage-manifest-gapfill` skills operationalize
  this ADR for agents.

## References

- [architecture-overview.md](../architecture-overview.md) [§ 6.12](../architecture-overview.md#612-crd-inventory), [§ 14](../architecture-overview.md#14-components-and-dependencies), [§ 15](../architecture-overview.md#15-glossary)
- [ADR 0030](./0030-crd-and-api-versioning-policy.md) (CRD / API versioning policy — the contract the interface-contract records)
- [ADR 0031](./0031-cloudevent-top-level-taxonomy.md) (CloudEvent taxonomy — captured in the Canon)
- [ADR 0043](./0043-proposed-not-in-source-marker.md) (the not-invented marker that pairs with the Canon)
- `retrospective/2026-05-30-8.md` (the session that established this structure)
