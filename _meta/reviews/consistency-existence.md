# Consistency-Existence Check: Spec/Plan Corpus

**Date:** 2026-05-31  
**Scope:** specs/ (119 files) + plans/ (119 files) versus _meta/glossary.md, _meta/interface-contract.md, and specs/adr/

---

## Task 1: CRD/XRD References

**Methodology:** grep specs/ and plans/ for backtick-quoted CamelCase identifiers; cross-check against canonical kinds in glossary.md and interface-contract.md.

**Result:** 31 non-canonical CRD/XRD references found. After filtering:
- **Kubernetes built-ins** (ConfigMap, Secret, CustomResourceDefinition, etc.) — not errors
- **Third-party vendor kinds** (Workflow, Cluster, RDSInstance from Crossplane, ApiServerSource from Knative, etc.) — not errors
- **Logical terms** (Tenant — documented as "logical isolation boundary" not a CRD) — not errors

**Genuine offenders (platform CRDs referenced but not in Canon):**
1. `SearchIndex` — used in specs/components/B/spec-B4.md:110 as claim form of `XSearchIndex`, **NOT listed in glossary.md or interface-contract.md**
2. `MongoDocStore` — used in specs/views/spec-V6-12.md:76 as claim form of `XMongoDocStore`, **NOT listed in Canon**
3. `ObjectStore` — used in specs/components/B/spec-B4.md:111 as claim form of `XObjectStore`, **NOT listed in Canon**

**Note:** These three are claim forms per ADR 0041 (XR ↔ claim naming convention). The canonical glossary documents `AuditLog` and `GrafanaDashboard` as claim forms (lines 41, 82) but omits `SearchIndex`, `MongoDocStore`, and `ObjectStore` despite them being referenced throughout the architecture specs. This is a **Canon incompleteness**, not a spec error.

**Count of missing CRDs:** 3

---

## Task 2: CloudEvent type references

**Methodology:** grep specs/ and plans/ for dotted reverse-DNS tokens (platform.*.* style); cross-check against the ten canonical event namespaces in interface-contract.md §2.

**Result:** All mentioned CloudEvent types fall under the ten canonical namespaces:
- `platform.lifecycle.*`
- `platform.audit.*`
- `platform.gateway.*`
- `platform.policy.*`
- `platform.capability.*`
- `platform.evaluation.*`
- `platform.approval.*`
- `platform.observability.*`
- `platform.tenant.*`
- `platform.security.*`

**Genuine offenders:** None found.

**Count of missing event types:** 0

**[PROPOSED — not in source] references:** 376 total occurrences across corpus (expected — specs intentionally defer per-event-type schemas to component B12's registry per interface-contract.md §6, line 8).

---

## Task 3: ADR references

**Methodology:** grep specs/ and plans/ for ADR-NNNN tokens; verify corresponding specs/adr/spec-ADR-NNNN.md file exists for each.

**Result:** 41 distinct ADRs referenced (ADR-0001 through ADR-0041). All have corresponding spec files present.

**Genuine offenders:** None found.

**Count of missing ADR specs:** 0

---

## Task 4: Front-matter canon hash consistency

**Methodology:** Sample ~15 spec files across components, views, and ADRs; examine canon-glossary and canon-interface hash fields in the inline metadata at the top of each file. Check for length/format consistency.

**Result:** **Significant inconsistency detected.** Hash values vary widely in format and truncation:

| Length | Example Value | Count |
|--------|---------------|-------|
| 8 chars | `b0edae10` | 1 |
| 12 chars | `45ee7b798c47` | 1 |
| 12 chars | `54f5ede58e5f` | 28 |
| 16 chars | `6aadcc2a4f383a26` | 4 |
| 40 chars (full SHA-1) | `b0edae10a2e649ba...` | 5 |
| "FROZEN" (literal) | `FROZEN` | 13 |
| "see _meta/..." (reference) | `see _meta/interface-contract.md` | 13 |
| Empty/absent | — | 38 |

**Findings:**
- **Truncation inconsistency:** Values are truncated to 8, 12, or 16 hex digits, or left full-length (40 chars for SHA-1), with no consistent policy.
- **Format variance:** Some files use literal `FROZEN`, others use `see _meta/...` references, others omit the field entirely.
- **No normalizing rule:** The canon-interface and canon-glossary hash fields lack a uniform format definition (e.g., "always 12 hex digits" or "reference via file path, not hash").

**Example files with conflicting formats:**
- spec-A1.md: `canon-glossary: b0edae10` (8 chars) + `canon-interface: 0ce201d5` (8 chars)
- spec-ADR-0027.md: `canon-interface: 45ee7b798c47` (12 chars)
- spec-ADR-0041.md: `canon-interface: FROZEN` (literal)
- spec-V6-08.md: `canon-interface: see _meta/interface-contract.md` (reference)

**This is a documentation pattern inconsistency, not a missing-entity error.** It suggests the Canon hash fields were added/maintained ad hoc without a locked schema.

---

## Summary

| Task | Offender Type | Count |
|------|---|---|
| Task 1 (CRD/XRD) | Missing claim-form CRDs in Canon | 3 |
| Task 2 (CloudEvent types) | Event types outside canonical namespaces | 0 |
| Task 3 (ADR references) | ADR specs with no file | 0 |
| Task 4 (Canon hash format) | Format inconsistency (not an entity count) | Pattern issue documented |
| — | `[PROPOSED — not in source]` total | 376 |

**Recommendation:** Task 1 offenders (SearchIndex, MongoDocStore, ObjectStore) should be added to glossary.md as claim forms per ADR 0041, matching the treatment of AuditLog and GrafanaDashboard. Task 4 requires a Canon revision to lock the hash field format (truncation depth, or reference-style policy).
