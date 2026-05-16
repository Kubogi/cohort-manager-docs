# Excel Pipeline

Excel is a load-bearing part of this system. Three flows touch it:

1. **Import students** (`POST /api/sinh-vien/import`) — admin uploads a multi-sheet roster `.xlsx` (one sheet per battalion); the backend parses it with the **`xlsx` (SheetJS)** library, treats each sheet name as a `đại đội`, and inserts students.
2. **Import grades** (`POST /api/diem/import`) — admin/staff/teacher uploads a grade-book for one `(he, mon)` pair; the backend parses each worksheet for student rows and creates or updates the `SinhVien.diem[]` subdocument for that subject, with scope filtering by `(khoa, donViLienKet)` plus the caller's `allowedUnits` / `teacherScope`.
3. **Export forms** (`GET /api/bieu-mau/export`) — frontend picks a template by `(hệ, danhSách, môn)`, backend opens the matching file from `backend/forms/` with `exceljs`, fills it with student/grade data, returns the binary.

> The two flows use **different libraries** — `exceljs` for export (writing into templates with rich formatting) and `xlsx`/SheetJS for import (faster read-only parsing). Don't mix them up when adding new endpoints.

This document explains the export template map (the most complex piece), the import file format, and the integer-arithmetic convention that keeps `tbMon` averages consistent.

## 1. The export template map

The mapping from `(hệ, danhSách)` → on-disk template file lives in `backend/src/config/bieuMau.config.js`. There are 11 templates total: 5 for Đại học, 6 for Cao đẳng. Each template has a *type* (`diemDanh` | `singleGrade` | `soDiem`) and a *sheet mode* (`single` | `multiByMon`).

### Đại học templates

| `danhSach` value | File | type | sheetMode | gradeField |
|---|---|---|---|---|
| Danh sách điểm danh | `Hệ ĐH/DS điểm danh.xlsx` | `diemDanh` | single | — |
| Danh sách điểm thường xuyên | `Hệ ĐH/DS điểm Thường xuyên.xlsx` | `singleGrade` | multiByMon | `thuongXuyen` |
| Danh sách điểm kiểm tra giữa học phần | `Hệ ĐH/DS điểm giữa kỳ.xlsx` | `singleGrade` | multiByMon | `giuaHP` |
| Danh sách điểm hết học phần | `Hệ ĐH/DS điểm hết HP.xlsx` | `singleGrade` | single | `hetHP` |
| Sổ điểm từng môn | `Hệ ĐH/Sổ điểm từng môn.xlsx` | `soDiem` | single | — |

### Cao đẳng templates

| `danhSach` value | File | type | sheetMode | gradeField |
|---|---|---|---|---|
| Danh sách điểm danh | `Hệ CĐ/DS điểm danh.xlsx` | `diemDanh` | single | — |
| Danh sách điểm miệng | `Hệ CĐ/DS điểm miệng.xlsx` | `singleGrade` | multiByMon | `mieng` |
| Danh sách điểm thường xuyên | `Hệ CĐ/DS điểm thường xuyên.xlsx` | `singleGrade` | multiByMon | `thuongXuyen` |
| Danh sách điểm kiểm tra giữa học phần | `Hệ CĐ/DS điểm giữa HP.xlsx` | `singleGrade` | multiByMon | `giuaHP` |
| Danh sách điểm hết học phần | `Hệ CĐ/DS điểm hết HP.xlsx` | `singleGrade` | single | `hetHP` |
| Sổ điểm các môn | `Hệ CĐ/Sổ điểm các môn.xlsx` | `soDiem` | single | — |

> The Đại học template **does not include "Danh sách điểm miệng"** — there is no oral-grade component for university-level students. The frontend filter dropdown should match this when `he === 'Đại học'`.

## 2. Sheet mode: `single` vs `multiByMon`

- **`single`** — the export writes one sheet. All rows go there.
- **`multiByMon`** — the export writes one sheet *per `mon`*. The template is shipped with multiple sheets pre-formatted. Each subject is assigned to a 1-based sheet index:

```js
// from bieuMau.config.js
MON_SHEET_INDEX = {
  'Đại học': {
    'Đường lối quốc phòng và an ninh của ĐCSVN': 1,
    'Công tác Quốc phòng và An ninh': 2,
    'Quân sự chung': 3,
    'Kỹ thuật chiến đấu bộ binh và chiến thuật': 4
  },
  'Cao đẳng': {
    'Chính trị 1': 1,
    'Chính trị 2': 2,
    'Quân sự': 3
  }
}
```

When the frontend doesn't specify `mon`, a `multiByMon` export writes data into every sheet listed for the chosen hệ. When `mon` is supplied, only that sheet receives data.

## 3. The export request

```
GET /api/bieu-mau/export
  ?he=Đại học
  &danhSach=Danh sách điểm hết học phần
  &mon=Quân sự chung      ← required for some, optional for multiByMon defaults
  &khoa=<id>              ← optional filter
  &daiDoi=<id>            ← optional filter
  &truong=<id>            ← optional filter
```

The controller:
1. Reads `(he, danhSach)` and looks up the template in `TEMPLATE_MAP`.
2. Errors out (400) if the combination is unknown or `mon` is invalid for the hệ.
3. Loads the template file from `FORMS_ROOT + template.file` via `exceljs`.
4. Queries `SinhVien` (with `applyUnitScope` + `applyTeacherScope` applied), pulling `khoa`/`daiDoi`/`truong` populated names.
5. For `singleGrade` templates, reads the matching grade subdocument from each student's `diem[]` and writes a number to the cell.
6. For `soDiem` templates, writes the full grade detail (`thuongXuyen`, `giuaHP`, `hetHP`, etc.) into a structured block per student.
7. For `diemDanh`, writes the roster only (no scores).
8. Calls `workbook.xlsx.write(res)` directly — the response stream is the binary file.

### Headers

```
Content-Type: application/vnd.openxmlformats-officedocument.spreadsheetml.sheet
Content-Disposition: attachment; filename="<sanitized name>.xlsx"
```

### Validation gaps

The route has **no Joi schema** today. The controller enforces shape ad-hoc:
- Unknown `he` → 400 `INVALID_HE`
- Unknown `danhSach` → 400 `INVALID_TEMPLATE`
- `mon` not in `MON_SHEET_INDEX[he]` for multiByMon → 400 `INVALID_MON`
- Required params missing → 400 `MISSING_PARAMS`
- Template file missing on disk → 500 `TEMPLATE_ERROR`

If you add a new template, you must:
1. Drop the file in `backend/forms/Hệ {ĐH,CĐ}/`.
2. Add an entry to `TEMPLATE_MAP[he][danhSach]`.
3. If `sheetMode: 'multiByMon'`, add entries to `MON_SHEET_INDEX[he]` for every covered subject.
4. Update the frontend dropdown options on `BieuMauInAn.tsx`.

## 4. The student import

```
POST /api/sinh-vien/import   (multipart/form-data)
  file:  <.xls or .xlsx, max 20 MB>
  khoa:  <ObjectId>    ← optional; default for rows missing khoa
  truong: <ObjectId>   ← optional; default for rows missing truong
```

The controller:
1. Receives the multipart upload via multer, file lands in `backend/uploads/.tmp/`.
2. Opens it with **`xlsx`** (SheetJS).
3. Iterates over **every sheet** in the workbook. Each sheet's **name** is treated as the `đại đội` for the students inside it (the value is *not* read from a cell). A sheet named exactly `SỐ LƯỢNG` is skipped — it's a summary tab in the operator's template, not data.
4. Within each sheet: **row 5 is the header, data starts at row 6.** Columns read by position:

   | Column | Field | Required |
   |---|---|---|
   | B | `maSV` | yes |
   | C | `hoTen` | yes |
   | D | `ngaySinh` | yes (accepts Excel date cell, serial number, or `dd/mm/yyyy` / `dd-mm-yyyy` / `dd.mm.yyyy` / ISO) |
   | E | `lop` | no |
   | F | `nganh` | no |
   | G | `gioiTinh` | no (`"Nam"` → `true`, `"Nữ"` → `false`) |
   | H | `soDienThoai` | no |
   | I | `cccd` | no |
   | J | `ghiChu` | no |

5. For each row:
   - Resolves the sheet-name `daiDoi` → `DaiDoi._id` inside the chosen `khoa`. If no match, creates a new `DaiDoi` (and counts it in `createdDaiDoi`).
   - Inserts a new `SinhVien` with `trangThai: 'Đang học'` if `(maSV, hoTen, ngaySinh)` is not in the unique index; else counts as a duplicate.
   - Row-level errors are collected (not aborted).
6. Calls `fs.unlink(filePath)` at the end of the function body — **without a try/finally wrapper**. If an exception is thrown before that line (e.g., during `DaiDoi.create` or `SinhVien.insertMany`), the temp file is orphaned until the cleanup cron runs.

> **The import column layout is not the same as the export templates.** Operators have a separate template they hand to schools; that template has its own header row at row 5 and uses sheet-name-as-battalion. The `backend/forms/Hệ {ĐH,CĐ}/` templates are export-only.

### Response

```json
{
  "data": {
    "inserted": <number>,
    "duplicates": <number>,
    "createdDaiDoi": ["<ten of auto-created battalion>", ...],
    "errors": [
      { "row": <1-based row number>, "sheet": "...", "maSV": "...", "reason": "..." }
    ]
  }
}
```

This shape lets the UI summarize the run — current frontend prints `inserted + duplicates` + each error row.

## 5. The grade import

`POST /api/diem/import` is implemented. It imports grades for **one `(he, mon)` pair at a time** from a workbook produced by the operator's grade-book template. Required multipart form fields: `file`, `he` (`"Đại học"` or `"Cao đẳng"`), `mon` (must belong to the supplied `he`), `khoa` (cohort scope), `donViLienKet` (partner-school scope).

Behavior:

1. Multer accepts `.xls`/`.xlsx` up to 20 MB into `backend/uploads/.tmp/`.
2. The service iterates every worksheet. Sheet-name and row-7 inspection detect each sheet's own `he` — sheets that disagree with the request's `he` are skipped with a warning in `errors`.
3. Data rows start at **row 10**. Rows whose `maSV` (column B) is not digits-only are skipped (signature / footer rows).
4. Column positions depend on `he` (see `HE_LAYOUTS[he].cols` in the service): `thuongXuyen`, optional `mieng` (Cao đẳng), `giuaHP`, `hetHP`.
5. Students are matched by `maSV` against the set pre-loaded from `(khoa, donViLienKet)` intersected with the caller's `allowedUnits` / `teacherScope`. Out-of-scope `maSV`s land in `missingStudents`, not `inserted`/`updated`.
6. If a `Diem` subdocument for `mon` exists on the student, only supplied fields are overwritten (empty cells preserve the previous value). If not, a fresh `Diem` is created with the right defaults for the `he` and any supplied values overlaid. The `pre('validate')` hook then computes `tbMon`.
7. Per-row writes are batched in parallel chunks of 50, keeping a thousand-row import to a few seconds.

Response shape:

```json
{
  "data": {
    "inserted": <number>,
    "updated":  <number>,
    "skipped":  <number>,
    "errors":          [{ "sheet": "...", "row": <int>, "maSV": "...", "reason": "..." }],
    "missingStudents": [{ "sheet": "...", "row": <int>, "maSV": "...", "hoTen": "..." }]
  }
}
```

Validation errors that abort the whole call use the standard envelope: `400 BAD_HE`, `400 MON_HE_MISMATCH`, `400 MISSING_SCOPE`. See [`docs/api/endpoints/diem/post-import.md`](../api/endpoints/diem/post-import.md) for the full contract.

## 6. The `tbMon` arithmetic

The pre-validate hook in `backend/src/models/sinhVien.js:42-48` computes the subject average. Weight rules:

| Subject group | Weights |
|---|---|
| First 4 subjects in `MON_ENUM` (Đại học) | `thuongXuyen × 1 + giuaHP × 3 + hetHP × 6`, divided by 10 |
| Remaining 3 subjects (Cao đẳng) | `thuongXuyen × 1 + mieng × 1 + giuaHP × 2 + hetHP × 6`, divided by 10 |

To avoid IEEE-754 drift (e.g. `0.1 + 0.2 = 0.30000…04`), the hook multiplies each component by 10, does integer arithmetic, sums to hundredths, then divides back by 10:

```js
const toTenths = (v) => Math.round((v || 0) * 10);
let hundredths;
if (isFirst) hundredths = toTenths(tx) * 1 + toTenths(giua) * 3 + toTenths(het) * 6;
else         hundredths = toTenths(tx) * 1 + toTenths(mieng) * 1 + toTenths(giua) * 2 + toTenths(het) * 6;
return Math.round(hundredths / 10) / 10;
```

Result: `tbMon` is always a multiple of 0.1, never `7.499999…`. This matters when the frontend or an Excel export compares averages exactly.

When exporting a grade book, treat `tbMon` as truth, *not* as a value to recompute. The exporter should print `student.diem[i].tbMon` verbatim — recomputing in JavaScript will reintroduce the drift this hook was designed to prevent.

## 7. Operational notes

- **Templates are committed to git.** Updating a template (formatting fix, new column) is a git change. Use `git lfs` if templates grow beyond a few MB each — currently they're 10–50 KB.
- **`backend/forms/` is read-only.** The export pipeline opens templates in read mode and writes the output to the response stream; it never modifies the template on disk. If you ever need to regenerate templates, do it in a separate tool that writes to a different folder.
- **Multi-sheet workbooks are large.** A `multiByMon` Đại học workbook with all 4 sheets filled can be 100 KB+. Streaming via `workbook.xlsx.write(res)` keeps memory bounded; do not buffer into a string.
- **No background processing.** Exports run on the request thread. A grade-book export for a full khóa (~1000 students × 7 subjects) takes a few seconds; longer requests should be designed as paginated downloads or moved to a worker if usage grows.
- **The `khao-sat-chat-luong/` template files are actively used.** All three `phieu-…` files are loaded by `POST /api/khao-sat-chat-luong/process` via the `TEMPLATES` constant in `khaoSatChatLuongProcess.service.js`. They are not unimplemented placeholders.

## 8. See also

- `backend/src/config/bieuMau.config.js` — template map source of truth
- `backend/src/services/bieuMau.service.js` — export logic
- `backend/src/services/sinhVien.service.js` — import logic
- [`file-storage.md`](file-storage.md) — multer wiring and `backend/uploads/` layout
- [`database.md`](database.md) — `SinhVien.diem` embedded subdocument and `tbMon` field
- [`docs/api/endpoints/bieu-mau/get-export.md`](../api/endpoints/bieu-mau/get-export.md) — per-endpoint export contract
