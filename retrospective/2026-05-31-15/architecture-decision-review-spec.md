# Spec: `architecture-decision-review`

## Intent

When a large design corpus (many specs/plans/ADRs) has been authored but not yet
ratified, the open architectural questions are scattered across dozens of files and
phrased in internal shorthand. This skill runs a **structured, interactive decision
review**: it harvests the open questions into a triaged register, drives a
human-readable walkthrough where the user rules on each, and maintains a durable
**decision ledger** that survives session loss and feeds a later implementation pass.
In the session that produced this spec, ~30 open items were resolved this way over a
single corpus, and the ledger (`DECISIONS-LOG.md`) became the authoritative input to
the next session's work. The skill exists because doing this ad hoc produced two
repeated failures: surfacing non-decisions as if they needed the user, and writing the
register in jargon the user refused to read.

## Trigger

- Direct: "review the decisions", "what's left to decide?", "walk me through the open
  questions", "let's ratify the spec", "triage the open architecture questions".
- Proactive: after a multi-piece spec/plan authoring run lands, or after an adversarial
  review produces a list of contradictions/`[PROPOSED]` items needing a human ruling.
- Negative: a single isolated design question (just answer it); routine code review
  (use `code-review`); implementation work (this skill decides, it does not build).

## Inputs

- A corpus of specs/plans/ADRs (and any frozen "Canon": glossary, interface-contract,
  piece index).
- Optional prior review artifacts (red-team findings, consistency reports).
- The repo's existing ADR set and decision-log conventions, if any.

## Outputs

- `…/DECISIONS-LOG.md` — the authoritative ledger, bucketed: **Decided · Future-version ·
  Out-of-scope · Still-open · My-scope-cleanup**. Each ruling is one self-contained entry.
- A plain-language `OPEN-DECISIONS.md` for the items still needing a human ruling.
- A list of **proposed-ADR titles** (no specs) for decisions the user may formalize.
- Updates to the project handoff so a fresh session can resume.

## Workflow

1. **Harvest** the open items: grep the corpus for `[PROPOSED]`/TODO markers, gather
   red-team findings and contradictions, and read each piece's open-questions section.
   Produce a flat list.
2. **Triage every item into exactly one bucket**: (a) *decision* — needs the user's
   judgment; (b) *my-scope cleanup* — a quality defect I should just fix (untestable AC,
   formatting, missing coverage); (c) *out-of-scope* — belongs to a layer this project
   does not own; (d) *future-version* — real but post-MVP. **Only bucket (a) goes to the
   user.** Mis-bucketing (b)/(c) as (a) is the dominant failure mode.
3. **Write the register in plain language.** Lead every item with a human name, never a
   bare code; include a one-time "cast of characters" mapping codes → plain names. For
   each decision give: *what it is · why it matters · options · recommendation · what I
   need from you.* Do **not** present a "complaint" (a defect description) as a decision.
4. **Walk the user through the decisions** in small clusters. For each ruling: record it
   in `DECISIONS-LOG.md` immediately and push (sandbox is ephemeral). When the user
   gives a fact-shaped premise ("X already handles this"), **verify it against the corpus
   before agreeing** (grep / fetch docs).
5. **Distinguish decisions from implementation details.** Deployment topology, exact
   field lists behind an abstraction, "how" mechanisms — these are build-time, not
   standing decisions. Do not elevate them.
6. **Surface proposed-ADR titles** (titles + one-line rationale only) for decisions that
   outlive the session. Do not auto-author ADRs.
7. **Close out**: update the handoff with done/next state; leave the corpus edits
   (applying the rulings to the spec files) as a clearly-scoped *later* pass — recording
   a decision and applying it are different jobs.

## Concrete examples

**Example A — a real "complaint" that was NOT a decision.** The register listed
"QN-02: inconsistent `canon-*` front-matter hash formatting." The user replied: *"why is
this even a question? This is your scope, not mine."* Correct handling: move it to the
*my-scope cleanup* bucket and fix it silently; never surface it as a user decision. The
skill's step-2 triage exists to catch exactly this before it reaches the user.

**Example B — a fact-premise that needed verification.** The user said the Kubernetes
agent-sandbox extension could likely replace Envoy. Instead of agreeing, the skill
fetched the API: `NetworkPolicySpec` exposes only `ingress`/`egress` rendered to native
NetworkPolicy — **L3/L4 only**, no FQDN/method control. Conclusion recorded: Envoy stays
for L7; the sandbox NetworkPolicy is a complementary L3/L4 baseline. Verifying changed
the decision.

## Anti-patterns

- **Complaints dressed as decisions.** "Spec X has untestable ACs" is your bug to fix,
  not a user choice. (Session: the user pushed back twice on this.)
- **Bare internal codes in user-facing text.** "A23/B5 bind to B12" is unreadable; the
  user rejected an entire register for it.
- **Agreeing with a premise without checking it.** Verify "X already does Y" against the
  corpus before building a decision on it.
- **Elevating implementation details** (sidecar vs gateway) to standing decisions.
- **Scope creep.** Surfacing baseline/platform/cluster concerns as in-scope decisions.
- **Letting the ledger live only in context.** Record and push each ruling as it lands.

## Acceptance criteria

1. Every open item lands in exactly one of the four buckets; only *decisions* reach the user.
2. The register is readable by a non-engineer (no unexplained codes; cast of characters present).
3. Each decision entry has what/why/options/recommendation/what-I-need.
4. Every ruling is recorded in the ledger and pushed before moving on.
5. Proposed-ADR titles are surfaced without specs; no ADR is auto-authored.

## Files this skill creates / modifies

- `…/DECISIONS-LOG.md` — the authoritative bucketed ledger (create/append).
- `…/OPEN-DECISIONS.md` — plain-language deep-dive on still-open items (create/update).
- Project handoff doc — done/next state (update).
