# Workflow: Excel import — Trực và kiểm soát quân sự

How to bulk-import the daily military-duty roster from an Excel file matching the `trực kiểm soát QS` template.

## When to use

The user has a duty-roster spreadsheet covering several weeks (one entry per day) and wants every day to land in the `TrucQuanSu` collection so the "Khóa học → Trực và kiểm soát quân sự" tab shows the schedule without manual data entry.

## Entry point

Frontend: **Thông tin chung → Khóa học → Trực và kiểm soát quân sự → Nhập Excel**.
Backend: [`POST /api/truc-quan-su/import`](../api/endpoints/truc-quan-su/README.md#post-apitruc-quan-suimport).

## Excel layout

### Sheet matching

The importer takes the first worksheet whose name, **lowercased and stripped of whitespace**, contains `trực` or `truc`. Either Vietnamese diacritics or the plain ASCII variant works:

| Sheet name | Matches? |
|---|---|
| `trực kiểm soát QS` | ✅ |
| `Trực Kiểm Soát QS` | ✅ |
| `truc kiem soat` | ✅ |
| `Schedule` | ❌ |

No matching sheet → `400 NO_TRUC_SHEET`.

### Row structure

Rows 1–4 are header (the template uses merged cells across those rows). **Data starts at row 5**, with **two rows per entry**:

```
Row 5: Thứ Hai     Nguyễn Đức Đăng  Dương Tuấn Anh  ... | Nguyễn Thị Hương | Nguyễn Thị Hương | ...
Row 6: 12/29/25                                          | Trịnh Thị Trang  |                   | ...
```

| Col | TrucQuanSu field | Row 1 of pair | Row 2 of pair |
|---|---|---|---|
| A | (day-of-week) / `ngay` | `"Thứ Hai"` (ignored) | `"12/29/25"` (M/D/YY) or a real Date cell |
| B | `trucLanhDao` | name | concat with row-1 (space-joined) |
| C | `trucChiHuy` | name | concat |
| D | `truongKhungNhaD1D2` | name | concat |
| E | `trucBan` | name | concat |
| F | `trucQuanYNgay[0]` / `[1]` | **first** day-shift quân-y | **second** day-shift quân-y (the only column where row 2 carries a *separate* person) |
| G | `trucQuanYDem` | name | concat |
| H | `trucDienNuoc` | name | concat |
| I | `donViKiemSoatQuanSu` | text | concat |
| J | `ghiChu` | text | concat |

#### Why F is special

The `TrucQuanSu` schema declares `trucQuanYNgay: [String]` with a default of `['', '']` — two doctors share the day shift. In the template, those two names appear stacked across the two rows of the entry. The importer treats those values as a 2-element array `[row1.F, row2.F]` *rather than* concatenating them.

#### Why other columns are concatenated

A single person's name can wrap from row 1 onto row 2 when the cell is narrow (e.g. `"Trịnh Thị"` + `"Trang"` → `"Trịnh Thị Trang"`). Concatenating both rows handles that uniformly.

#### Date parsing

The importer prefers `cell.value` when ExcelJS has resolved it as a `Date`. Otherwise it falls back to parsing the cell text as `M/D/YY` (two-digit year → 2000 + Y). Pairs with an unparseable date are **silently skipped** — this lets the importer tolerate trailing blank rows below the last entry.

## Behavior

### Dedup

By `ngay` (the schema has `ngay: { unique: true }`). Repeated dates — whether already in the DB or repeated in the file — are counted under `duplicates`; the request never fails because of a duplicate date.

### Response

```json
{
  "data": {
    "inserted": 42,
    "duplicates": 0,
    "errors": []
  }
}
```

The frontend renders these counts in a follow-up modal.

## Edge cases

| Scenario | Behavior |
|---|---|
| Workbook has no `trực`/`truc` sheet | `400 NO_TRUC_SHEET`. Nothing is written. |
| Trailing blank rows at the bottom | Silently skipped (date parse fails). |
| Row pair with valid date but all name columns empty | Inserted as a row with empty strings everywhere — admins can edit later. |
| Re-import same file | All entries become `duplicates`; `inserted: 0`. |
| Sheet has only the header rows (no data) | `inserted: 0`, `duplicates: 0`, `errors: []`. |

## Smoke test

1. Open the "Nhập Excel" modal on Tab 2 and upload `Mẫu CB QL và trực kiểm soát QS.xlsx`.
2. Expect: `inserted: 42` (the template has 42 entries — (88 - 4) / 2), `duplicates: 0`.
3. Re-upload immediately: `inserted: 0`, `duplicates: 42`.
4. Verify that each row's `trucQuanYNgay` is a 2-element array (e.g. `["Nguyễn Thị Hương", "Trịnh Thị Trang"]`), not a single string.

## Related

- [`POST /api/truc-quan-su/import`](../api/endpoints/truc-quan-su/README.md#post-apitruc-quan-suimport)
- [`TrucQuanSu schema`](../backend/schemas/trucQuanSu.md)
- [Khóa học UI page](../frontend/pages/thong-tin-chung/khoa-hoc.md)
