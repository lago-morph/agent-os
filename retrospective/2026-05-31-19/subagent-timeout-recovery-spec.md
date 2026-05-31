# Spec: `subagent-timeout-recovery`

## Intent

When a long-running subagent hits a stream idle timeout, the naive response is to re-run it from scratch — which wastes time on files already completed and may timeout again at the same point. This skill provides a targeted recovery pattern: assess what landed, grep for what was missed, fix the gaps manually or with a small focused follow-up agent. Grounded in two real timeouts (Agents 2 and 5) during the Crossplane v2 corpus sweep, where checkpoint commits preserved ~80% of each agent's work and recovery took 15 minutes vs re-running would have taken 45+.

## Trigger

**Direct triggers:**
- An agent completes with "stream idle timeout" in the result.
- An agent that was launched for a large file set returns without updating all the files in its scope.
- `git status` shows some — but not all — of an agent's assigned files modified.

**Proactive triggers:**
- Any subagent that has been running > 30 minutes on a > 15-file task.
- The orchestrator notices an agent completion message but only partial files updated.

**Negative triggers:**
- The agent genuinely produced nothing (zero files changed) — that's a re-run, not a recovery.
- The gap is a single file — just fix it directly, no recovery pattern needed.

## Inputs

- The timed-out agent's scope: which files it was supposed to touch.
- The work description: what changes it was supposed to make (rename map, ruling list, etc.).
- Access to `git status` and `git diff` to assess what landed.

## Outputs

- All of the timed-out agent's scope updated (gap files fixed).
- A checkpoint commit capturing everything completed by the timed-out agent (if not already committed).
- A targeted verification grep confirming no residuals in the agent's scope.

## Workflow

### Step 1 — Commit what exists immediately

Before doing anything else:

```bash
git add -A
git status --short   # see what the agent wrote
git commit -m "Checkpoint: <agent-name> partial work (stream idle timeout)"
```

This is the single most important step. The sandbox is ephemeral. If the agent wrote 11 of 13 files and then timed out, those 11 files are on disk right now. Commit before investigating, before running any greps, before anything.

### Step 2 — Identify what the agent was supposed to do

Re-read the agent's brief (or the master task list):
- Which files were in scope?
- What changes were expected (rename map, ruling IDs, etc.)?

Cross-reference against `git status` / `git diff --name-only HEAD~1` to see what actually changed.

Produce a gap list: `expected_scope - completed_files = gaps`.

### Step 3 — Run targeted greps for residuals

Don't re-read every file. Grep for the specific terms that were supposed to be changed:

```bash
# Example: checking for v1 XRD names in the timed-out agent's file set
grep -rn "XPostgres\|XSearchIndex\|XObjectStore\|XMongoDocStore" \
  specs/components/A/   # or whichever slice timed out
grep -rn "\bclaim shape\b\|\bclaim admission\b\|\bclaim API\b" \
  specs/components/A/
```

For each hit: read 3–5 lines of context. Classify:
- **Genuine residual** — still has the old term, needs fixing.
- **False positive** — JWT/OIDC context, historical lineage note, or intentional exception; leave it.

### Step 4 — Decide: manual fix or mini-agent

**Fix manually** when:
- Fewer than ~15 individual lines need changing.
- The changes are mechanical (find-and-replace of a known term).
- Each change requires little judgment.

**Use a targeted mini-agent** when:
- More than ~15 lines across > 5 files.
- The changes require re-expression of logic, not just term substitution.
- The gap spans a coherent sub-task (e.g., "thread D-02 into all A-workstream plans").

For a mini-agent, write a new brief scoped only to the gap files. Include the same rename map and ruling list from the original brief. Explicit scope: "Touch ONLY these files: [list]. Do not re-edit files already updated."

### Step 5 — Apply fixes

**If fixing manually:**

For each gap file, read it, make the targeted edits, and verify the context is correct. A find-and-replace without reading is wrong — one occurrence per paragraph is usually fine; a paragraph that uses the old term throughout its logic may need re-expression.

**If using a mini-agent:**

Launch with `run_in_background: true`. When it completes, run `git status` and commit immediately.

### Step 6 — Verify the recovered scope

After all gaps are fixed:

```bash
grep -rn "OldTerm1\|OldTerm2\|..." path/to/recovered/scope/
# Target: zero hits (excluding classified exceptions)
```

If hits remain, classify them again. Fix genuine residuals. Document exceptions in a comment or note.

### Step 7 — Commit the recovery

```bash
git add -A
git commit -m "Recovery: <agent-name> timeout gaps fixed — <N> targeted edits"
git push -u origin <branch>
```

Label the commit as a recovery so it's clear in `git log` what happened.

## Concrete examples

### Example 1 — Agent 2 (Workstream A) timeout, Crossplane v2 sweep

**Situation:** Agent 2 was responsible for A1–A23 specs + plans. It timed out after ~45 minutes. `git status` showed 40+ A-workstream files modified.

**Step 1 — Commit immediately:**
```bash
git add -A
git commit -m "Crossplane v2 sweep + ruling threading: partial in-flight (checkpoint 2/4)"
```

**Step 2 — Gap identification:**
Expected: A1–A23 specs + A1–A23 plans = 46 files.
Completed: 40 files modified (per git diff).
Gap: A21 spec and plan (REQ-A21-13 added, but 3 other locations still said "quotas deferred").

**Step 3 — Targeted grep:**
```bash
grep -n "quotas.*deferred\|deferral\|defer" specs/components/A/spec-A21.md
```
Output: 3 hits — the ADR-0037 note, the §4.1 field note, and OQ-A21-5 status.

**Step 4 — Decision:** 3 lines, mechanical update → fix manually.

**Step 5 — Manual fixes:**
- ADR-0037 note: added "(quotas deferral reversed per #5/#9 in Session 3)"
- §4.1: changed "Role-catalog still deferred, quota still deferred" → "Role-catalog still deferred, quota now mandatory"
- OQ-A21-5: changed "low: quotas deferred per ADR-0037" → "resolved: quotas no longer deferred per #5/#9"

**Step 6 — Verify:**
```bash
grep -n "quotas.*deferred\|deferral" specs/components/A/spec-A21.md
# 0 hits
```

**Step 7 — Commit:**
```bash
git commit -m "A21 quota deferral conflict resolved — OQ-A21-5 marked resolved"
```

---

### Example 2 — Agent 5 (Views) timeout, Crossplane v2 sweep

**Situation:** Agent 5 was responsible for V6-01–V6-13 specs + plans (26 files). Timed out after writing ~22 of 26 files.

**Step 1 — Commit immediately:**
```bash
git add -A
git commit -m "Views v2 fix + ADR sweep progress (checkpoint 4/N)"
```

**Step 2 — Gap identification:**
```bash
grep -rn "XPostgres\|XSearchIndex\|\bclaim shape\b\|ADR 0041\b" specs/views/ plans/views/
```
Hits in: `specs/views/spec-V6-09.md`, `plans/views/plan-V6-09.md`, `specs/views/spec-V6-12.md`.

**Step 3 — Read hits in context:**
- `spec-V6-09.md`: "spanning claim-consumption contract" (line 47), "TenantOnboarding claim" (AC-06), "claim's presence" (§10) — all Crossplane-context, genuine residuals.
- `plan-V6-09.md`: "claim-consumption invariants" (task description), "apply TenantOnboarding claim" (AC-06) — genuine.
- `spec-V6-12.md`: "ADR 0041 / V6-03" in scope-out section — needs updating to 0044.

**Step 4 — Decision:** 6 lines across 3 files → fix manually.

**Step 5 — Manual fixes:** Read each file, applied targeted Edit calls.

**Step 6 — Verify:**
```bash
grep -rn "claim-consumption\|\bclaim shape\b\|ADR 0041" specs/views/ plans/views/
# 0 hits
```

**Step 7 — Commit:**
```bash
git commit -m "Views v2 residuals fixed — V6-09 and V6-12 targeted edits"
```

## Anti-patterns

- **Re-running the full agent from scratch.** The timed-out agent wrote 11 of 13 files correctly. Re-running it rewrites those 11 files (wasting time, potentially introducing drift) and may time out again at the same point. Always scope recovery to the gap only.
- **Investigating before committing.** The sandbox is ephemeral. If you run three greps before committing, and the sandbox recycles, all the timed-out agent's disk work is gone. Commit first, investigate second.
- **Treating all "old term" grep hits as residuals.** Some hits are intentional: JWT/OIDC "claims" are a different concept; historical lineage notes quoting the old spec are correct. Classify before fixing.
- **Writing a mini-agent brief that overlaps already-completed files.** "Do all views" when 11 of 13 are already done risks re-expression drift or accidental revert on the 11. Scope the brief to gap files only, with explicit "do not touch" list for completed files.
- **Skipping the verification grep.** Recovery fixes are targeted and can have their own gaps. A final grep over the recovered scope confirms zero residuals and takes < 60 seconds.
- **Losing the recovery in the commit history.** Label recovery commits clearly ("Recovery: [agent-name] gaps") so `git log` distinguishes original checkpoint commits from gap-fill commits.

## Acceptance criteria

1. After recovery, `grep -rn "OldTerm" <agent-scope-dirs>` returns zero genuine (non-exception) hits.
2. All files in the timed-out agent's scope have been modified (git log confirms each file touched at least once).
3. The checkpoint commit (Step 1) exists in `git log` with a timestamp before any recovery edits — proving the original agent's work was preserved.
4. The recovery commit is labeled distinctly from the original checkpoint commit.
5. Total time from timeout detection to recovery complete is < 20 minutes for a gap of ≤ 15 files.

## Files this skill creates / modifies

| Path | Description |
|---|---|
| Any files in the timed-out agent's scope | In-place fixes for residual old-terminology or missed rulings |
| (No new files created by this skill) | Recovery edits go to the gap files directly |
