# DAG Reciprocity Audit — `_meta/piece-index.csv`

**Audit date:** 2026-05-31  
**Rows checked:** 54 (pieces A1–C9; VIEW/ADR rows have no upstream/downstream fields and are skipped)  
**Method:** For every entry in `upstream` or `downstream`, verify the reciprocal entry exists on the referenced piece.

---

## Asymmetry Table

All 5 asymmetries originate from the recent patch that set `B4.downstream = A18;A21;A23;B19;B11`. The receiving pieces were not updated to list B4 in their `upstream`.

| # | Edge | Present on | Missing on | Class | Recommended Action |
|---|------|-----------|------------|-------|--------------------|
| 1 | B4→A18 | `B4.downstream` | `A18.upstream` | TYPO | Add `B4` to `A18.upstream` (currently `A11;A7` → becomes `A11;A7;B4`) |
| 2 | B4→A21 | `B4.downstream` | `A21.upstream` | TYPO | Add `B4` to `A21.upstream` (currently `A9;A22` → becomes `A9;A22;B4`) |
| 3 | B4→A23 | `B4.downstream` | `A23.upstream` | TYPO | Add `B4` to `A23.upstream` (currently `A7;B19` → becomes `A7;B19;B4`) |
| 4 | B4→B11 | `B4.downstream` | `B11.upstream` | **AMBIGUOUS** | **Human decision required** — context flags `B11→B4` direction as a known open question. Current CSV has B4 listing B11 as downstream (B4 is upstream of B11), but the direction may be reversed. Do not auto-add until direction is confirmed. |
| 5 | B4→B19 | `B4.downstream` | `B19.upstream` | TYPO | Add `B4` to `B19.upstream` (currently `A3;A7;A9;B5;B12` → becomes `A3;A7;A9;B5;B12;B4`) |

**Totals:** 5 asymmetries — 4 TYPO (auto-applied in proposed CSV), 1 AMBIGUOUS (B4→B11).

---

## Notes on Context-Flagged Questions

The following DAG-edge questions were flagged in the task context. None produce additional *missing-reciprocal* asymmetries beyond those already captured above, but they have semantic implications:

| Question | Current State | Finding |
|----------|---------------|---------|
| `B11→B4` direction | B4.downstream contains B11; B11.upstream does NOT contain B4 | Exactly asymmetry #4 above — classified AMBIGUOUS. |
| `A10→Postgres/B4` | A10 does not reference B4 in either direction; B4 does not reference A10 | No edge between A10 and B4 currently exists. If this edge should exist, it needs to be added in both directions. Out of scope for auto-fix. |
| `A16↔A14` direction | A14.downstream contains A16; A16.upstream contains A14 — fully reciprocal | The *reciprocity* is correct. The open question is whether the semantic *direction* is right (A14 upstream of A16, or vice-versa). No structural asymmetry to fix here — needs a human decision on direction. |

---

## Dangling References

**None.** Every piece ID referenced in any `upstream` or `downstream` field resolves to a row present in the CSV.

---

## Proposed Fixes (applied in `piece-index.proposed.csv`)

Only TYPO-class edges are applied. AMBIGUOUS edge B4→B11 is left untouched.

| Piece | Field | Before | After |
|-------|-------|--------|-------|
| A18 | upstream | `A11;A7` | `A11;A7;B4` |
| A21 | upstream | `A9;A22` | `A9;A22;B4` |
| A23 | upstream | `A7;B19` | `A7;B19;B4` |
| B19 | upstream | `A3;A7;A9;B5;B12` | `A3;A7;A9;B5;B12;B4` |
