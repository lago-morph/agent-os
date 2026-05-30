# Build Waves (authoritative layering)

Derived from `architecture-overview.md` §14.7 (dependency DAG) and the §14.1–14.6 component
tables. This file fixes the **wave layering** that `piece-index.csv` encodes, so the ~30
parallel spec authors share one mental model of build order. Authoring of specs is parallel;
the waves below describe **real-build** order, not authoring order.

## Wave table

| Wave | Band | Components |
|---|---|---|
| **W0** | Foundation (no internal deps; parallelizable from day one) | A1, A2, A3, A4, A6, A7, A8, A9, A11, A13, A14, A15, B22 |
| **W1** | First dependents | A5, A10, A18, A19, A20, B1, B2, B3, B4*, B8, B11, B13 |
| **W2** | Second layer | A16, A17, A22, B6, B9, B12, B16, B17, B20 |
| **W3** | Third layer | A21, B7, B10, B14, B19-core |
| **W4** | Fourth layer | A23, B5, B15, B18, B21, B19-ui |
| **consumer** | Author-parallel; real-build after A/B | C1–C9, D1–D3, E1–E2, F1–F6 |
| **auth** | Authoring-parallel context (no build wave) | Views V6-01..V6-13, ADRs ADR-0001..ADR-0041 |

\* **B4 (Crossplane v2 Compositions) is not listed in any wave set in the source.** It owns the
XRDs that A18, A21, A23, and B19 consume, so it is placed in **W1** (foundation band) here as a
best-read deviation. This is the single wave-assignment gap in §14 — see "Contradictions" below.

## Foundation set

The DAG marks as **foundation** (no internal dependency within their workstream, parallelizable
from day one): the W0 set above. Specifically called out in §14.7:

- **B22 (security threat model design)** — a *design specification, not code*; ships before the
  first wave of A/B implementation and feeds standards forward into every component.
- **A14 (HolmesGPT)** — lands very early per ADR 0012's phased trajectory: phase 1 on the raw
  cluster with read-only ServiceAccount + read-only AWS IAM, then incrementally rewired through
  the platform layers (LiteLLM, audit adapter, OPA decision points) as they land.
- **A11 (OpenSearch)** and **A13 (Tempo + Mimir)** — in early so other components can contribute
  knowledge and emit traces from the start.

## The B19 → A23 → B5 cycle resolution

The raw hard edges form an apparent loop across these three pieces:

- `B19 --> A23` (Kargo composes with the `Approval` CRD for human gates)
- `A23 --> B5` (Kargo's Headlamp plugin is a cross-cutting plugin)
- `B5 --> B19` (B5 ships the approval-queue UI that the approval system needs)

This is resolved by **splitting B19**:

- **B19-core (W3)** — the `Approval` CRD, OPA elevation logic, Argo Workflow integration,
  CloudEvent emission of decisions. This is what A23 (W4) depends on; it lands first.
- **B19-ui (W4)** — the Headlamp approval-queue plugin, delivered alongside **B5 (W4)** (B5 owns
  the cross-cutting Headlamp plugins, including the Kargo plugin and the approval-queue UI).

With the UI portion deferred to W4, the dependency order is acyclic:
`B19-core (W3) → A23 (W4)` and `A23 (W4) → B5/B19-ui (W4)` co-land. In `piece-index.csv` the
single `B19` row carries the full upstream/downstream edge set; its **wave is W3** (the critical
path is the core), and B5 sits in W4.

## Non-blocking fan-out edges

Per §14.7, two fan-out edge groups express *"consumed by every component"* rather than *"must
precede"* — they are **continuously available inputs, not wave-depth-setting**:

- **B14 (test framework) → A, → B**
- **B22 (security design) → A, → B**

Likewise the **dotted toolset-contribution edges into A14** (`A13 -.-> A14`, `A2 -.-> A14`) and
the **CapabilitySet dotted reference edges** in §6.8 are non-blocking. These appear only as a
downstream note in the CSV and do **not** set wave depth.

## Views and ADRs are authoring-parallel context

Views (V6-01..V6-13, the §6.1–6.13 architecture sections) and ADRs (ADR-0001..ADR-0041) are
**not build pieces**. They are the frozen design context every component spec binds to. In
`piece-index.csv` they carry `wave=auth`, empty `upstream`/`downstream`, and no workstream — they
can be authored in parallel with everything else and impose constraints rather than consume
outputs.

## Reference

- `architecture-overview.md` §14.7 — dependency overview (mermaid DAG, foundation notes).
- `_meta/piece-index.csv` — the encoded per-piece wave / tier / edge data.
- `_meta/interface-contract.md`, `_meta/glossary.md` — the frozen names specs bind to.
