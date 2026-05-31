---
name: architecture-decision-review
description: Triage a body of review findings against a spec/plan corpus into four buckets — decisions the user must rule on, my-scope cleanup I just fix, out-of-scope items to record-and-decline, and future-version backlog — keeping a single decisions log as the source of truth and verifying every architectural claim against the source before asserting it. Use when reviewing or consolidating findings over an architecture corpus (specs, plans, ADRs, a Canon), when the user asks "what needs a ruling vs what should you just fix?", "triage these review findings", "build a decisions log", "what's in scope?", or when a review surfaced a mix of contradictions, nits, and open design choices that must be sorted before build. Trigger phrasings include "go through the review findings", "which of these are actually decisions", "log the rulings", "is that your scope or mine?". Do NOT use for reviewing a single small diff (use a code-review skill), for authoring one ADR from a settled decision (use the `adr` skill), or for harvesting session lessons (use `self-retrospective`).
---

# Skill: architecture-decision-review

A red-team / consolidation pass over an architecture corpus produces a pile of findings of
wildly different kinds: a frozen-contract contradiction that blocks the build, a cosmetic
formatting nit, a genuinely-open design choice nobody has made, and a "this isn't even our
job" scope-creep item — all mixed together with the same severity styling. Treated as one
flat list, the build stalls: the user gets asked to rule on cosmetic typos, and real
blockers get buried under backlog. This skill imposes the **four-bucket triage** that sorts
every finding by *who acts and when*, records rulings in one authoritative log, and forces
**verify-before-asserting** on every architectural claim — so the log never enshrines a
wrong "fact."

## When to use

- After a review/consistency sweep over a spec/plan/ADR corpus produced a heterogeneous
  set of findings that must be sorted before build can proceed.
- When the user asks "what needs a ruling vs what should you just fix?", "triage these
  findings", "build/maintain the decisions log", or "is that your scope or mine?".
- Proactively when you notice you are about to ask the user to decide something that is
  plainly mechanical cleanup, or about to silently "fix" something that is actually a
  binding architectural choice.
- NOT for a single small diff (use a code-review skill), authoring one ADR from a settled
  decision (use the `adr` skill), or harvesting session lessons (use `self-retrospective`).

## The four buckets (the core)

Every finding lands in exactly one of these. The classifying question is **who acts, and
when** — not how severe it is.

| Bucket | Who acts | When | Examples from the agent-os review |
|--------|----------|------|-----------------------------------|
| **DECISIONS** | the user rules | now (gates the build) | D-01 authMode enum, D-02 approval-event owner, #5/#9 quota enforcement, #24–26 policy bundle |
| **MY-SCOPE CLEANUP** | I just fix it | now (no ruling needed) | QN-02 hash-format normalization, QN-04 untestable ACs, D-09 csv reciprocity |
| **OUT OF SCOPE** | nobody (record & decline) | never (it's not agent-os's job) | #6/#7/#8 cluster isolation, #13–17 disaster recovery, KMS/rotation cadence |
| **FUTURE VERSION** | deferred to a later release | post-MVP | #11 cost showback, #18 CRD migration, #27 break-glass |

The dangerous confusions this prevents:

- **Complaint-vs-decision.** A finding *flagged* as needing attention is not automatically a
  decision the user must make. QN-02 ("inconsistent `canon-*` front-matter hash formatting")
  arrived in the consolidated `DECISIONS-NEEDED.md` register as a "quality nit." But it
  requires no architectural ruling — it is mechanical normalization. It belongs in
  MY-SCOPE CLEANUP, and the right move is to fix it, not ask. Surfacing it as a "decision
  needed" wastes the user's attention on a typo.
- **Cleanup-masquerading-as-decision** and its inverse, a real choice silently "cleaned up"
  by an agent that didn't realize it was binding. The bucket test — *who acts, and when* —
  catches both.

## Workflow

1. **Gather the raw findings.** Collect from every review source — red-team reviews,
   consistency sweeps, the §10 open-questions of each spec. Give each a stable ID
   (`D-01`, `QN-02`, `#5`) so it can be tracked across the log even as its bucket changes.
2. **Bucket each finding by the "who acts, and when" test**, not by severity:
   - Does resolving it require an *architectural ruling only the user can make*? →
     **DECISIONS**.
   - Is it mechanical / objective with one correct fix (formatting, untestable-AC rewrite,
     a dependency-graph reciprocity error)? → **MY-SCOPE CLEANUP** — note it, then fix it,
     no ruling sought.
   - Is it a baseline / platform / cluster concern outside this project's mandate? →
     **OUT OF SCOPE** — record *why*, so it is never re-raised.
   - Is it real and in-scope but not v1? → **FUTURE VERSION**.
   - If genuinely undecided, leave it in **STILL OPEN** until ruled.
3. **Verify before asserting any architectural claim.** Before writing a recommendation or
   a ruling that rests on a technical fact ("X can't do Y, therefore keep Z"), confirm the
   fact against the source/spec — do not assert from memory. (See the NetworkPolicySpec
   example below.) An unverified "fact" baked into the log propagates to every consumer.
4. **Fix the MY-SCOPE CLEANUP items immediately.** These need no ruling. Do them, and record
   them in the log *for traceability only* (e.g. "D-09 — resolved; csv reciprocity fixed"),
   so a reader knows they were handled and need not revisit.
5. **Present only the DECISIONS bucket to the user for a ruling** — tightly, each with the
   conflict in one or two lines and a recommended direction. Do not pad the ask with cleanup
   or out-of-scope items.
6. **Record every ruling in one decisions log** (`_meta/reviews/DECISIONS-LOG.md` in this
   repo) — the single source of truth, organized by the four buckets plus STILL OPEN. When a
   STILL-OPEN item is ruled, *move* it into DECIDED (don't duplicate); when a DECISION turns
   out to be cleanup, move it to MY-SCOPE CLEANUP. The log's bucket headers are the contract.
7. **Flag, don't apply, corpus-wide consequences.** A decision often implies edits across
   many files (e.g. the OPS-layer trim, the v1→v2 sweep). Record the consequence and the
   recommended action in the log, and get explicit go-ahead before touching the corpus —
   never unilaterally rewrite ~270 references off the back of one ruling.
8. **Propose ADRs, don't author them here.** Decisions that deserve a formal record become
   *proposed-ADR titles*; the user decides per ADR whether to author it (via the `adr`
   skill). This skill triages and logs — it does not write ADR bodies.

## Concrete examples

### Example 1 — complaint vs decision: "QN-02 is your scope, not mine"

**Input.** The consolidated register `_meta/reviews/DECISIONS-NEEDED.md` lists, under
"Quality nits (non-blocking)":

> **QN-02** — Inconsistent `canon-*` front-matter hash formatting. Lengths 8/12/16/40 +
> mixed "FROZEN"/file-ref styles. Cosmetic; normalize in a sweep.

**Wrong move.** Carry QN-02 forward as a "decision needed" and ask the user which hash
length to standardize on. This treats a mechanical nit as an architectural ruling and
burns the user's attention.

**Right move (apply the bucket test).** *Who acts, and when?* Normalizing front-matter
hashes is objective mechanical cleanup with one correct outcome — there is no architecture
in it. It goes in **MY-SCOPE CLEANUP**. The log records it as:

> ## MY-SCOPE CLEANUP (not decisions — I fix these, no ruling needed)
> - **QN-02** — inconsistent `canon-*` front-matter hash formatting. Mechanical
>   normalization.

The user is never asked. The item is fixed and noted for traceability. (Same treatment for
QN-04's untestable ACs and the D-09 csv-reciprocity fix — both objective, both MY-SCOPE.)

### Example 2 — verify before asserting: sandbox `NetworkPolicySpec` is L3/L4

**Input.** Finding D-08 raises an egress contract: should the platform keep the Envoy L7
proxy, or can the Kubernetes-native agent-sandbox network policy cover egress control?

**Wrong move.** Assert from memory "native NetworkPolicy can do FQDN allowlisting, so drop
Envoy" — and bake that into the ruling. If the assumption is wrong, every downstream egress
decision inherits the error.

**Right move (verify first).** Confirm the capability against the source/spec: Kubernetes
`NetworkPolicy` (the sandbox `NetworkPolicySpec`) operates at **L3/L4 only** — IP/CIDR and
port. It cannot express FQDN or HTTP-method allowlisting. The platform's `EgressTarget`
needs `fqdn` / `scheme` / `allowedMethods`, which is **L7**. Only after verifying does the
ruling get written, and it is now correct:

> ### Egress (D-08)
> - **Envoy stays.** The agent-sandbox `NetworkPolicySpec` is **L3/L4 only** (IP/CIDR/port,
>   native NetworkPolicy); it cannot do FQDN or HTTP-method allowlisting... so Envoy remains.
> - **Two layers:** sandbox `NetworkPolicySpec` = L3/L4 default-deny baseline; Envoy = L7
>   (FQDN/method) for external egress.

The verified fact ("L3/L4 only") is what makes the two-layer ruling sound. Asserting the
opposite from memory would have produced a confidently-wrong architecture.

## Anti-patterns

- **Presenting cleanup as a decision.** Asking the user to rule on a cosmetic hash-format
  nit (QN-02) or an objectively-broken AC (QN-04). If there is one correct fix and no
  architecture in it, it is MY-SCOPE CLEANUP — fix it.
- **Silently "fixing" a real architectural choice.** The inverse error: an agent normalizes
  something that was actually a binding decision. Run the bucket test before touching it.
- **Asserting a technical fact from memory.** Writing "native NetworkPolicy does FQDN" into
  a ruling without verifying. Verify against the source first; a wrong "fact" in the log
  propagates everywhere.
- **One flat severity-sorted list.** Sorting findings by HIGH/BLOCKER/nit instead of by
  bucket. Severity tells you urgency *within* the DECISIONS bucket; it does not tell you
  which bucket a finding belongs in.
- **Re-raising out-of-scope items.** Failing to record *why* something is out of scope, so
  it gets surfaced again next review. The OUT OF SCOPE bucket exists to record-and-decline
  permanently.
- **Unilaterally applying a corpus-wide consequence.** Sweeping ~270 references or trimming
  six OPS specs off one ruling without explicit go-ahead. Flag the consequence; wait for go.
- **Authoring ADR bodies here.** This skill surfaces proposed-ADR titles and logs rulings;
  it does not write ADRs. Hand those to the `adr` skill when the user opts in.
- **Two sources of truth.** Leaving a ruled item in both STILL OPEN and DECIDED. Move, don't
  duplicate — the log has exactly one home per finding.

## Acceptance criteria

1. Every finding is in exactly one bucket (DECISIONS / MY-SCOPE CLEANUP / OUT OF SCOPE /
   FUTURE VERSION / STILL OPEN), classified by the "who acts, and when" test, not severity.
2. The user is asked to rule on **only** the DECISIONS bucket — no cleanup or out-of-scope
   items padded into the ask.
3. Every MY-SCOPE CLEANUP item is fixed and recorded for traceability; no ruling was sought
   for any of them.
4. Every architectural claim in a ruling was verified against the source before being
   asserted (no memory-only "facts").
5. One decisions log is the single source of truth; each finding has exactly one home, and
   ruled items were *moved* (not duplicated) into DECIDED.
6. Corpus-wide consequences are flagged with a recommended action and left un-applied until
   the user gives explicit go-ahead.

## Files this skill creates / modifies

- `_meta/reviews/DECISIONS-LOG.md` — the authoritative four-bucket log; the single source of
  truth for what was decided, deferred, declined, or self-fixed. Created/updated here.
- `_meta/reviews/DECISIONS-NEEDED.md` (and a plain-language `-PLAIN.md` variant) — the
  consolidated raw-findings register that feeds the triage; read in, annotated with bucket
  assignments.
- Corpus files under `specs/`, `plans/`, `adr/` — touched only for MY-SCOPE CLEANUP fixes,
  or for decided changes *after* explicit go-ahead; never swept unilaterally off one ruling.
