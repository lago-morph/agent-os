# Component Spec: OPS6 — RBAC Granularity (Trimmed)

> Status: Draft (trimmed scope)
> Workstream: OPS (Operations & Platform Engineering)
> Component ID: OPS6
> Depends on: native Kubernetes RBAC, Gatekeeper (OPA) admission, B3 / B16 (policy bundle)

---

## 1. Purpose & Scope

OPS6 originally covered Rego (OPA) human-factors and policy lifecycle broadly.
After the scope cut, **only the RBAC-granularity slice survives** (decision
#23). The policy-bundle ownership, signing, and staged-rollout concerns are
already decided and implemented elsewhere (B3 / B16) and are only
cross-referenced here.

**Out of scope / not covered here (for traceability):**

- **Anti-lockout / break-glass** for the policy plane — post-MVP (future
  version, decision #27). v1 relies on the policy simulators (A20) to catch a
  bad policy before deploy; if one slips through, an operator fixes it manually.
- **Policy bundle ownership, signing, and staged rollout** (decisions
  #24 / #25 / #26) — already decided and implemented in B3 / B16; OPS6 only
  cross-references them (see §3).

---

## 2. RBAC Granularity (Survived Scope — #23)

The platform MUST provide **distinct RBAC permissions at the proposed
granularity**, so that operationally-distinct actions are separately grantable.
Concretely:

- **Budget permissions** are distinguished into separate grantable actions:
  - set an **agent-type default budget**,
  - apply a **per-instance budget override**,
  - act as a **budget-maintainer**.
- **Policy permissions** are distinguished into:
  - **policy-change** (modify policy),
  - **policy-maintain** (operate/maintain without changing).

These are distinct so an operator can be granted one without implying the
others.

**Field-level rules native RBAC cannot express** (e.g. distinctions that depend
on a field value within a resource rather than on the verb/resource pair) MUST
be enforced via **Gatekeeper (OPA) admission**, which is already deployed.

---

## 3. Policy Bundle (Cross-Reference — Decided Elsewhere)

The following are **already decided and implemented in B3 / B16** and are listed
here only for traceability; OPS6 does not re-specify them:

- **Ownership (#24):** the platform security team owns the global / cross-tenant
  OPA bundle; tenants propose changes by pull request, approved by a security
  reviewer before merge.
- **Signing (#25):** policy bundles are cryptographically signed and verified at
  load time; the engine refuses an unsigned or tampered bundle.
- **Staged rollout (#26):** a new/changed bundle runs in audit mode first, then
  flips to enforce after review; the Kargo promotion pipeline gates the flip.

See the B3 / B16 specs for the authoritative treatment.

---

## 4. Acceptance Criteria

- **AC-1:** Distinct RBAC permissions exist for: agent-type default budget,
  per-instance budget override, and budget-maintainer — each grantable
  independently of the others.
- **AC-2:** Distinct RBAC permissions exist for policy-change vs
  policy-maintain — each grantable independently.
- **AC-3:** A field-level rule that native RBAC cannot express is enforced via
  Gatekeeper (OPA) admission (e.g. a request that native RBAC would allow but
  the field-level policy forbids is rejected at admission).

---

## 5. References

- DECISIONS-LOG decision #23 (RBAC granularity — survived).
- DECISIONS-LOG decisions #24 / #25 / #26 (policy bundle ownership / signing /
  staged rollout — decided and implemented in B3 / B16; cross-referenced only).
- DECISIONS-LOG decision #27 (anti-lockout / break-glass — post-MVP, future
  version; not covered here).
- B3 / B16 (policy bundle framework and content).
