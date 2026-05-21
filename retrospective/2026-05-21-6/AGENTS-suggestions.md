# AGENTS.md suggestions — 2026-05-21-6

These are proposed additions to the project's agents file (typically `AGENTS.md` at the repo root). Each section contains:

1. **Proposed addition** — the exact text to paste.
2. **Why this earns its place in your agents file** — the argument for doing it, grounded in something that happened (or nearly happened).

Decide each on its own merits. Skip ones that don't apply to your operating posture; copy-paste the ones that do.

The repo has no `AGENTS.md` at the time this retrospective is being written. Adopting any of these implicitly creates one.

---

## Suggestion 1: The `Agent` tool's `model` parameter is real and routable

### Proposed addition

> **Subagent model overrides.** The `Agent` tool accepts an optional `model` parameter taking `opus`, `sonnet`, or `haiku` (not date-stamped IDs). Omitting it inherits from the agent definition's frontmatter or, failing that, from the parent. Use the override when (a) a task is cheap and would waste Opus capacity, (b) you want a parallel head-to-head across models, or (c) you're matching a specific capability tier to a specific task.
>
> *Grounded in: subagent model exploration session, PR #6.*

### Why this earns its place in your agents file

Agents in this codebase reach for `Agent` constantly but typically don't think about the model dimension — the tool's `model` parameter is buried in the JSONSchema description and easy to miss. In this session it took an explicit user question ("can you spawn a subagent with a different model?") to surface that it works. Without an agents-file rule, the next agent will likely default to inheriting Opus for tasks that don't need it — wasting capacity on Haiku-class work and missing the opportunity to use cross-model comparison as a debugging tool. Marginal cost of adopting the rule: one paragraph. Marginal cost of not adopting it: every future agent re-discovers the parameter from scratch.

---

## Suggestion 2: Self-report is the only in-harness model-identity check

### Proposed addition

> **Verifying which model a subagent is actually running.** Inside the harness, there is no environment variable that exposes the model in use — `ANTHROPIC_MODEL` is empty and no `CLAUDE_CODE_*` var carries the model ID. The strongest available identity check is to ask the subagent to (a) self-report its model ID from its system prompt and (b) attempt a capability test whose outcome is model-discriminating. Treat both as indirect signals — strong together, weak in isolation. If you need ground-truth model identity, it lives in Anthropic-side billing telemetry, not the container.
>
> *Grounded in: model-identity probe in subagent-model-exploration session, PR #6.*

### Why this earns its place in your agents file

A future agent investigating "is the model parameter actually being respected?" will, in the absence of this rule, spend tool calls grepping env vars that don't exist — exactly what happened in this session before the limit was characterized. Recording the finding once saves the rediscovery cost forever. The rule also names the right alternative (capability test as indirect signal), so the next agent has a path forward rather than just a dead end.

---

## Suggestion 3: Parallel subagent dispatch requires a single assistant message

### Proposed addition

> **Parallel dispatch is one message, not one-message-per-agent.** When dispatching multiple subagents intended to run concurrently, **all** `Agent` tool calls must be emitted in a single assistant message. Splitting them across messages serializes the dispatch and silently destroys the parallelism. This applies to fanouts, benchmarks, and any A/B comparison where wall-clock matters.
>
> *Grounded in: 3-way benchmark dispatch in subagent-model-exploration session, PR #6.*

### Why this earns its place in your agents file

This is the kind of rule that's obvious in retrospect but easy to violate by accident — an agent thinking "I'll dispatch one, then another, then another" produces serial execution and a corrupted latency comparison without any error message. The cost of the rule is one sentence per dispatch decision. The cost of violating it is a benchmark that looks completed but whose timings are meaningless and whose total wall-clock is 3× what it should have been.

---

## Suggestion 4: Identical prompts are mandatory when comparing models

### Proposed addition

> **Byte-identical prompts when comparing models.** When dispatching the same task to multiple model variants, the prompt strings must be byte-identical. Even small phrasing differences ("solve this" vs "compute this", "show all reasoning" vs "show your work") confound the comparison — you cannot tell whether the differing outputs are model-driven or prompt-driven. Construct the prompt once and pass the same string to every dispatch.
>
> *Grounded in: 3-way benchmark dispatch in subagent-model-exploration session, PR #6.*

### Why this earns its place in your agents file

The natural impulse during head-to-head dispatch is to "tweak the prompt for each model" — to soften the problem for Haiku, to add detail for Opus. Each of those tweaks turns the comparison into noise. This rule preempts the impulse with a one-line discipline. The shipped `benchmark-subagent` skill bakes this in, but the rule is broader than the skill: any time an agent is comparing model behavior, it applies.

---

## Suggestion 5: "Think hard" in a prompt does NOT engage extended thinking via `Agent`

### Proposed addition

> **The word "think" is content, not a harness toggle.** Putting "think hard", "think carefully", or "use extended thinking" in a prompt dispatched via the `Agent` tool does **not** allocate an extended-thinking budget on the subagent. It is a content cue only — the subagent reads the words and may try to reason more carefully, but no separate reasoning channel is engaged and no thinking-budget parameter is exposed at this dispatch path. If you need real extended thinking, route through whatever mechanism your harness exposes for that (often a parent-level configuration, not a per-dispatch flag).
>
> *Grounded in: 3-way benchmark in subagent-model-exploration session — all three subagents reported "extended thinking: no" despite the prompt containing "Think hard before answering" — PR #6.*

### Why this earns its place in your agents file

Without this rule, every future agent designing a hard-problem dispatch will assume the keyword does something. It doesn't — the subagents in this session reported, when asked, that no thinking budget was allocated despite the prompt explicitly including the trigger phrase. The cost of being wrong about this is wasted prompt tokens and false confidence in the dispatch design. The cost of recording it is one sentence.

---
