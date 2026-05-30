# Spec: `canon-freeze-before-fanout`

## Intent

When many independent authoring agents write parts of a single coherent artifact set in
parallel, they drift: the same entity gets spelled three ways, fields get invented, ADR
or section numbers get miscited. This skill eliminates that class of failure by
extracting an authoritative, frozen "Canon" from the source material *once, serially,
before any fan-out*, then binding every downstream agent to it. In the session that
produced this skill, 21 parallel agents wrote 238 files describing one platform; without
a frozen Canon listing the 31 CRD/XRD names, the CloudEvent taxonomy, and the glossary,
the corpus would have contained dozens of incompatible names. With it, drift was zero and
genuine gaps surfaced as explicit flags instead of silent fabrications.

## Trigger

- Direct: "freeze the canon", "build a shared vocabulary before fanning out", "make a
  glossary/interface contract for the agents".
- Proactive: before dispatching ≥3 agents that will independently produce files
  referencing a shared set of named entities (CRDs, APIs, events, modules, terms).
- Negative: do NOT use for a single-agent task, or when agents produce fully independent
  outputs that share no vocabulary.

## Inputs

- The source material (architecture docs, schemas, an existing glossary section, an
  inventory table).
- The list of pieces/units to be authored in parallel.
- Any naming conventions the project already uses.

## Outputs

- A `_meta/` (or equivalent) directory containing:
  - `glossary.md` — every proper noun an author could misspell, with canonical
    casing + one-line meaning. Marked FROZEN.
  - `interface-contract.md` — the cross-piece registry: every shared entity (CRD,
    API, event type, schema) → owning unit + the fields the *source actually states*
    (gaps marked "not specified in source", never invented).
  - `piece-index.csv` — one row per unit with id/kind/tier/dependencies/etc.
  - Optionally a wave/layering doc derived from the dependency graph.

## Workflow

1. Identify the source's authoritative naming surfaces (inventory tables, glossary,
   schema sections, decision records).
2. Dispatch **one** agent (serial, foreground) to extract the Canon. Instruct it
   explicitly: extract what the source *says*; do not invent; mark genuine gaps as
   "not specified in source"; capture exact casing.
3. Require the agent to return a validation summary: entity count captured, row count,
   and **any contradictions or gaps it found in the source** (these are high-value).
4. Spot-check the Canon yourself (counts, a few entity spellings, the gaps section).
   Fix any errors the extraction agent introduced before freezing.
5. Commit the Canon. Treat it as immutable for the run.
6. In every downstream author's prompt, include: "MANDATORY FIRST STEP — read the Canon
   files. Use ONLY names from the Canon. Anything not in the Canon must be tagged
   `[PROPOSED — not in source]`, never asserted."
7. After the run, grep for the `[PROPOSED]` tag — it is your complete, deduplicated list
   of real source gaps.

## Concrete examples

**Example 1 (this session).** Source: `architecture-overview.md` §6.12 (CRD inventory),
§15 (glossary), 41 ADRs. The Canon agent produced `_meta/glossary.md` (capturing exact
casings like `MCPServer`, `XPostgres`, `A2APeer`), `_meta/interface-contract.md` (31
CRDs → owning component, the closed set of 10 `platform.*` CloudEvent namespaces, and an
explicit "Gaps — do not invent" section), and a 119-row `piece-index.csv`. The agent
caught that B4 (Crossplane Compositions) was absent from the orchestrator's wave list
despite owning every substrate XRD — a real bug fixed before fan-out. Downstream: 196
`[PROPOSED]` flags across 95 files, zero name drift.

**Example 2 (generalized).** Building OpenAPI specs for 12 microservices in parallel.
Freeze a Canon of shared DTO names, error-code enum, and auth-header contract first.
Each service-spec agent binds to it; a service needing a DTO field not in the Canon
writes `balance: number [PROPOSED — not in source]` rather than inventing `accountBalance`
vs `bal` vs `balanceAmount` across three specs.

## Anti-patterns

- **Letting each author re-derive vocabulary from the raw source.** Every author
  re-reading a 150 KB doc both wastes budget and produces divergent readings.
- **Allowing the Canon to be edited mid-run.** A change at the base invalidates work
  above it. Freeze it; if it must change, that's a deliberate Canon revision, not a
  silent edit.
- **Inventing fields to fill gaps.** The session's discipline was the opposite: surface
  gaps as flags. A Canon that fabricates is worse than no Canon.
- **Skipping the human spot-check of the Canon.** It is the single load-bearing artifact;
  an error in it propagates to every downstream file.

## Acceptance criteria

1. A frozen Canon exists and is committed before any parallel authoring begins.
2. The Canon's entity list matches the source's authoritative inventory (verifiable count).
3. Every author prompt references the Canon and the `[PROPOSED]` tagging rule.
4. Post-run, a single grep produces the complete gap list.
5. No two output files spell the same shared entity differently.

## Files this skill creates / modifies

- `_meta/glossary.md` — frozen canonical vocabulary.
- `_meta/interface-contract.md` — frozen cross-piece interface registry.
- `_meta/piece-index.csv` — the unit manifest.
- `_meta/waves.md` (optional) — dependency-derived build layering.
