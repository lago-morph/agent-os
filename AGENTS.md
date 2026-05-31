# AGENTS.md

Operating rules for AI agents working in this repository. These are distilled from
real sessions; each rule names the failure it prevents. See `retrospective/` for the
sessions that produced them.

## Parallel authoring and multi-agent fan-out

**Freeze a Canon before fanning out.** Before dispatching three or more agents that will
independently produce files referencing shared named entities (CRDs, events, APIs,
modules, glossary terms), first run ONE serial agent to extract a frozen Canon into
`_meta/` (`glossary.md`, `interface-contract.md`, `piece-index.csv`). Every downstream
author must read it first and use ONLY its names. Spot-check the Canon before freezing —
it is the single load-bearing artifact. *Without it, independent agents drift
deterministically: the same entity gets three spellings and fields get invented, and the
errors look authoritative so they propagate.* (See the `canon-freeze-before-fanout` skill.)

**Flag gaps, never fabricate.** When authoring a spec, plan, or doc from source material,
use only facts the source states. If you need a name, field, or value the source doesn't
provide, write it followed by `[PROPOSED — not in source]` rather than asserting it as
fact. After a run, `grep -rn 'PROPOSED — not in source'` is the complete gap list. *A
fabricated CRD field is indistinguishable from a real one to a future reader — the flag
converts silent hallucination into a visible, greppable backlog.* (Formalized in ADR 0043.)

**Subagents return compact summaries, not file contents.** When dispatching authoring
agents that write to disk, instruct each to return only: files written, flags raised, and
open questions — never the file bodies. *The orchestrator tracking a wide fan-out is the
real bottleneck; full file bodies in returns saturate its context and it loses track of
which agents finished.*

**Run a dedicated adversarial planner in parallel for high-stakes plans.** Before any
execution that dispatches many agents or is costly/hard to reverse, run a dedicated
adversarial planner alongside the cooperative ones. Separate its *findings* (adopt the
quality controls) from its *conclusions* (which explicit user intent may override).
Surface the tension at the go/no-go gate. *A lone planner shares the optimism bias of
whoever will execute; the red-teamer routinely finds the failure mode that would cost the
most.* (See the `cooperative-adversarial-planning` skill.)

## Persistence and completeness

**Checkpoint progressively during long parallel runs.** When a multi-agent run will last
more than a few minutes, commit and push completed work in waves rather than only at the
end. The remote execution sandbox is ephemeral and can restart at any time; only
committed-and-pushed work survives. *A real mid-run container restart lost zero committed
work because of progressive checkpoints.*

**Verify fan-out completeness from disk, against a manifest — never from self-reports.**
After a fan-out expected to produce an enumerable file set, commit the working tree, then
check each expected path from the manifest for existence AND minimum substance (size +
required sections). Gap-fill only the genuinely-missing/stub set. *Agents report "done"
optimistically and report files "already existing" when they merely see a peer's work —
neither is proof; only a disk check against an explicit manifest is.* (See the
`coverage-manifest-gapfill` skill.)
