# SPEC <PIECE_ID> — <Title>

> kind: COMPONENT|VIEW|ADR · workstream: <A..F | —> · tier: <T0|T1|T2>
> upstream: [<piece ids>] · downstream: [<piece ids>] · adrs: [<nnnn>] · views: [<6.x>]
> canon-glossary: <hash> · canon-interface: <hash>

## 1. Purpose & Problem Statement
What this piece is and the problem it solves in the platform. 1–3 paragraphs.

## 2. Scope
### 2.1 In scope
### 2.2 Out of scope (and where it lives instead — name the owning piece)

## 3. Context & Dependencies
Upstream pieces consumed (and exactly what is consumed). Downstream consumers.
ADR decisions honored (cite ADR numbers + the constraint each imposes).

## 4. Interfaces & Contracts
Use ONLY names that appear in `_meta/interface-contract.md` / `_meta/glossary.md`.
Anything not in Canon MUST be tagged `[PROPOSED — not in source]`.
### 4.1 CRDs / XRDs (schema fields, version per ADR 0030)
### 4.2 APIs / SDK surfaces
### 4.3 CloudEvents emitted / consumed (taxonomy per ADR 0031)
### 4.4 Data schemas / connection-secret contracts

## 5. OSS-vs-Custom Decision
Named upstream project + version where applicable; fork / config / wrap / build-new;
ADR linkage; rationale. (COMPONENT pieces; N/A — <reason> for VIEW/ADR.)

## 6. Functional Requirements
Numbered, testable. `REQ-<PIECE_ID>-NN: <statement>`.

## 7. Non-Functional Requirements
Security, multi-tenancy (§6.9), observability (§6.5), scale, versioning (ADR 0030).

## 8. Cross-Cutting Deliverable Checklist
COMPONENT (Workstream A especially) — the §14.1 standard set, each marked
applicable / N/A: Helm/manifests · per-product docs (10.5) · runbook (10.7) ·
alerts · Grafana dashboard (Crossplane XR) · Headlamp plugin · OPA/Rego
integration · audit emission (ADR 0034) · Knative trigger flow · HolmesGPT
toolset · 3-layer tests (Chainsaw/Playwright/PyTest) · tutorials & how-tos.
(VIEW/ADR: `N/A — <reason>`.)

## 9. Acceptance Criteria
`AC-<PIECE_ID>-NN: <binary pass/fail>` — each maps to a REQ in §6.

## 10. Risks & Open Questions
Each with a blast-radius rating (low/med/high) and, where surfaced by review,
the reconciliation note or `[PROPOSED]` flag.

## 11. References
architecture-overview.md anchors (§ and line), ADRs, views, related pieces.
