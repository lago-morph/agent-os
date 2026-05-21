---
name: benchmark-subagent
description: Run a side-by-side benchmark of Claude models by dispatching the same prompt to multiple subagents with different `model` overrides (typically Opus, Sonnet, Haiku) in parallel, then analyzing differences in correctness, reasoning depth, latency, and token usage. Use when the user wants a head-to-head capability comparison — i.e., a benchmark-synonym (benchmark, compare, evaluate, test, race, shootout, bake-off, head-to-head, A/B, rank, "stack up", "pit against", vs, versus) appears in close proximity to a target-synonym (model, models, LLM, LLMs, Claude, Opus, Sonnet, Haiku, AI, or any word containing "agent" as a substring — agent, agents, subagent, subagents, multi-agent, agentic). All case combinations apply. Example trigger phrasings — "let's compare models", "benchmark the LLMs on X", "race Sonnet vs Haiku on Y", "which agent is smarter at Z", "evaluate the subagents on W", "how does Opus stack up against Sonnet", "model bake-off", "shootout between Claudes". Do NOT trigger on "which model should I use in production" (recommendation, not benchmark), discussion of agent architecture without comparative intent, or "agent" appearing in unrelated contexts (HTTP user-agent strings, etc.).
---

# Skill: benchmark-subagent

Run a side-by-side capability benchmark: identical prompt, multiple Claude variants, dispatched in parallel via the `Agent` tool's `model` override, then analyzed.

The dispatcher (you) owns prompt assembly, parallel dispatch, result collection, and the comparison write-up.

---

## Inputs

| Input | Default | Notes |
|-------|---------|-------|
| `PROBLEMS` | the two defaults below | Problem(s) the user wants tested. One or more. If user supplied one, use only that one. |
| `MODELS` | `[default, sonnet, haiku]` | Models to compare. `default` means omit the `model` parameter — inherits from parent (typically Opus in this harness). |
| `THINK_HINT` | true | Prepend "Think carefully before answering." to each problem. |
| `SELF_REPORT` | true | Ask each subagent to self-identify (model ID + whether extended thinking was used). |

---

## Step 1 — assemble the prompt(s)

For each problem, build an identical prompt that includes:

1. The problem statement.
2. The think-hint (if enabled): "Think carefully before answering."
3. Specific deliverables (e.g., "show all reasoning", "give the final answer in a boxed equation").
4. The self-report block (if enabled), appended verbatim:
   > After your answer, on a separate line, report:
   > - The exact model ID you are running on (check your system prompt / self-knowledge).
   > - Whether you used extended thinking / deep thinking mode for this response (yes/no, and how you know).

The same prompt string goes to every model — do not customize per model.

---

## Step 2 — dispatch in parallel

For each `(problem, model)` pair, call the `Agent` tool with:

- `subagent_type`: `"general-purpose"`
- `model`: the model name from `MODELS` (omit the parameter entirely for the `default` slot)
- `description`: `"Benchmark — <model> — <short problem name>"`
- `prompt`: the assembled prompt from Step 1

**All Agent calls must go out in a single assistant message** so the harness runs them concurrently. If you split them across messages, they serialize and the latency comparison is corrupted.

If `MODELS` has 3 entries and `PROBLEMS` has 2 entries, that's 6 parallel subagents.

---

## Step 3 — collect results

Each Agent result returns the answer plus a `<usage>` block containing `total_tokens`, `tool_uses`, and `duration_ms`. Record all three per subagent, alongside the subagent's self-reported model ID.

---

## Step 4 — analyze

For each problem, score each model on:

- **Correctness**: matches the known solution? (`✓` / `partial` / `✗`)
- **Key insights captured**: list 2–4 critical reasoning steps the answer needed to hit; mark each hit/missed.
- **Failure mode** (if wrong): describe the specific error — over-simplification, wrong lemma, computational slip, ignored a constraint, etc.
- **Self-reported model ID**: what the subagent claimed (cross-check against the `model` parameter you passed).
- **Duration**: from `<usage>`.
- **Tokens**: from `<usage>`.

---

## Step 5 — present the comparison

Output **one table per problem**:

| Model | Correctness | Key insight hit/missed | Failure mode | Self-reported ID | Duration | Tokens |
|---|---|---|---|---|---|---|

Then a 3–5 sentence synthesis covering:

- Which model was strongest on this problem and why.
- Where weaker models broke down (be specific — name the conceptual step they fumbled).
- Whether the capability ordering matched expectation (Opus ≥ Sonnet ≥ Haiku) or surprised.
- Whether self-reported model IDs are consistent with the `model` parameter you passed (this is the strongest in-harness model-identity check available).

If multiple problems were run, end with a 1–2 sentence overall verdict.

---

## Default problems

If the user did not specify a problem type, run **both** of these. They are chosen because they (a) are graduate-level, (b) have definite right answers, (c) live in different fields, and (d) have empirically discriminated well between Opus, Sonnet, and Haiku.

### Problem A — Algebraic topology

> Compute π₂(S¹ ∨ S²) — the second homotopy group of the wedge of a circle and a 2-sphere — as a module over the group ring ℤ[π₁(S¹ ∨ S²)] = ℤ[t, t⁻¹]. Show:
> (a) what the universal cover looks like,
> (b) how covering-space theory + the Hurewicz theorem give π₂,
> (c) the final answer as a ℤ[t, t⁻¹]-module (free? torsion? rank?).

**Known answer:** π₂(S¹ ∨ S²) ≅ ℤ[t, t⁻¹], free of rank 1 as a ℤ[t, t⁻¹]-module. Universal cover is ℝ with one S² attached at every integer (one preimage per deck transformation); the deck shift acts on H₂ by sending [S²ₙ] ↦ [S²ₙ₊₁].

**Common failure mode** (observed in Haiku): claims the universal cover is just "ℝ ∨ S²" (one sphere, not infinitely many), then abandons the cover and applies Hurewicz directly to S¹ ∨ S² — which is invalid (X is not simply connected) — and concludes π₂ ≅ ℤ with trivial t-action. This is the precise opposite of the correct answer.

**Key insights checklist:**
- [ ] Universal cover has one S² *per integer* of ℝ (not just one).
- [ ] π_k(X̃) ≅ π_k(X) for k ≥ 2 (covering maps preserve higher homotopy).
- [ ] Hurewicz applies to X̃ (simply connected), NOT to X directly.
- [ ] Deck transformation t acts by shift on H₂, making the answer a free rank-1 ℤ[t, t⁻¹]-module.

### Problem B — Probability theory

> n people check coats and receive numbered tickets. On leaving, each person grabs a coat uniformly at random from the coats still on the rack (without replacement). Let Xₙ = number of people who get their own coat back.
> (a) Compute E[Xₙ] exactly.
> (b) Compute Var[Xₙ] exactly.
> (c) Prove that Xₙ converges in distribution to Poisson(1) as n → ∞.

**Known answer:**
- E[Xₙ] = 1 by linearity of indicator variables (P(person i gets own coat) = 1/n, summed gives 1).
- Var[Xₙ] = **1 exactly** (not just asymptotically). The covariance computation gives Cov(Iᵢ, Iⱼ) = 1/(n(n-1)) − 1/n² = 1/(n²(n-1)) for i≠j, and the variance terms ∑Var(Iᵢ) = (n−1)/n plus the n(n−1) covariance contributions = 1/n sum to exactly 1.
- Limiting distribution: factorial moments E[Xₙ(Xₙ−1)···(Xₙ−k+1)] = 1 for all k ≤ n, matching Poisson(1)'s factorial moments; this proves convergence in distribution by the method of moments.

**Common failure mode:** weaker models nail E[Xₙ] = 1 but for the variance either (i) get the covariance wrong by treating Iᵢ, Iⱼ as independent, (ii) compute the variance only asymptotically and miss the "exactly 1" punchline, or (iii) for part (c), hand-wave with "by the law of small numbers" instead of computing factorial moments or applying Bonferroni / inclusion-exclusion.

**Key insights checklist:**
- [ ] Use indicator variables and linearity for E[Xₙ].
- [ ] Recognize that P(Iᵢ = Iⱼ = 1) = 1/(n(n-1)), not 1/n².
- [ ] Show the exact cancellation that makes Var[Xₙ] = 1 for all n ≥ 2.
- [ ] Use factorial moments (or inclusion-exclusion on derangements) for the Poisson(1) limit.

---

## Tips and gotchas

- The `Agent` tool's `model` parameter accepts only `opus`, `sonnet`, or `haiku` (not date-stamped IDs). The self-reported ID from inside the subagent is the strongest available identity signal — there is no environment variable that exposes the model in use.
- Extended thinking is **not** triggered by the word "think" in prompts dispatched via `Agent`. The "think carefully" hint is content only — useful as a prompt cue, but it does not engage a reasoning budget.
- Keep the prompt **byte-identical** across models — even tiny phrasing differences confound the comparison.
- Dispatch foreground (default `run_in_background: false`) so you can see all three results in one turn. Background dispatch defeats the purpose of a head-to-head.
- If the user offers a problem that doesn't have a definite right answer (e.g., "write me a poem"), warn them — without a ground truth you can compare style but not correctness.

---

## When NOT to use this skill

- The user wants a single answer, not a head-to-head ("solve this for me" ≠ "have your models race to solve this").
- The user wants a model recommendation ("which model should I pick for X") — that's an opinion request, not a benchmark.
- The user is debugging one specific model's behavior — use the `Agent` tool directly with the model in question.
- The user is asking about agent architecture, multi-agent design, or HTTP user-agent strings — the word "agent" alone is not a trigger without a comparative intent.
