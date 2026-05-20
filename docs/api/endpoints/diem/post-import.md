# POST /api/diem/import

**Endpoint**: `POST /api/diem/import`
**Authentication**: Required (Bearer)
**Roles**: `admin`, `staff`, `teacher`
**Source**: [diem.route.js:27](../../../../backend/src/routes/diem.route.js#L27), [diem.controller.js:30](../../../../backend/src/controllers/diem.controller.js#L30), [diem.service.js:255](../../../../backend/src/services/diem.service.js#L255)
**Last verified**: 2026-05-16

## Description

Imports grades for one **môn (subject)** from an Excel grade-book. Each worksheet in the workbook is scanned for student rows; for every student whose `maSV` is in the supplied `(khoa, donViLienKet)` scope, the corresponding `SinhVien.diem[]` subdocument for `mon` is created (if absent) or updated (if present).

Per-row writes are deferred until the whole workbook has been parsed and then dispatched in parallel chunks of 50 student saves, which keeps a thousand-row import to a few seconds.

> No Joi validation middleware is applied to this route. Required form-field checking (`he`, `mon`, `khoa`, `donViLienKet`) happens inside the controller and service, not via a validator.

## Request

`multipart/form-data`:

| Field | Type | Required | Notes |
|---|---|---|---|
| `file` | `.xls` / `.xlsx` | yes | The grade-book. Multer rejects other extensions and >20 MB. |
| `he` | string | yes | Education system. Must be `"Đại học"` or `"Cao đẳng"`. |
| `mon` | string | yes | Subject name. Must match a `MON_ENUM` value belonging to the supplied `he`: `"Đại học"` subjects are the first four of `MON_ENUM`, `"Cao đẳng"` subjects are the rest. See [SinhVien schema](../../../backend/schemas/SinhVien.md) for the full enum. |
| `khoa` | string (ObjectId) | yes | Cohort scope. Students outside this cohort are not matched. |
| `donViLienKet` | string (ObjectId) | yes | Partner school scope. Stored as `truong` on each `SinhVien`. |

If `he` is unrecognised the service returns `400 BAD_HE`. If `mon` does not belong to `he`, the service returns `400 MON_HE_MISMATCH`. Missing `khoa` or `donViLienKet` returns `400 MISSING_SCOPE`.

### Workbook structure

For each worksheet:

- Row 7 (columns A/B/C) is used only to emit a **subject-title mismatch warning** — it is NOT used for `he` detection. `he` detection is performed by `detectSheetHe()`, which reads **row 9, column G**: if the cell contains `"miệng"` the sheet is treated as Cao đẳng; if it contains `"giữa"` the sheet is treated as Đại học. If a sheet is detected as a different `he` than the request, that sheet contributes a warning to `errors` and is skipped.
- Row 10 onwards are data rows. The service skips rows whose `maSV` (column B) is not digits-only (signature / footer rows).
- Columns read depend on `he` (see [excel-pipeline architecture](../../../architecture/excel-pipeline.md) for the layout map):
  - `thuongXuyen`, `mieng` (Cao đẳng only), `giuaHP`, `hetHP` — at column positions defined by `HE_LAYOUTS[he].cols`.

The student is matched by `maSV` against students pre-loaded from `(khoa, donViLienKet)` filtered by the caller's `allowedUnits` / `teacherScope`. **Students outside scope are not visible to the importer**; they appear in `missingStudents` rather than being inserted.

When a `Diem` subdocument already exists for `mon` on the matched student, only the fields supplied in the row are overwritten — empty cells leave the previous value alone. When no `Diem` exists, a new one is created with `thuongXuyen: 0, giuaHP: 0, hetHP: 0` defaults (plus `mieng: 0` for Cao đẳng) and any supplied values overlaid. The `tbMon` average is auto-computed by `pre('validate')` on save.

### Dash cells = miễn thành phần

A cell containing `-` or `—` (em-dash) on a grade column is treated as "exempt this exam". The importer stores `null` for that component, which the rescaled `tbMon` formula then skips (see [POST /api/diem](./post.md#per-component-exemption-null)). Two consequences:

- **Insert path**: a row with mixed dashes and numbers writes the numbers as-is and the dashes as `null`. A row of all dashes still inserts a `Diem` (all-null components, `tbMon: null`) — useful for marking a student as fully exempt-by-component without using `monMienHoc`.
- **Re-import path**: a dash overwrites a previously-saved number → that component flips to exempt and `tbMon` is recomputed on the next save. A blank cell still preserves the previous value.

Out-of-range numerics (`> 10`, `< 0`, non-numeric and non-dash) still land in `errors` as before.

## Response

### 200 OK

```json
{
  "data": {
    "inserted": 87,
    "updated": 142,
    "skipped": 5,
    "errors": [
      { "sheet": "Sheet1", "row": 14, "maSV": "SV023", "reason": "Điểm phải nằm trong [0, 10]" }
    ],
    "missingStudents": [
      { "sheet": "Sheet1", "row": 18, "maSV": "SV099", "hoTen": "Nguyễn Văn X" }
    ]
  }
}
```

| Field | Type | Meaning |
|---|---|---|
| `inserted` | number | Students that did not have a `Diem` for `mon` before this import and got a new one. |
| `updated` | number | Students whose existing `Diem` for `mon` was modified. |
| `skipped` | number | Rows that could not be applied (sum of two cases: student not in scope, or read/save error). |
| `errors` | object[] | Per-row failures. Includes `sheet`, `row`, `maSV`, and a Vietnamese `reason`. Sheet-level warnings have no `row`. |
| `missingStudents` | object[] | Rows where `maSV` was not found in the resolved `(khoa, donViLienKet)` scope. Useful for cleaning up the workbook before re-importing. |

### Errors

| Status | `error.code` | When |
|---|---|---|
| 400 | (no code, message-only) | No file in the request: `{ error: { message: "Vui lòng chọn file Excel" } }`. |
| 400 | (no code, message-only) | Missing `he`, `mon`, `khoa`, or `donViLienKet` form field: `{ error: { message: "Thiếu hệ, môn, khóa hoặc đơn vị liên kết" } }` (controller-level). |
| 400 | `BAD_HE` | `he` not in `{Đại học, Cao đẳng}`. |
| 400 | `MON_HE_MISMATCH` | `mon` does not belong to the supplied `he`. |
| 400 | `MISSING_SCOPE` | `khoa` or `donViLienKet` empty (defense-in-depth after the controller check). |
| 401 | `UNAUTHORIZED` | Missing or invalid access token. |
| 403 | `FORBIDDEN` | Role is not in `['admin', 'staff', 'teacher']`. Viewer is **not** allowed on this endpoint. |
| 413 | — | Multer rejects files over 20 MB. |

Per-row issues do not return non-2xx. They land in `errors` and `missingStudents` alongside the successful counts.

## Example

```bash
curl -X POST https://example.com/api/diem/import \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -F "file=@./sotong.xlsx" \
  -F "he=Đại học" \
  -F "mon=Đường lối quốc phòng và an ninh của ĐCSVN" \
  -F "khoa=64a1b2c3d4e5f6a7b8c9d0e1" \
  -F "donViLienKet=64a1b2c3d4e5f6a7b8c9d0e2"
```

## Related

- [POST /api/diem](./post.md) — create a single grade row
- [GET /api/diem/summary](./get-summary.md)
- [Excel pipeline architecture](../../../architecture/excel-pipeline.md) — template map and column layout per `he`
- [SinhVien schema](../../../backend/schemas/SinhVien.md) — note `diem[]` is embedded; `tbMon` is computed
