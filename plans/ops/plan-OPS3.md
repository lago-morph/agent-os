# Implementation Plan: OPS3 — Secrets (Trimmed)

> Status: Draft (trimmed scope)
> Workstream: OPS (Operations & Platform Engineering)
> Component ID: OPS3

---

## Scope

Only the secret-delivery flow survives: ESO materialization, the reloader
startup gate, and the LiteLLM PushSecret entry flow. Secret-at-rest encryption,
KMS choice, and rotation cadence are baseline concerns (out-of-scope) and are
not implemented here.

## Work Items

1. **ESO wiring.** Configure `SecretStore` / `ExternalSecret` resources so
   agent-os workloads consume secrets as ESO-materialized Kubernetes Secrets.
   Do not own the external store, key material, or KMS.
2. **Reloader startup gate.** Make agent-os startup gate on reloader presence;
   wire the external-store → ESO → k8s-Secret → reloader-restart chain.
3. **LiteLLM PushSecret flow.** LiteLLM reads credentials from a Secret in its
   own namespace; operator-entered (GUI) credentials are written to that Secret
   and pushed to the external store via PushSecret.
4. **Per-secret authority declaration.** For each secret, declare pull (external
   store authoritative) XOR push (LiteLLM/k8s authoritative); enforce that never
   both are declared.

## Tests

- Assert a consuming workload receives an ESO-materialized Secret.
- Assert startup fails closed when reloader is absent.
- Assert an external-store update propagates through ESO and triggers a reloader
  restart of the consumer.
- Assert a GUI-entered LiteLLM credential lands in the namespace Secret and
  propagates to the external store via PushSecret.
- Assert each declared secret has exactly one authority (pull XOR push).

## References

ADR 0020 (ESO-delivered credentials). DECISIONS-LOG: at-rest encryption, KMS,
and rotation cadence are baseline (out-of-scope).
