# Component Spec: OPS3 — Secrets (Trimmed)

> Status: Draft (trimmed scope)
> Workstream: OPS (Operations & Platform Engineering)
> Component ID: OPS3
> Depends on: External Secrets Operator (ESO), Reloader, LiteLLM, ADR 0020 (MCP-service secrets via ESO)

---

## 1. Purpose & Scope

OPS3 originally covered the full secret/key lifecycle: secret-at-rest
encryption / KMS choice, rotation cadence, and the ESO + reloader + LiteLLM
entry flow. After the scope cut, **only the secret-delivery flow survives** —
how secrets reach agent-os workloads via ESO, how startup gates on reloader,
and how LiteLLM credentials are entered and propagated.

**Out of scope / not covered here (for traceability):**

- **Secret-at-rest encryption** (envelope encryption of Kubernetes Secrets and
  the backing store) — baseline concern (OOS).
- **KMS choice / key management** — baseline concern (OOS).
- **Rotation cadence** (per-class intervals, overlap windows, scheduled
  rotation) — baseline concern (OOS).

These are platform/cluster responsibilities owned outside the agent-os boundary.

---

## 2. ESO Abstraction (Survived Scope)

- The **External Secrets Operator (ESO)** is the secret-delivery abstraction:
  agent-os workloads consume secrets as standard Kubernetes Secrets that ESO
  materializes from the external secret store.
- agent-os **configures** ESO `SecretStore` / `ExternalSecret` resources; it
  does **not** own the external store, the key material, or the KMS that
  protects it (those are baseline).

---

## 3. Reloader Startup Gate (Survived Scope)

- agent-os startup **gates on reloader presence**: if reloader is not present,
  startup does not proceed, because the secret-propagation chain depends on it.
- The propagation chain is:
  **external store → ESO updates the Kubernetes Secret → reloader restarts the
  consuming workloads** so they pick up the new secret value.

---

## 4. LiteLLM PushSecret Flow (Survived Scope)

- LiteLLM reads its model-provider credentials from a **Secret in its own
  namespace**.
- Operators enter credentials via the **LiteLLM GUI**; LiteLLM writes them to
  the Kubernetes Secret.
- An ESO **PushSecret** then propagates that Secret **out** to the external
  store, so the external store reflects what was entered.

**Authority rule (per-secret):** for each secret, the spec MUST declare whether
the **external store (pull)** or **LiteLLM/k8s (push)** is the authoritative
source — **never both**. A secret declared pull-authoritative is materialized by
ESO from the store; a secret declared push-authoritative originates in
LiteLLM/k8s and is pushed to the store via PushSecret.

---

## 5. Acceptance Criteria

- **AC-1:** A workload consuming a secret receives it as a Kubernetes Secret
  materialized by ESO from the configured `ExternalSecret`.
- **AC-2:** With reloader absent, agent-os startup does not proceed (the startup
  gate fails closed).
- **AC-3:** Updating the value in the external store causes ESO to update the
  Kubernetes Secret and reloader to restart the consuming workload.
- **AC-4:** A credential entered via the LiteLLM GUI is written to the LiteLLM
  namespace Secret and propagated to the external store via PushSecret.
- **AC-5:** Every secret declared in the spec has exactly one declared authority
  (pull from external store XOR push from LiteLLM/k8s) — never both.

---

## 6. References

- ADR 0020 (MCP-service / credential secrets delivered via ESO).
- DECISIONS-LOG: secret-at-rest encryption, KMS choice, and rotation cadence
  are baseline concerns (out-of-scope) and are not covered here.
