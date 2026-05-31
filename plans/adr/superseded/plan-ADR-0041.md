# PLAN ADR-0041 — Substrate abstraction via Crossplane Compositions `[SUPERSEDED]`

> ⚠️ **STATUS: SUPERSEDED by ADR-0044.** The platform uses Crossplane v2. Do not implement
> from this document. See `plan-ADR-0044.md`.

> spec: SPEC-ADR-0041 · kind: ADR · tier: T0
> wave: authoring-parallel (auth) · estimate: S
> upstream-pieces: [A7, A11] · downstream-pieces: [B4, A18, A21, A23, B19, B11]

(Front-matter note: `_meta/piece-index.csv` carries `wave=auth`, tier `T0`, estimate `S`, and
empty upstream/downstream for ADR rows. The edge set above is taken from the spec front-matter —
the enforcer/consumer set this plan maps — since the ADR is a constraint piece with no build wave.)

## 1. Implementation Strategy

This is an **ADR plan** — an **enforcement & verification map** for a settled decision, not a build
sequence. ADR-0041 fixes the substrate-abstraction pattern: each substrate-asymmetric primitive is
**one XRD + one Composition per substrate** (kind + AWS for v1.0); both Compositions write the
**same connection-secret shape** (`host`, `port`, `user`, `password`, `dbname` or equivalent); XR
status is **substrate-agnostic** (`ready`, `endpoint`, `version`); every cluster carries
`platform.io/environment=kind|aws` and an **OPA Gatekeeper admission policy rejects any claim with
no matching Composition**; the v1.0 substrate XRDs are `XPostgres`, `XSearchIndex`, `XObjectStore`,
`XMongoDocStore`, with `XAuditLog` / `XAgentDatabase` / `XGrafanaDashboard` composing them; the
claim drops the `X` prefix; XRD versioning (ADR 0030) applies to both Compositions; capability
parity is **not** promised. The connection-secret-shape and status-field discipline are **tested
architectural invariants, not conventions**. The strategy names the enforcing component pieces and
maps each AC to the piece+layer that proves conformance. The **primary enforcer is B4** (builds
every XRD's kind+AWS Compositions and the conformance tests). Supporting enforcers per the task
brief: **A18** (`XAuditLog` consumes `XPostgres`+`XObjectStore`), **A21** (`TenantOnboarding`
provisioning over the primitives), **A23** (Kargo promotes the uniform claim shapes — cross-cuts
ADR-0040), and **B11** (memory backend adapter over `MemoryStore` composing substrate primitives).
A7/Gatekeeper enforces the environment-label admission guardrail; A11 is the `XSearchIndex` kind
backend.

## 2. Ordered Task List

Ordered as the conformance-verification critical path (B4 is foundation-band W1; consumers land
later).

- `TASK-01: Confirm A7/Gatekeeper carries the environment-label admission policy (reject claim with no matching Composition for `platform.io/environment`) — produces: enforcement binding for REQ-04/AC-04 — depends-on: []`
- `TASK-02: Map B4 wrap-rule enforcement — exactly one XRD + one Composition per substrate per primitive; no parallel manifest sets — produces: enforcement binding for REQ-01/AC-01 — depends-on: []`
- `TASK-03: Map B4 connection-secret conformance test (both Compositions write the canonical five-field shape on kind and AWS) — produces: enforcement binding for REQ-02/AC-02 — depends-on: [TASK-02]`
- `TASK-04: Map B4 substrate-agnostic status conformance test (only `ready`/`endpoint`/`version`; no RDS ARN / Service path leakage) — produces: enforcement binding for REQ-03/AC-03 — depends-on: [TASK-02]`
- `TASK-05: Map the v1.0 XRD set + composition hierarchy (`XPostgres`/`XSearchIndex`/`XObjectStore`/`XMongoDocStore` primitives; `XAuditLog`/`XAgentDatabase`/`XGrafanaDashboard` composers) and X-prefix↔claim naming — produces: enforcement binding for REQ-05/REQ-06 (AC-05/AC-06) — depends-on: [TASK-02]`
- `TASK-06: Map XRD versioning enforcement (claim-shape change → conversion webhook + deprecation window on both Compositions) per ADR 0030 — produces: enforcement binding for REQ-07/AC-07 — depends-on: [TASK-05]`
- `TASK-07: Map capability-parity-caveat enforcement (claim shape identical; kind `XObjectStore` "no archive" documented on the XRD) and the NOT-wrapped exceptions (Knative sources, bootstrap, cloud stack, sizing/region) — produces: enforcement binding for REQ-08/REQ-09 (AC-08/AC-09) — depends-on: [TASK-05]`
- `TASK-08: Map consumer conformance (A18 `XAuditLog`, A21 `TenantOnboarding`, A23 Kargo claim promotion, B11 memory adapter) against the claim contract — produces: downstream conformance surface — depends-on: [TASK-03, TASK-04]`
- `TASK-09: Assemble the conformance matrix (REQ→AC→enforcer-piece→test-layer) — produces: §5 conformance map — depends-on: [TASK-03..TASK-08]`

## 3. Dependency Map

### 3.1 Upstream pieces that must ship first (HARD) — id + what is consumed
- **A7** (OPA/Gatekeeper) — the admission engine for the environment-label guardrail; consumed for REQ-04. (Policy *content* is B16 over B3.)
- **A11** (OpenSearch) — the `XSearchIndex` kind-side backend; consumed for REQ-05 (`XSearchIndex`).

### 3.2 Downstream pieces blocked on this — ids
- **B4** (Crossplane v2 Compositions) — primary realizer; owns every Composition + the conformance tests.
- **A18** (audit) — `XAuditLog` composes `XPostgres` + `XObjectStore`.
- **A21** (tenant onboarding) — `TenantOnboarding` provisions over the primitives.
- **A23** (Kargo) — promotes the uniform claim shapes (cross-cuts ADR-0040).
- **B19** (approval system) — approval-related provisioning over the abstraction.
- **B11** (memory backend adapter) — operates over `MemoryStore` composed from substrate primitives.

### 3.3 Continuous (non-blocking) inputs
- **B14** (agent-platform test framework) — the harness the conformance/dual-mode tests run in.
- **B22** (security threat-model design) — admission-guardrail posture standards (fan-out edge).
- **B16/B3** (OPA policy content/framework) — the Rego for the environment-label admission policy (content owner; non-wave-setting for the ADR).

## 4. Parallelizable Subtasks

- TASK-01 (admission guardrail) and TASK-02 (wrap rule) are independent and run concurrently.
- After TASK-02, **TASK-03** (secret shape), **TASK-04** (status fields), and **TASK-05** (XRD set/naming) fan out concurrently.
- TASK-06 and TASK-07 both depend only on TASK-05 and run concurrently.
- TASK-08 joins TASK-03+TASK-04; TASK-09 is the final join.

## 5. Test Strategy

The decision's discipline is a **tested invariant** (spec §7 Testability), enforced by **B4's**
three-layer tests; the ADR ships no code. Map of AC → layer → enforcing piece:

| AC | Requirement | Layer | Enforcing piece / fixture |
|---|---|---|---|
| AC-01 | one XRD + one Composition/substrate; no parallel manifest set | Chainsaw (XRD/Composition inventory) | B4 |
| AC-02 | both Compositions write identical connection-secret shape (kind + AWS) | PyTest (secret-shape conformance) + Chainsaw | B4 |
| AC-03 | XR status exposes only `ready`/`endpoint`/`version`; no substrate-specific leak | Chainsaw (status assertion) | B4 |
| AC-04 | claim with no matching Composition rejected at admission | Chainsaw (Gatekeeper admission) + PyTest/conftest (Rego) | A7 + B16 |
| AC-05 | primitives exist; `XAuditLog`/`XAgentDatabase`/`XGrafanaDashboard` compose them | Chainsaw (composition graph) | B4 (+ A18 for `XAuditLog`) |
| AC-06 | claim `Postgres` ↔ XR `XPostgres` resolve to same XRD | Chainsaw (claim/XR resolution) | B4 |
| AC-07 | claim-shape change triggers conversion-webhook + deprecation window | Chainsaw (conversion) + PyTest (versioning) | B4 |
| AC-08 | kind `XObjectStore` "no archive" while claim shape identical; documented | Chainsaw (dual-mode) + PyTest (doc-presence) | B4 (dual-mode CI per ADR 0033) |
| AC-09 | a documented exception (e.g. Knative SQS source) shown NOT wrapped | PyTest (exception assertion) | B4 / A23-Knative |

Fixtures/fakes for not-yet-landed upstreams: a **dual-mode (kind + AWS-emulated) cluster fixture**
so secret-shape and status parity (AC-02/03/08) are exercised before real AWS; a **labelled-cluster
fixture** (`platform.io/environment` set/unset) to drive the admission guardrail (AC-04) before
B16's Rego content lands; a **conversion-webhook stub** for AC-07.

## 6. PR / Branch Mapping

### 6.1 Stack position — base branch = `wave/auth` (ADRs are authoring-parallel context)
ADR-0041 is a constraint piece (`wave=auth`), no build wave. Its spec+plan land on the
authoring-parallel base so B4 and all consumer specs bind to it.

### 6.2 PR — `piece/ADR-0041-substrate-abstraction` → base `wave/auth`; carries spec + plan for this piece
Single PR with `spec-ADR-0041.md` + `plan-ADR-0041.md`. No code (enforcement lives in B4 + consumer PRs).

### 6.3 Merge order — within-wave siblings independent; wave rolls up to main
Independent of other `auth` siblings. Real-build enforcement threads through the waves: A7/A11
(W0) → **B4 (W1, foundation band — owns the Compositions + conformance tests)** → consumers
A18 (W1), A21 (W3), A23 (W4), B11 (W1), B19 (W3).

## 7. Effort Estimate

Per-task: TASK-01 S, TASK-02 S, TASK-03 S, TASK-04 S, TASK-05 M (XRD-set + hierarchy mapping),
TASK-06 S, TASK-07 S, TASK-08 S, TASK-09 S. **Rollup: S** (matches piece-index; the heavy lift is
B4, an XL component). **Critical path within the piece:** TASK-02 → TASK-05 → TASK-06/07 →
TASK-09.

## 8. Rollback / Reversibility

Backing out the spec+plan PR removes the constraint document; it touches no running system. The
decision is foundational: if withdrawn, B4 loses its uniform-claim contract, and every consumer
(A18 `XAuditLog`, A21 `TenantOnboarding`, A23 Kargo cross-substrate promotion, B11 memory adapter)
loses substrate transparency — they would fall back to substrate-aware manifests, which the ADR
explicitly rejects. Because the connection-secret shape and status fields are versioned invariants,
any future change is a conversion-webhook-mediated migration (REQ-07), not a hard break. Reverting
the ADR document itself implies no data migration; reverting B4 Compositions is the real
operational rollback and is out of scope for this ADR plan (lives in the B4 plan).
