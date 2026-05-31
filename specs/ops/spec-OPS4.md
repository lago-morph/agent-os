# Component Spec: OPS4 — Isolation (Trimmed)

> Status: Draft (trimmed scope)
> Workstream: OPS (Operations & Platform Engineering)
> Component ID: OPS4
> Depends on: agent-sandbox (A6), Envoy egress proxy, D-08 (sandbox egress baseline)

---

## 1. Purpose & Scope

OPS4 originally covered multi-tenant compute isolation broadly: container
runtime class, soft vs hard multi-tenancy, general tenant network isolation, and
agent-sandbox isolation. After the scope cut, **only the agent-sandbox isolation
slice survives.**

**Out of scope / not covered here (for traceability):**

- **Container runtime class** (gVisor / Kata / runc selection policy) — baseline
  / cluster concern, out-of-scope (decision #6).
- **Soft vs hard multi-tenancy / node-level isolation** (node-sharing,
  node-pool entitlement) — baseline / cluster concern, out-of-scope
  (decision #7).
- **General tenant network isolation** (cross-tenant pod-to-pod policy across the
  cluster) — baseline / cluster concern, out-of-scope (decision #8).

These are platform/cluster responsibilities owned outside the agent-os boundary.

---

## 2. Agent-Sandbox Isolation (Survived Scope)

Per **D-08**, the surviving isolation concern is the network posture of the
**agent sandbox** itself.

### 2.1 Default-deny baseline (L3/L4)

- Sandbox pods run under a Kubernetes **`NetworkPolicy` default-deny** baseline
  at L3/L4: by default a sandbox pod may not open arbitrary connections.

### 2.2 L7 external egress via Envoy

- External egress from a sandbox is mediated by **Envoy** at L7; the sandbox does
  not reach external FQDNs except through the Envoy egress chokepoint.

### 2.3 Sandbox egress floor (minimum baseline, extensible)

The default-deny baseline permits a **minimum egress floor** — the set of
destinations a sandbox needs to function. This floor is a baseline and is
**explicitly extensible** (per D-08). The floor comprises:

- the **LiteLLM gateway** (model access),
- **Envoy** (L7 external egress),
- the **audit endpoint**,
- **CoreDNS** (in-cluster DNS),
- the **memory backend**,
- the **OTel collector** (telemetry),
- the **event broker**.

Destinations beyond this floor are added explicitly; the floor is the minimum,
not the maximum.

---

## 3. Quota Field (Survived — Cross-Reference)

The `quotas` field on `AgentEnvironment` is the quota carrier for tenant
resource ceilings. It is **owned and specified by OPS1** (trimmed quota slice);
OPS4 does not re-specify it. See `spec-OPS1.md`.

---

## 4. Acceptance Criteria

- **AC-1:** A sandbox pod is subject to a default-deny `NetworkPolicy` at L3/L4;
  a connection to a destination not on the egress floor (and not explicitly
  added) is denied.
- **AC-2:** External egress from a sandbox traverses Envoy at L7; a direct
  external connection bypassing Envoy is denied.
- **AC-3:** Each destination in the sandbox egress floor (LiteLLM gateway,
  Envoy, audit endpoint, CoreDNS, memory backend, OTel collector, event broker)
  is reachable from a sandbox under the baseline policy.
- **AC-4:** Adding an explicit extra egress destination to a sandbox is honored
  without removing the default-deny posture for everything else.

---

## 5. References

- D-08 (agent-sandbox egress baseline — surviving isolation slice).
- `spec-OPS1.md` (the `AgentEnvironment.quotas` field — survived, cross-ref).
- DECISIONS-LOG: container runtime class (#6), node-level isolation (#7), and
  general tenant network isolation (#8) are baseline/cluster concerns
  (out-of-scope) and are not covered here.
