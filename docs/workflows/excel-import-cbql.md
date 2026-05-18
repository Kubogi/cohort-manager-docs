# Workflow: Excel import — Cán bộ quản lý (CBQL)

How to import the management-staff roster from an Excel file matching the `Mẫu CB QL` template, and how the import attaches each new CBQL to its `DaiDoi` under the chosen khóa.

## When to use

The user has an Excel workbook handed over by a partner school listing the staff officers assigned to each đại đội for a cohort. They want every officer to appear in `CanBoQuanLy` and the corresponding rows on the "Khóa học → Cán bộ quản lý" tab — without typing each one in manually.

## Entry point

Frontend: **Thông tin chung → Khóa học → Cán bộ quản lý → Nhập excel**.
Backend: [`POST /api/can-bo-quan-ly/import`](../api/endpoints/can-bo-quan-ly/post-import.md).

## Excel layout

### Sheet matching

The importer takes the first worksheet whose name, **lowercased and stripped of whitespace**, contains `cbql`. This makes the matcher tolerant of casing and accidental spacing:

| Sheet name | Matches? |
|---|---|
| `Mẫu CB QL` | ✅ (normalises to `mẫucbql`) |
| `DS CBQL` | ✅ |
| `cbql-k175` | ✅ |
| `Roster` | ❌ |

No matching sheet → the request fails with `400 NO_CBQL_SHEET`.

### Row structure

Header sits on row 1; data starts on row 2. A person can span **1 or 2 rows**:

- A row whose column A (`TT` — số thứ tự) is non-empty marks the **start of a new person**.
- Subsequent rows with empty `TT` are **continuation rows** of the previous person. The importer concatenates their column-D values into the person's `chucVu` (joined with `"; "`) and keeps the first non-empty `daiDoi` code / ghi chú it encounters.

| Col | Field | Notes |
|---|---|---|
| A | `TT` | Boundary marker only. Not stored. |
| B | `hoTen` | Required (taken from the first row). |
| C | `capBac` | Stored verbatim (e.g. `"4//"`, `"3//"`). |
| D | `chucVu` | Multi-line — concatenated with `"; "` if it spans 2 rows. |
| E | `ghiChu` | Free text. |
| F | `daiDoi` code | E.g. `"c14"`. Lowercased before lookup. |

## Behavior

### Dedup (CanBoQuanLy)

The importer dedups by **`hoTen`**. A name already present in `CanBoQuanLy` → counted under `duplicates`, no re-insert. A duplicate hoTen is still eligible for the DaiDoi attachment step (see below) — useful when re-running an import after fixing đại đội codes in the spreadsheet.

### DaiDoi attachment

When the request includes a `khoa` form field, every person in the workbook is looked up against `DaiDoi.findOne({ ten: daiDoiCode, khoa })` (codes are lowercased). On match, the CBQL is appended to the daiDoi's `canBo` array — along with **placeholder** values for the aligned arrays:

| Array | Pushed value |
|---|---|
| `canBo` | `<CanBoQuanLy._id>` |
| `soQD` | `""` |
| `ngayQD` | `null` |
| `hieuLuc` | `{}` (empty subdoc) |

The `DaiDoi.pre('validate')` hook requires those four arrays to have the same length; pushing placeholders preserves that invariant. Admins can fill in the real QĐ values from the "Sửa đại đội" modal later.

Codes that don't match a daiDoi under the khóa are returned in `missingDaiDoi`.

### Response

```json
{
  "data": {
    "inserted": 9,
    "duplicates": 2,
    "attached": 11,
    "missingDaiDoi": ["c99"]
  }
}
```

The frontend renders these counts in a follow-up modal; `missingDaiDoi` is also flagged in a separate warning notification.

## Edge cases

| Scenario | Behavior |
|---|---|
| Workbook has no `cbql`-matching sheet | Whole request fails with `400 NO_CBQL_SHEET`. Nothing is written. |
| `khoa` form field omitted | CBQL rows are inserted, but `attached` is 0 and `missingDaiDoi` is empty (the lookup step is skipped). |
| Same person's `chucVu` wraps onto a continuation row | Joined with `"; "` so both lines survive. |
| `daiDoi` code typo (e.g. `"c188"` instead of `"c18"`) | The CBQL is inserted, but the code shows up in `missingDaiDoi` and is **not** attached. Fix the spreadsheet (or rename the daiDoi) and re-run; the previously-inserted name will be counted as a duplicate but can still be attached. |
| Re-import same file | All `inserted` become `duplicates`; `attached` rises by the number of matches that previously failed to attach. |

## Smoke test

1. Pick a khóa under which daiDois named `c14`..`c24` already exist (the student importer auto-creates these from sheet names).
2. Open the "Nhập excel" modal and upload `Mẫu CB QL và trực kiểm soát QS.xlsx`.
3. Expect: `inserted: 11`, `duplicates: 0`, `attached: 11`, `missingDaiDoi: []`.
4. Re-upload immediately: `inserted: 0`, `duplicates: 11`, `attached: 0`, `missingDaiDoi: []` (each daiDoi already has the same canBo from step 2).

## Related

- [`POST /api/can-bo-quan-ly/import`](../api/endpoints/can-bo-quan-ly/post-import.md)
- [`CanBoQuanLy schema`](../backend/schemas/CanBoQuanLy.md)
- [`DaiDoi schema`](../backend/schemas/DaiDoi.md)
- [Khóa học UI page](../frontend/pages/thong-tin-chung/khoa-hoc.md)
