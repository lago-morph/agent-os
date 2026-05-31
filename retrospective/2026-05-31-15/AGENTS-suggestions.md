# AGENTS.md suggestions — 2026-05-31-15

Proposed additions to the project's agents file (`AGENTS.md` at the repo root). Each
section has the exact text to paste and the argument for it, grounded in this session.
Decide each on its own merits.

---

## Suggestion 1: Decisions vs. complaints

### Proposed addition

> **Bring decisions, not complaints.** Before surfacing an item to the user, bucket it:
> a *decision* needs their judgment (an architecture trade-off, an ownership call); a
> *defect* in something you produced (untestable AC, formatting drift, missing coverage)
> is yours to fix silently. Only decisions go to the user, and each one as: *what it is ·
> why it matters · options · your recommendation · what I need from you.* Never present a
> defect description as if it were a choice.
>
> *Grounded in: the user rejecting "QN-01…QN-05" — "give me decision points, not complaints."*

### Why this earns its place in your agents file

Twice in one session the user had to stop and say a surfaced item was the agent's job,
not theirs ("why is this even a question? This is your scope"). Each mis-bucketed item
cost a round-trip and visible frustration. The rule costs one triage step per item and
converts a noisy "here are 12 problems" into a clean "here are 3 choices, here's what I'd
do." It is the single highest-leverage behavioral fix from the session.

---

## Suggestion 2: Plain language in anything a human will read

### Proposed addition

> **Write human-facing docs in plain language.** In any document or message meant for a
> person to act on, lead with the plain-English name of a thing, not its internal code or
> file id. Spell out every acronym on first use. Provide a one-time "cast of characters"
> if codes are unavoidable. A reader must never have to look something up to understand a
> sentence.
>
> *Grounded in: the user rejecting the first decision register — "written for an AI, not a human."*

### Why this earns its place in your agents file

The first decision register ("the 4 events that A23/B5/B10/A14 bind to…") was unreadable
to the decision-maker and had to be fully rewritten — a wasted artifact plus a re-do.
Bare internal codes are free for the agent and expensive for the human. The rule costs a
few extra words per reference and a cast-of-characters block once; it prevents throwing
away whole documents.

---

## Suggestion 3: Verify a factual premise before you build on it

### Proposed addition

> **Verify before you assert.** Before stating a fact about a library, API, CRD,
> upstream tool, or this codebase — or before agreeing with a "X already does Y" premise —
> confirm it with a grep, a doc fetch, or a file read. Prefer the primary source. Flag
> what is inference vs. confirmed.
>
> *Grounded in: corrections this session — kagent was rejected for ARK; `AgentEnvironment`
> is a Crossplane XR; the agent-sandbox `NetworkPolicySpec` is L3/L4 only (so Envoy stays).*

### Why this earns its place in your agents file

Several confident-sounding claims were wrong until checked, and one (sandbox can replace
Envoy) would have driven a bad architecture decision had the API not been fetched. The
user explicitly values grounding ("get it right"). One grep or one doc fetch is cheap;
an architecture decision built on a wrong premise is not.

---

## Suggestion 4: Stay in scope — don't drag in baseline/platform concerns

### Proposed addition

> **Guard scope.** This project owns the agent runtime layer. Cluster baseline, platform
> provisioning, and infra-operations concerns (secret-at-rest/KMS, disaster recovery,
> node isolation, component autoscaling, SLOs, cluster health/alerting) are out of scope
> unless explicitly stated. Do not surface them as in-scope decisions or author
> requirements for them.
>
> *Grounded in: the OPS1–OPS6 layer over-reaching into baseline/platform scope, which
> propagated into the decision register and drew "stop creeping your scope."*

### Why this earns its place in your agents file

A subagent-authored "Operational/NFR layer" invented requirements for DR, node isolation,
autoscaling, and SLOs — none of which this project owns — and the agent piped them to the
user as decisions. Most of a six-piece layer turned out to be out-of-scope. A scope check
before authoring or surfacing a requirement is nearly free and prevents generating (and
having to retract) whole bodies of work.

---

## Suggestion 5: Confirm an edit actually changed the file

### Proposed addition

> **Verify mechanical edits landed.** After a `sed`/scripted edit, diff or grep to confirm
> the change actually applied before committing — and never write a commit message that
> claims a change you haven't verified. Note: repo files may use CRLF line endings, which
> silently break `$`-anchored `sed` patterns; prefer the Edit tool or an unanchored match,
> then verify.
>
> *Grounded in: a `^…$`-anchored `sed` that no-op'd on a CRLF line, producing a commit
> whose message claimed a fix that wasn't applied (caught on verify, corrected).*

### Why this earns its place in your agents file

A quota/csv fix was committed with a message asserting it was done; the `sed` had silently
matched nothing because the line ended in `\r`. Without the post-edit grep it would have
shipped a lie in the history. One verification grep per scripted edit is trivial; an
inaccurate commit record erodes trust in the whole audit trail.

---

## Suggestion 6: Bound what a single subagent must read

### Proposed addition

> **Don't hand one subagent the whole corpus.** A subagent's task must be sized to its
> context and time budget. Do not ask one agent to read a large set of files (tens of KB+)
> and synthesize; slice the work, or have the orchestrator synthesize from agents' brief
> reports. If a synthesis agent must read a lot, give it a targeted reading list.
>
> *Grounded in: a consolidator subagent timing out at ~28 minutes after being asked to
> read ~100 KB of review files; the orchestrator wrote the output by hand instead.*

### Why this earns its place in your agents file

One over-scoped agent burned ~28 minutes and produced nothing. The cost of slicing (or
synthesizing centrally) is a little orchestration; the cost of not slicing was a dead
agent and a manual redo. Pairs naturally with the existing subagent-prompting guidance.

---

## Suggestion 7: Record decisions in a durable ledger, and push immediately

### Proposed addition

> **Persist decisions as they're made.** Maintain a single decision ledger
> (`DECISIONS-LOG.md`) bucketed Decided / Future / Out-of-scope / Open. Record each ruling
> the moment it's made and push it — the sandbox is ephemeral and an unpushed decision is
> a lost decision. Keep "decision recorded" separate from "decision applied to the specs":
> they are different passes.
>
> *Grounded in: the session's rulings being captured in `DECISIONS-LOG.md` and pushed
> per-ruling, which made a clean cross-session handoff possible.*

### Why this earns its place in your agents file

Decisions made only in conversation evaporate at context truncation or session end. A
committed, pushed ledger turned a long interactive review into a resumable artifact a
fresh session can execute from. The marginal cost is one commit per ruling; the payoff is
that no decision has to be re-litigated.

---

## Suggestion 8: When fanning out writers, commit centrally

### Proposed addition

> **Subagents write files; the orchestrator commits.** When dispatching parallel agents
> that produce files in a shared working tree, instruct them not to run git at all; the
> orchestrator stages and commits centrally after each wave. This avoids index races and
> keeps history coherent. Have agents write to distinct paths to avoid collisions.
>
> *Grounded in: the 13-agent fan-out, where central commits kept a clean per-area history
> and avoided concurrent-git conflicts.*

### Why this earns its place in your agents file

Parallel agents each running `git add/commit/push` against one repo is a race waiting to
corrupt the index or the branch. Telling them to only write files — and committing
centrally — costs nothing and removes an entire class of concurrency bug.

---

## Suggestion 9: Don't elevate implementation details to decisions

### Proposed addition

> **Separate architecture decisions from build-time details.** Deployment topology
> (sidecar vs. gateway), exact field lists behind an abstraction, and "how" mechanisms are
> implementation choices made at build time — not standing architecture decisions. Don't
> put them in front of the user as decisions, and don't block on them.
>
> *Grounded in: flagging Envoy's sidecar-vs-gateway wiring as a "decision" — the user:
> "where Envoy lives is not important at this level… why is this one special?"*

### Why this earns its place in your agents file

Elevating an ordinary build detail to a ratification-blocking decision wastes the user's
attention and implies false precision. The rule is a quick self-check ("is this a
trade-off the user must own, or a detail an implementer will pick?") and keeps the
decision queue to things that actually need a human.
