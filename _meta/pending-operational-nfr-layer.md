# PENDING — Phase: Operational / NFR Architecture Layer

> Requested by user mid-run (2026-05-30). Runs AFTER the 119-piece spec/plan work,
> verification, consistency sweep, and PR graph are complete. Same cooperative +
> adversarial treatment; red agents push HARD (high blast radius).

## Why
The source architecture (`architecture-overview.md`) specifies functional structure
well but treats operational / non-functional architecture thinly — most of it is
deferred into Workstream F (F1–F6) and `future-enhancements.md`. This is a whole
missing **layer**, not a set of components. It must be specified as cross-cutting
architecture that constrains and is realized by many existing pieces, plus the ADRs
needed where decisions are genuinely open.

## Pieces to author (proposed IDs `OPS1..OPS6`; spec + plan each; new dir `specs/operational/`, `plans/operational/`)
- **OPS1 — Scale & cost architecture.** Capacity model, throughput/SLO targets per
  tier (gateway RPS, sandbox cold-start, broker backlog), autoscaling strategy,
  cost model + per-tenant cost attribution, right-sizing. Realized by A1/A4/A6/F5;
  relates `future-enhancements.md` §3 (SLO mgmt).
- **OPS2 — Disaster recovery architecture.** Beyond F2's *testing*: the DR design —
  RTO/RPO per data class, backup topology (Postgres/OpenSearch/object store/secrets),
  failover model, restore ordering, cross-AZ/region stance. Relates ADR 0034, F2,
  `future-enhancements.md` §1.
- **OPS3 — Key & secret management architecture.** Secret/key lifecycle, rotation,
  KMS/envelope encryption, ESO topology, virtual-key + credential handling across
  system/agent/tenant/user modes, IRSA/Workload Identity trust roots. Relates A18,
  B1, V6-11, ADR 0028/0029, F4.
- **OPS4 — Multi-tenant compute isolation.** Node pools / taints, sandbox isolation
  depth (gVisor/Kata), noisy-neighbor controls, resource quotas per tenant, network
  isolation, escape-blast-radius. Relates A6, ADR 0016, B20, `backlog` §1.2 (quotas).
- **OPS5 — Day-2 operations architecture.** Upgrade/patch strategy (platform + CRDs +
  vendor charts), CRD version migration ops (ADR 0030), capacity ops, incident
  response, the Crossplane `Operation` type question (backlog §2.20 / §3.6), Kargo
  promotion ops. Relates A23, B9, F6.
- **OPS6 — Rego / policy human-factors & lifecycle.** Who authors Rego, authoring
  ergonomics, policy testing/CI, review + approval, the policy simulator's role
  (A20), guardrails against policy mistakes, RBAC-floor conformance, bundle
  versioning/rollout. Relates A7, B3, B16, A20, ADR 0018.

## Deliverable shape
- Each OPS piece: SPEC = cross-cutting NFR architecture + invariants + acceptance
  (SLOs/RTO/RPO/rotation intervals as testable targets); PLAN = realization &
  verification map across existing component piece IDs + the gaps that need new work.
- Surface **proposed ADRs** where a decision is genuinely open (do NOT silently
  decide) — list titles for the user, author only if asked.
- Cross-link back into the relevant component specs (note where an existing spec
  must absorb an NFR obligation).
- Produce its own stacked PRs in the graph (one per OPS piece).
