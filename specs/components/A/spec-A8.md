# SPEC A8 — LibreChat (locked-down config; OpenAI ↔ A2A adapter if needed)

> kind: COMPONENT · workstream: A · tier: T1
> upstream: [] · downstream: [B1] · adrs: [0007, 0025, 0017, 0038, 0039, 0034, 0031, 0030] · views: [6.1, 6.9, 7.1]
> canon-glossary: b0edae10a2e6 · canon-interface: 0ce201d5d5af

## 1. Purpose & Problem Statement

A8 installs and configures **LibreChat** as the platform's locked-down end-user chat frontend
(architecture-overview.md §6.1, §7.1; ADR 0007). End users talk to **Platform Agents**, which
appear in LibreChat's endpoint picker as if they were models, surfaced through **LiteLLM**. Skills,
tools, MCP, and RAG are configured server-side on the agent and are **not** user-controllable from
the UI; LibreChat exposes only the endpoint picker, chat history, and **file upload**.

The platform's posture (ADR 0007) is "locked down" for native agents, plugins, and MCP — all
disabled in `librechat.yaml` — but **file upload is permitted** because it is genuinely useful
(HolmesGPT diagnostics: log/screenshot/config-dump uploads). Uploads route to the **`Memory` CRD**
for the active conversation (ADR 0025 access modes) and are gated by **OPA** with three outcomes
(allowed / forbidden / allowed-after-scanning). A8 also owns delivery of the **OpenAI ↔ A2A
adapter** *only if* LiteLLM does not ship that translation in OSS by install time — LibreChat
itself stays unmodified either way.

## 2. Scope

### 2.1 In scope
- LibreChat Helm install (unmodified upstream image) + the configured `librechat.yaml`.
- Lock-down config: native agent feature, plugins, and MCP **disabled**; endpoint picker, chat
  history, and **file upload** visible.
- Keycloak SSO; endpoint visibility scoped from platform JWT claims (`capability_set_refs`,
  `platform_roles`, etc. — §6.9).
- Upload → `Memory` CRD wiring (ADR 0025 access modes) for the active conversation.
- OPA upload gating with three outcomes (allowed / forbidden / allowed-after-scanning); the
  scan-then-allow path may invoke an approval step (ADR 0017). Default policy **permissive for
  everyone**; abuse limits (rate/size/MIME) enforced by OPA, not RBAC-gated to users.
- **OpenAI ↔ A2A adapter** (small Python service between LibreChat and LiteLLM) — delivered *only
  if* LiteLLM lacks the translation in OSS at install (decision confirmed during implementation).
- End-to-end streaming preservation across the OpenAI-compat ↔ A2A boundary.
- LibreChat-resident state (conversations, accounts) in the shared Postgres deployment.
- Standard §14.1 Workstream-A deliverable set (see §8).

### 2.2 Out of scope (and where it lives instead)
- The OpenAI-compat ↔ A2A translation **inside LiteLLM** — **LiteLLM (A1)**; A8's adapter is a
  fallback only.
- The default chat endpoint logic — **Interactive Access Agent (A16)** is what LibreChat connects
  to from day one.
- Authoring the upload OPA Rego content — **OPA policy library (B3/B16)**; A8 contributes its
  component-specific bundle and integration.
- The upload-policy editor surface — **Headlamp policy editor (ADR 0039)**, framework A9/A22;
  policy simulator is A20 (ADR 0038).
- `Memory` CRD / `MemoryStore` definition and access modes — **ARK (A5)** / **B4** / ADR 0025.
- The content-scan (AV/DLP) engine itself — `[PROPOSED — not in source: scanning backend is
  design-time]`; A8 invokes the scan callback.
- SSO proxy config — **B1**.

## 3. Context & Dependencies

**Upstream consumed:** none declared in piece-index (W0). Functionally A8 binds to LiteLLM (A1) as
its model gateway, Keycloak (baseline), the `Memory` CRD (ARK A5), and shared Postgres (B4
`Postgres`) — these are foundation/baseline surfaces it configures against, with mock-out per the
§10 documentation-before-implementation process where a dependency has not landed.

**Downstream consumers:**
- **B1** — SSO/auth proxy layer fronts LibreChat with oauth2-proxy / Keycloak.

**ADR decisions honored:**
- **ADR 0007** — LibreChat (configured) as the chat frontend; native agents/plugins/MCP disabled
  via `librechat.yaml`; file upload permitted and contained by OPA + scanning; A8 owns the
  OpenAI ↔ A2A adapter if LiteLLM doesn't ship it in OSS; LibreChat installed unmodified.
- **ADR 0025** — uploads route to the `Memory` CRD per its access modes (mode lives on
  `MemoryStore`, not the binding).
- **ADR 0017** — the allowed-after-scanning path may invoke an approval step.
- **ADR 0039 / ADR 0038** — upload OPA bundle is editable via the Headlamp policy editor and
  testable via the policy simulator before promotion (consumed, not owned by A8).
- **ADR 0034** — LibreChat conversation events emit via the audit adapter.
- **ADR 0031** — A8 events fall under `platform.audit.*` (conversation events) and uploads may
  touch `platform.approval.*` (via B19); per-event names deferred to B12.
- **ADR 0030** — A8 ships no platform CRD; the optional adapter uses URL-path versioning.

## 4. Interfaces & Contracts

### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
- A8 defines **no platform CRD**. It **consumes** `Memory` (ARK-owned) — `memoryStoreRef` — as the
  upload destination, and indirectly `MemoryStore` (XR, B4) — `accessMode`, `backendType` (ADR
  0025). May produce an `Approval` (B19/Argo-owned) on the scan-then-allow path —
  `requestingAgent`, `actionType`, `actionAttributes`, `defaultLevel`, `evidenceRefs[]`,
  `decision`, `decidedBy`, `decidedAt`.
- Consumes `Postgres` connection secret for LibreChat state.

### 4.2 APIs / SDK surfaces
- **LibreChat ↔ LiteLLM** over OpenAI-compatible HTTP; LiteLLM translates to A2A so Platform
  Agents appear as models (§6.1, §7.1). Streaming chunks preserved across the boundary.
- **OpenAI ↔ A2A adapter** (optional, A8-owned) — a small Python service; if shipped, exposes a
  URL-path-versioned (`/v1/...`) HTTP surface between LibreChat and LiteLLM (ADR 0030 §3.3).
  `[PROPOSED — not in source: exact adapter API shape is design-time and conditional.]`
- `librechat.yaml` — the configuration surface (ADR 0007's load-bearing mechanism).

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
- **Emitted:** `platform.audit.*` for conversation events (via the audit adapter, ADR 0034).
- **Consumed/triggered:** scan-then-allow may emit/await `platform.approval.*` (via B19).
  `[PROPOSED — not in source: specific event names deferred to B12.]`

### 4.4 Data schemas / connection-secret contracts
- LibreChat-resident state (conversations, accounts) uses the **`Postgres` uniform connection
  secret** (host/port/user/password/dbname; ADR 0044) and inherits backup/PITR/DR posture (§7.2).
- Upload payloads land in the `Memory` CRD's backing store per ADR 0025; the memory access path is
  ARK-owned.

## 5. OSS-vs-Custom Decision
- **Named project:** LibreChat (glossary: "Locked-down chat frontend (A8; ADR 0007)").
- **Mode:** install + **configure** (Helm + `librechat.yaml`), unmodified upstream — *no fork*.
  Plus a small **build-new** Python OpenAI ↔ A2A adapter, **conditional** on LiteLLM lacking the
  translation in OSS at install time (ADR 0007 consequence; backlog §1.16).
- **Version:** pin a tested LibreChat version `[PROPOSED — not in source]`.
- **ADR linkage:** ADR 0007 (chosen over Open WebUI / custom UI; configuration story is decisive).
- **Rationale:** OSS, Keycloak-SSO-friendly, supports a real `librechat.yaml` profile for the
  frontend-only posture; custom UI rejected (highest build cost for an intentionally minimal
  frontend).

## 6. Functional Requirements
- **REQ-A8-01:** A8 SHALL install LibreChat via Helm using the unmodified upstream image, as a
  single ArgoCD-synced release.
- **REQ-A8-02:** A8 SHALL ship a `librechat.yaml` that **disables** the native agent feature,
  plugins, and MCP, and leaves the endpoint picker, chat history, and file upload visible.
- **REQ-A8-03:** A8 SHALL SSO LibreChat through Keycloak and scope visible endpoints from platform
  JWT claims (a user sees only Platform Agents permitted by their `capability_set_refs`/roles).
- **REQ-A8-04:** A8 SHALL route uploaded files to the active conversation's `Memory` CRD (ADR 0025)
  rather than to LibreChat local filesystem.
- **REQ-A8-05:** A8 SHALL gate uploads through OPA with three outcomes — **allowed**, **forbidden**
  (policy reason returned), **allowed-after-scanning** (held, scanned, then attached) — with a
  default policy permissive for everyone and abuse limits (rate/size/MIME) enforced by OPA.
- **REQ-A8-06:** A8 SHALL invoke an approval step (ADR 0017) on the scan-then-allow path when the
  policy demands human-in-the-loop confirmation.
- **REQ-A8-07:** A8 SHALL preserve streaming end-to-end (LibreChat ↔ LiteLLM ↔ Platform Agent),
  with stream chunks preserved across the OpenAI-compat ↔ A2A boundary.
- **REQ-A8-08:** A8 SHALL deliver the OpenAI ↔ A2A adapter (small Python service, LibreChat
  unmodified) **iff** LiteLLM does not provide the translation in OSS at install; the need is
  confirmed during implementation and documented either way.
- **REQ-A8-09:** A8 SHALL store LibreChat-resident state (conversations, accounts) in the shared
  Postgres via the `Postgres` connection secret, inheriting its backup/PITR/DR posture.
- **REQ-A8-10:** A8 SHALL emit conversation events as audit records via the platform audit adapter
  (ADR 0034) — never writing audit stores directly.

## 7. Non-Functional Requirements
- **Security / multi-tenancy (§6.9):** endpoint visibility strictly claim-driven; upload abuse
  contained by OPA + scanning, not RBAC allowlists (ADR 0007); RBAC-as-floor / OPA-as-restrictor
  (ADR 0018). Forbidden file types (executables, archives outside allowlist) rejected.
- **Observability (§6.5):** LibreChat emits OTel through the standard collector; agent/gateway
  spans carry `trace_id`, deep-linkable to Langfuse (ADR 0015).
- **Scale:** sized for interactive user concurrency; streaming must not buffer to completion.
- **Versioning (ADR 0030):** pin tested LibreChat version; optional adapter uses URL-path
  versioning.

## 8. Cross-Cutting Deliverable Checklist (§14.1)
- Helm/manifests — **applicable**.
- Per-product docs (10.5) — **applicable**.
- Runbook (10.7) + backup/restore — **applicable** (state in shared Postgres; inherits posture).
- Alerts — **applicable** (LibreChat down, upload-path failures, Memory-write failures).
- Grafana dashboard (Crossplane XR) — **applicable** (chat volume, upload outcomes, stream errors).
- Headlamp plugin — **N/A as A8-owned editor** — upload-policy editing is the Headlamp **policy
  editor** (ADR 0039, framework A22) over the OPA bundle, not an A8 CRD editor.
- OPA/Rego integration — **applicable** (the three-outcome upload-gate bundle; abuse limits).
- Audit emission (ADR 0034) — **applicable** (conversation events).
- Knative trigger flow — **applicable** (scan-then-allow → `platform.approval.*` path via B19).
- HolmesGPT toolset — **N/A** — A8 is a passive frontend; HolmesGPT consumes uploads *through* the
  Memory path, not via an A8 tool. `[PROPOSED — minimal/none]`
- 3-layer tests (Chainsaw/Playwright/PyTest) — **applicable**.
- Tutorials & how-tos — **applicable** (invoke an agent via LibreChat — §10.1).

## 9. Acceptance Criteria
- **AC-A8-01 (REQ-A8-01):** ArgoCD sync brings LibreChat to Ready using the unmodified upstream
  image as one release. *(Chainsaw)*
- **AC-A8-02 (REQ-A8-02):** With the shipped `librechat.yaml`, the UI exposes the endpoint picker,
  history, and upload; native agents, plugins, and MCP are absent. *(Playwright)*
- **AC-A8-03 (REQ-A8-03):** Two users with different JWT claims see different endpoint lists
  matching their `capability_set_refs`/roles. *(Playwright)*
- **AC-A8-04 (REQ-A8-04):** An uploaded file appears in the active conversation's `Memory` CRD and
  is readable by the connected agent; nothing is written to LibreChat local FS. *(PyTest)*
- **AC-A8-05 (REQ-A8-05):** A forbidden MIME is rejected with a policy reason; an allowed MIME
  attaches; an over-size/over-rate upload is blocked by OPA. *(PyTest)*
- **AC-A8-06 (REQ-A8-06):** A scan-then-allow file requiring human confirmation produces an
  `Approval`; on approval it attaches, on rejection it does not. *(Chainsaw + PyTest, stub B19)*
- **AC-A8-07 (REQ-A8-07):** A streamed agent response renders incrementally in LibreChat; chunks
  are preserved across the OpenAI-compat ↔ A2A boundary. *(Playwright)*
- **AC-A8-08 (REQ-A8-08):** With LiteLLM translation present, no adapter is deployed and chat
  works; with translation absent (simulated), the A8 adapter is deployed and chat works — LibreChat
  image unchanged in both. *(PyTest + Chainsaw)*
- **AC-A8-09 (REQ-A8-09):** Conversations persist across a LibreChat pod restart (state in
  Postgres). *(PyTest)*
- **AC-A8-10 (REQ-A8-10):** A conversation event produces an audit record via the adapter. *(PyTest)*

## 10. Risks & Open Questions
- **R1 (med):** Whether the OpenAI ↔ A2A adapter is needed is **unresolved until install** (ADR
  0007) — A8 must carry both code paths. Blast radius: adapter is a small isolated service.
- **R2 (med):** Content-scan (AV/DLP) backend for scan-then-allow is `[PROPOSED — not in source]`;
  A8 invokes a callback but the scanner is design-time.
- **R3 (low):** Upload OPA bundle content is owned by B3/B16; A8 must integrate but not author the
  full Rego. Coordination risk only.
- **R4 (low):** Streaming preservation across the A2A boundary depends on LiteLLM (A1) behavior;
  covered by AC-A8-07 against a real/mocked LiteLLM.

## 11. References
- architecture-overview.md §6.1 (gateway / OpenAI-compat ↔ A2A, lines ~170–246); §6.9
  (multi-tenancy); §7.1 (interactive chat, lines ~1007–1076); §7.2 (LibreChat state in Postgres);
  §14.1 (A8 row, line ~1674).
- ADR 0007 (LibreChat configured, file upload, adapter); ADR 0025 (memory access modes);
  ADR 0017 (approval on scan-then-allow); ADR 0038 (policy simulator); ADR 0039 (Headlamp policy
  editor); ADR 0034 (audit); ADR 0031 (CloudEvents); ADR 0030 (versioning).
- Views V6-01 (Gateway), V6-09 (Multi-tenancy).
- Related pieces: A1 (LiteLLM), A16 (Interactive Access Agent default endpoint), A5 (Memory CRD),
  B1 (SSO), B3/B16 (upload OPA), B19 (approval), A22 (policy editor), A20 (simulator).
