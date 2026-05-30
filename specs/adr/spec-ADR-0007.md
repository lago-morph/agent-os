# SPEC ADR-0007 — LibreChat (configured) as the chat frontend [PROPOSED]

> kind: ADR · workstream: — · tier: T1
> upstream: [A1;A16] · downstream: [B1] · adrs: [0007] · views: [6.1;6.9;6.11;7.1]
> canon-glossary: 6aadcc2a4f38 · canon-interface: 54f5ede58e5f

## 1. Purpose & Problem Statement
ADR 0007 fixes LibreChat (configured, installed unmodified) as the chat frontend (component A8). This SPEC states what honoring that decision requires: LibreChat is installed unmodified via Helm and configured via `librechat.yaml` to disable the native agent feature, plugins, and MCP — leaving only the endpoint picker (Platform Agents via LiteLLM), chat history, and file upload visible; SSO is via Keycloak with visible endpoints scoped from platform JWT claims; uploads route to the `Memory` CRD and are OPA-gated with allowed / forbidden / allowed-after-scanning outcomes, default-permissive. The decision is settled; this SPEC captures the obligations it imposes and the acceptance criteria that prove it is honored.

## 2. Scope
### 2.1 In scope
- LibreChat installed unmodified (Helm) + the configured `librechat.yaml` (component A8).
- Disabled surfaces: native agent feature, plugins, MCP. Visible surfaces: endpoint picker, chat history, file upload.
- Keycloak SSO; endpoint visibility scoped from platform JWT claims (`capability_set_refs`, roles).
- Upload → `Memory` CRD wiring; OPA-gated upload (allowed / forbidden / allowed-after-scanning), default-permissive.
- The Interactive Access Agent (A16) as the day-one default endpoint.

### 2.2 Out of scope (and where it lives instead)
- LiteLLM gateway + endpoints — component A1; Interactive Access Agent build — A16.
- SSO/auth-proxy layer config — B1; Keycloak claim schema — ADR 0029.
- Memory access-mode enforcement — ADR 0025; approval step for scan-then-allow — ADR 0017.
- Upload OPA bundle authoring/editing/testing — B16 + Headlamp editor (ADR 0039) + simulator (ADR 0038).
- OpenAI↔A2A adapter (delivered by A8 only if LiteLLM lacks OSS translation at install) — A8/backlog §1.16.

## 3. Context & Dependencies
Upstream consumed: A1 (LiteLLM endpoints LibreChat picks from), A16 (default endpoint). Downstream consumers: B1 (SSO/auth proxy layer).
ADR decisions honored: **0007** — LibreChat unmodified + `librechat.yaml` locks the surface, uploads OPA-gated to `Memory`; **0025** — uploads routed per memory access modes; **0017** — scan-then-allow may invoke an approval step; **0029** — endpoint visibility from JWT claims; **0038/0039** — upload policy tested/edited via simulator/Headlamp.

## 4. Interfaces & Contracts
### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
Consumes `Memory` (namespaced; `memoryStoreRef`) as the upload destination per ADR 0025 access modes. No new CRD introduced by this ADR.

### 4.2 APIs / SDK surfaces
LibreChat speaks OpenAI-compatible HTTP to LiteLLM; the OpenAI↔A2A translation occurs at the gateway (or via an A8-owned adapter if not in LiteLLM OSS at install). Endpoint visibility is driven by the **Platform JWT** claims (`platform_tenants`, `platform_namespaces`, `platform_roles`, `tenant_roles`, `capability_set_refs`).

### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
Upload-policy decisions surface as OPA decisions under `platform.policy.*`; scan-then-allow that triggers an approval surfaces under `platform.approval.*`. Per-event-type names deferred to B12.

### 4.4 Data schemas / connection-secret contracts
LibreChat-resident state (conversations, accounts) lives in the shared Postgres deployment, inheriting backup/PITR/DR posture. Uploaded artifacts are attached to conversation memory via the `Memory` CRD; no direct store write from the UI.

## 5. OSS-vs-Custom Decision
Upstream project: **LibreChat**, installed unmodified via Helm and configured via `librechat.yaml` (A8); a small optional Python OpenAI↔A2A adapter is custom only if needed. Mode: install + config (no fork). Rationale per ADR 0007: LibreChat is OSS, Keycloak-SSO friendly, and uniquely supports a real configuration profile that disables agents/plugins/MCP while keeping the endpoint picker + history + upload. Rejected: Open WebUI (weaker config story), custom UI (highest build/maintenance for an intentionally minimal frontend), RBAC-gated upload (ticket friction vs policy/scanning).

## 6. Functional Requirements
- REQ-ADR-0007-01: LibreChat MUST be installed unmodified; all customization MUST live in `librechat.yaml` (no fork).
- REQ-ADR-0007-02: The native agent feature, plugins, and MCP MUST be disabled in `librechat.yaml`.
- REQ-ADR-0007-03: Only the endpoint picker, chat history, and file upload MUST be user-visible.
- REQ-ADR-0007-04: SSO MUST be via Keycloak; visible endpoints MUST be scoped from platform JWT claims.
- REQ-ADR-0007-05: Uploads MUST route to the `Memory` CRD per ADR 0025 access modes and be exposed to the connected agent.
- REQ-ADR-0007-06: OPA MUST gate uploads with three outcomes (allowed / forbidden / allowed-after-scanning); scan-then-allow MAY invoke an approval step (ADR 0017).
- REQ-ADR-0007-07: Upload MUST be default-permissive for everyone (OPA enforces rate/size/MIME), not RBAC-gated to a user set.
- REQ-ADR-0007-08: The Interactive Access Agent (A16) MUST be the default endpoint from day one.

## 7. Non-Functional Requirements
- Security/tenancy: endpoint visibility per JWT claims (§6.9, §6.11); upload abuse contained by policy + scanning, not RBAC.
- Streaming: preserved end-to-end LibreChat ↔ LiteLLM ↔ Platform Agent across the OpenAI-compat ↔ A2A boundary.
- Data: LibreChat state in shared Postgres (backup/PITR/DR).
- Power-user affordances (personal agents, plugin marketplace, user MCP) remain off by design (revisit backlog §3.12).

## 8. Cross-Cutting Deliverable Checklist
N/A — ADR. The §14.1 deliverable set is owned by component A8; conformance items appear in §9 / the PLAN.

## 9. Acceptance Criteria
Decision honored when:
- AC-ADR-0007-01: The deployed LibreChat image is the upstream image; only `librechat.yaml` differs. (REQ-01)
- AC-ADR-0007-02: The native agent feature, plugins, and MCP are absent from the UI. (REQ-02, REQ-03)
- AC-ADR-0007-03: A user sees only Platform-Agent endpoints permitted by their `capability_set_refs`/roles. (REQ-04)
- AC-ADR-0007-04: An uploaded file is attached to the conversation's `Memory` and reachable by the connected agent. (REQ-05)
- AC-ADR-0007-05: An OPA-forbidden MIME is rejected with a policy reason; an allowed-after-scanning file is held, scanned, then attached. (REQ-06)
- AC-ADR-0007-06: Default policy permits upload for any authenticated user; only rate/size/MIME limits apply (no user allowlist). (REQ-07)
- AC-ADR-0007-07: A fresh LibreChat session defaults to the Interactive Access Agent endpoint. (REQ-08)

## 10. Risks & Open Questions
- OpenAI↔A2A translation may not be in LiteLLM OSS at install (blast radius: med) — A8 owns a small adapter; decision confirmed during implementation (backlog §1.16).
- Upload abuse surface (med) — contained by OPA rate/size/MIME + scanning; simulator-tested before promotion.
- Power-user pressure for personal agents/MCP (low) — off by design; revisit per backlog §3.12.

## 11. References
- ADR 0007 (`adr/0007-librechat-locked-down-frontend.md`) — the decision.
- Enforcing components: A8 (LibreChat configured, owner), A1 (LiteLLM endpoints), A16 (Interactive Access Agent), B1 (SSO/auth proxy), A7/B16 (upload OPA), A18 (scan/audit), A9 (Headlamp policy editor).
- architecture-overview.md §5, §6.1, §6.9, §6.10, §6.11, §7.1, §7.2, §14.1; architecture-backlog.md §2.10, §1.16, §3.12.
- Related: ADR 0017, 0025, 0029, 0038, 0039.
