# Open Decisions — what's left to decide (plain language)

The decisions already made are in `DECISIONS-LOG.md`. This document covers only what is **still open**, in plain English, with enough detail to decide. For each: **what it is · why it matters · the options · my recommendation · what I need from you.**

> **STATUS — updated 2026-05-31:** Five of the six below are now **DECIDED** (recorded in `DECISIONS-LOG.md`): #1 policy bundle, #2 memory store (design-time abstraction owned by the memory adapter), #3 quota schema (native CPU/memory, extensible typed list, presence required / value may be `unlimited`), #5 `system` & `system-mediated` meanings, #6 egress baseline. **Only #4 (Crossplane v1→v2 pass) remains** — its *approach* is decided (supersede ADR-0041 → new ADR → move old to `adr/superseded/`, then sweep); it needs the **go to launch**. Sections below are kept as the rationale of record.

---

## 1. The policy bundle — who controls the rules, and how they ship

**What it is.** Every agent action is checked against written rules (policies) run by the policy engine (OPA). Those rules are packaged into a "bundle" the engine loads. Three questions hang off it.

**Why it matters.** The bundle is both a security control and a single point of failure: a bad or tampered bundle could block every agent, or quietly allow everything.

**The three questions + my recommendation:**
- **Who owns the global (cross-tenant) bundle?** → *Recommend:* the platform security team owns it; tenants propose changes by pull request, approved by a security reviewer before merge.
- **Should bundles be cryptographically signed and verified at load time?** → *Recommend:* yes — without it, anyone who compromises the build pipeline could push a malicious rule set. Cheap security baseline.
- **Should a new bundle run in "watch-only" mode before it can block?** → *Recommend:* yes — a new/changed bundle first runs in audit mode (logs what it *would* block), and flips to enforcing only after review; the promotion pipeline (Kargo) gates that flip, like it gates code.

*(The related "bad policy locks everyone out" question is already decided: for v1 we rely on the policy simulators to catch it before deploy, and an operator fixes it manually if it slips through. Dedicated break-glass is post-MVP.)*

**What I need:** a yes/adjust on each of the three.

---

## 2. The memory store's connection details (D-04)

**What it is.** When the infrastructure layer builds an agent's memory store, it hands the memory adapter a small bundle of connection details (a "connection secret") so the adapter can connect. The exact contents were never written down.

**Why it matters.** The adapter is already coded to expect specific details; if the infrastructure layer doesn't promise them in a defined shape, the two won't line up at build time.

**My recommendation.** Define a standard set: host, port, scheme, username, password/token, the index-or-namespace name, and optionally a CA certificate for TLS. (The backing store is Letta on top of OpenSearch, so these are what a client needs.)

**What I need:** confirm that field set, or hand it to the infra-layer + adapter owners to finalize.

---

## 3. The tenant quota fields (needed to land "quota required")

**What it is.** You decided quota must be a required field when a tenant is onboarded, enforced with native Kubernetes limits. We still need the actual list of quota fields.

**Why it matters.** "Quota is required" can't be built until we know *what* quantities are capped.

**My recommendation.** A set like: max agents, max sandboxes, CPU ceiling, memory ceiling, warm-pool cap, virtual-key count, cost budget, request rate limit. Enforcement splits naturally: CPU/memory/pod-style limits → native Kubernetes `ResourceQuota`/`LimitRange`; cost budget + rate limit → the LiteLLM gateway (via the kopf operator).

**What I need:** confirm/trim the field list and the enforcement split.

---

## 4. Crossplane v1 → v2 conformance pass (the big one)

**What it is.** The specs were written using Crossplane **v1** concepts ("claims," the `X`-prefixed composite/claim split) — ~270 places, and it's baked into a foundational decision record (ADR-0041). The platform is Crossplane **v2**, which removed all of that.

**Why it matters.** Left as-is, the specs describe a resource model the platform won't use; teams building "to spec" would build the wrong thing, and a foundational ADR enshrines the wrong convention.

**My recommendation.** A coordinated pass: first **supersede ADR-0041** with a new "Crossplane v2 resource model" decision record (cleaner than editing in place), then sweep the corpus to re-express terminology *and* the affected logic (admission rules, naming, versioning) in v2 terms. This is **not** a find-and-replace — some passages describe genuine v1 behavior that needs rethinking. I'd run it as parallel sub-agents, each owning a slice, with a verification pass.

**What I need:** your go-ahead to start, and a yes on "supersede ADR-0041 with a new ADR" (vs editing it in place).

---

## 5. What "system-mediated" means now (auth modes)

**What it is.** Earlier we set the two credential modes to `system` and `system-mediated` and retired the user-supplied one. But once we established there are *no end-users*, `system-mediated` lost its original meaning ("act on a user's behalf").

**Why it matters.** A mode with no clear definition is untestable and confusing.

**The options:**
- **(a) Redefine it:** `system` = a static credential read from a namespace secret; `system-mediated` = the platform brokers a *dynamic/federated* credential (cloud workload identity, token exchange) on the agent's behalf. Both keep the "agents never hold credentials" rule.
- **(b) Simplify for v1:** implement only `system` (static namespace secret), and reserve `system-mediated` as a documented seam to add later if federation is ever needed.

**My recommendation:** (b) for v1 — less to build now, nothing lost, the door stays open.

**What I need:** pick (a) or (b).

---

## 6. The default sandbox egress baseline

**What it is.** Every agent sandbox is blocked from the network by default and allowed out only to an approved list. We agreed a starting baseline — and that it is an **extensible baseline, not a frozen ceiling**. Decided so far: the LiteLLM gateway, the Envoy egress proxy, the audit endpoint.

**Why it matters.** Miss something an agent legitimately needs and agents break on day one; the baseline should cover the obvious in-cluster platform services.

**My recommendation — add these to the baseline:** cluster DNS (agents can't resolve names without it), the memory backend, the telemetry/trace collector, and the event broker.

**What I need:** confirm or prune those four. (Anything beyond the baseline is added as discovered — explicitly not a closed list.)

---

### How to use this
Answer in any order. **#4 (Crossplane v2)** is the largest and unblocks the most real cleanup, so it's the highest-leverage one to settle first. Everything you decide gets recorded in `DECISIONS-LOG.md`.
