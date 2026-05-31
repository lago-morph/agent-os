# ADR 0043: `[PROPOSED — not in source]` as the mandatory not-invented marker

- **Status**: Accepted
- **Date**: 2026-05-30

## Context

When agents author detailed specs and plans from the architecture documents, they
routinely reach the edge of what the source actually states. A spec for the audit
adapter needs a field the overview never enumerates; a plan for an MCP service needs
an event-type name the CloudEvent taxonomy (ADR 0031) leaves to the schema registry
(B12); a plan for persistent-volume access (B20) needs an entire CRD the architecture
never defines.

At that edge an agent has two choices: **invent** a plausible name/field/value and
assert it as fact, or **flag** the gap. Invention is the dangerous default — a
fabricated CRD field is indistinguishable from a real one to a future reader, so it
becomes a landmine that downstream implementers build conflicting code against. In the
first large spec/plan run (PR #8), the discipline of flagging instead of inventing
produced 196 explicit markers across 95 files — which together formed the actionable
backlog of genuine source gaps (the missing B20 mechanism, deferred event-type names,
per-primitive secret fields, and so on), rather than a corpus of confident hallucination.

## Decision

**Authoring agents must flag, not fabricate. The mandatory marker is the literal
string `[PROPOSED — not in source]`.**

- When a spec/plan needs a name, field, value, or interface that the architecture
  documents (overview, ADRs, backlog) and the frozen Canon do **not** state, the agent
  writes its best proposal followed by `[PROPOSED — not in source]` — never asserting
  it as established fact.
- The em-dash spelling is fixed so the marker is reliably greppable:
  `grep -rn 'PROPOSED — not in source'` over the corpus yields the complete,
  deduplicated list of real source gaps.
- The marker is paired with the frozen Canon (ADR 0042): the Canon tells an author
  what the authoritative names are; this marker is what the author uses when the needed
  name is not among them.
- Load-bearing proposals (anything touching a contract other pieces consume) should
  also be surfaced in the piece's "Risks & Open Questions" section so the gap is
  visible to a reader who is not grepping.

## Consequences

- Hallucination becomes visible and auditable instead of silent. The cost is a few
  words inline; the payoff is that the entire set of genuine gaps is a single search.
- The marker is the hand-off surface to the design backlog: each flag is a decision the
  architecture still owes, and many already trace to `architecture-backlog.md` deferrals.
- The exact string is part of the contract — variants (different dash, different
  wording) break the grep and must be avoided. Tooling and reviewers should treat the
  literal `[PROPOSED — not in source]` as canonical.
- Operationalized for agents by the `canon-freeze-before-fanout` skill and recorded as
  an operating rule in `AGENTS.md`.

## References

- [ADR 0042](./0042-spec-plan-corpus-structure-and-frozen-canon.md) (spec/plan corpus structure and the frozen Canon this marker pairs with)
- [ADR 0031](./0031-cloudevent-top-level-taxonomy.md) (event-type names deferred to B12 — a common source of `[PROPOSED]` flags)
- [architecture-backlog.md](../architecture-backlog.md) (deferred decisions many flags trace to)
- `retrospective/2026-05-30-8.md` (the session that established this convention; 196 flags across 95 files)
