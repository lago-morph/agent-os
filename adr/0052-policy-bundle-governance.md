# ADR 0052: Policy-bundle governance

- **Status**: Accepted
- **Date**: 2026-05-31

## Context

The platform enforces authorization and admission decisions with OPA/Gatekeeper
(ADR 0002). The individual Rego policies do not run loose; they are assembled into a
**bundle** that the policy engine loads at runtime. A bundle is shared infrastructure:
the global / cross-tenant bundle decides what every tenant's workloads may and may not
do, so a single bad rule can deny legitimate traffic platform-wide or, worse, silently
permit something it should block.

That makes three questions load-bearing and impossible to leave to detailed design:
**who is allowed to change the bundle**, **how the engine knows the bundle it loaded is
the one that was approved**, and **how a change reaches enforcement without breaking
production on the first request**. A related but separable question — how an operator
recovers if a deployed policy locks the platform out of its own control plane — also
needs an explicit answer, even if that answer is "later".

These were tracked as open review items #24 (ownership), #25 (signing), #26 (staged
rollout), and #27 (anti-lockout / break-glass). #24/#25/#26 are now decided; #27 is
explicitly deferred. This ADR records those decisions.

## Decision

OPA policies are assembled into a signed bundle that the engine loads, governed as
follows.

- **Ownership (#24).** The **platform security team owns the global / cross-tenant OPA
  bundle.** Tenants do not edit it directly. A tenant proposes a change by **pull
  request**, and a **security reviewer must approve it before merge.** The bundle is
  therefore governed like any other reviewed, version-controlled artifact, with a single
  accountable owner rather than ad hoc per-tenant edits.

- **Signing (#25).** Policy bundles are **cryptographically signed**, and the engine
  **verifies the signature at load time.** The engine **refuses to load an unsigned or
  tampered bundle.** A bundle that does not verify is never enforced; the engine fails
  closed on the integrity check rather than running unverified rules.

- **Staged rollout (#26).** A new or changed bundle does not go straight to
  enforcement. It first runs in **audit mode** — it evaluates every request and **logs
  what it would block**, without actually blocking anything — so the real impact of the
  change is observable before it bites. It flips from audit to **enforce** only **after
  review** of that audit signal. The **Kargo promotion pipeline (ADR 0040) gates the
  flip**, exactly as it gates application code through environments: the audit-to-enforce
  transition is a promotion that Kargo controls, not a manual toggle on a live cluster.

- **Anti-lockout / break-glass is deferred to a future version (#27).** v1 does **not**
  ship a dedicated break-glass or anti-lockout mechanism. Instead, v1 relies on the
  **policy simulators (ADR 0038, A20)** to catch a bad policy **before** it is deployed:
  a proposed bundle is simulated against representative requests during review, which is
  where a lockout-inducing rule is expected to be caught. If a bad policy nonetheless
  slips through, the v1 recovery path is **manual** — an operator intervenes and fixes
  the policy directly. A purpose-built break-glass path is **explicitly out of scope for
  the MVP** and is post-MVP work.

## Consequences

- The global bundle has one accountable owner (the platform security team) and a single
  reviewed change path (PR + security approval), so there is no route for a tenant to
  alter cross-tenant policy unreviewed.
- Load-time signature verification means the engine will only ever enforce a bundle that
  was approved and signed; an unsigned or modified bundle is rejected rather than run.
  Key management for the signing identity is a detailed-design concern this ADR does not
  pin down.
- Audit mode plus Kargo-gated promotion gives every policy change an observable
  dry-run and a controlled, reviewable path to enforcement, at the cost of an extra
  promotion step before a change takes effect — the same trade-off already accepted for
  code.
- The deferral of #27 is a deliberate, named scope boundary, not an oversight: in v1 the
  safety net against a lockout-inducing policy is pre-deploy simulation plus manual
  operator recovery. Until dedicated anti-lockout machinery is built post-MVP, the
  residual risk is that a policy which evades simulation and reaches enforce mode could
  require hands-on operator remediation. This trade-off is accepted for the MVP.
- Bundle governance applies to the policy library framework (B3) and to the policy
  content that populates it (B16); this ADR governs how that content is owned, signed,
  and rolled out, not what individual rules say.

## References

- [ADR 0002](./0002-opa-gatekeeper-policy-engine.md) (OPA/Gatekeeper as the policy engine that loads the bundle)
- [ADR 0038](./0038-policy-simulators.md) (policy simulators, A20 — the v1 pre-deploy safety net relied on in lieu of break-glass)
- [ADR 0040](./0040-kargo-promotion-fabric.md) (Kargo promotion pipeline that gates the audit-to-enforce flip)
- `_meta/reviews/DECISIONS-LOG.md` (decisions #24/#25/#26; #27 deferred to a future version)
- B3 (policy library framework) / B16 (policy library content) — the bundle this ADR governs
