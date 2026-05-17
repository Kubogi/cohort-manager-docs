# Excel Import — Students

**Triggered from**: [Cơ sở dữ liệu sinh viên](../frontend/pages/quan-ly-sinh-vien/co-so-du-lieu-sinh-vien.md) page, **Nhập từ Excel** button.

**Touches**: `POST /api/sinh-vien/import` (multipart upload, `xlsx`/SheetJS parse), `SinhVien`, `DaiDoi`.

**Who can do this**: `admin` only.

## Goal

Admin uploads a multi-sheet Excel roster received from a partner school. Each sheet's name identifies a `đại đội` (battalion); each row inside the sheet describes one student. The backend inserts new students under the chosen `khoa`, auto-creates any missing battalions, deduplicates against existing students, and reports per-row errors.

## Workbook expectations

| Aspect | Convention |
|---|---|
| **File format** | `.xls` or `.xlsx`, ≤ 20 MB. Other extensions are rejected by multer's `fileFilter`. |
| **Sheet ⇒ daiDoi** | Each sheet's name is the battalion name. A sheet literally named `SỐ LƯỢNG` is skipped (it's a summary tab in the operator's template). |
| **Header row** | Auto-detected. The parser scans from row 1 and treats the first row whose A/B/C cells are `STT` / `CCCD` / `Mã SV` (case-insensitive, whitespace trimmed) as the header. Sheets without a matching header are skipped silently. |
| **Data rows** | The row immediately after the detected header onward. |
| **Column layout** | Fixed by position: A=STT (skipped), B=`cccd`, C=`maSV`, D=`hoTen`, E=`ngaySinh`, F=`noiSinh`, G=`gioiTinh` (`"Nam"`/`"Nữ"`), H=`danToc`, I=`lop`, J=`nganh`, K=`soDienThoai`, L=`ngayNhapHoc`, M=`ngayVe`, N=`trangThai` (case-insensitive; falls back to `'Đang học'` if empty or unmatched), O=`ghiChu`. |
| **Date format** | `ngaySinh` accepts an Excel date cell, an Excel date-serial number, or text in `dd/mm/yyyy`, `dd-mm-yyyy`, `dd.mm.yyyy`, or ISO. |

The import column layout is **different** from the export templates in `backend/forms/Hệ {ĐH,CĐ}/`. Don't try to reuse a grade-book template as an import file; ask the operator for their import template.

## Happy path

1. **Admin opens the page**, clicks **Nhập từ Excel**.
2. **Picks the file**, selects `khoa` from a dropdown (required default for rows). Optionally picks `truong` (default `donViLienKet`).
3. **Submits.** Frontend sends `POST /api/sinh-vien/import` as `multipart/form-data` with `file`, `khoa`, `truong`.
4. **Backend (`sinhVien.controller.importData`):**
   - Multer drops the file into `backend/uploads/.tmp/`.
   - Calls `sinhVien.service.importFromExcel(path, { khoa, truong }, actor)`.
5. **Service:**
   - Opens the workbook with `xlsx` (SheetJS).
   - Iterates each sheet. Skips `SỐ LƯỢNG`. For each remaining sheet:
     a. Resolves sheet name → existing `DaiDoi` under the chosen `khoa`. If none exists, creates one and increments `createdDaiDoi`.
     b. Locates the header row (first row whose A/B/C cells are `STT` / `CCCD` / `Mã SV`, case-insensitive) and iterates the rows after it. For each row:
        - Reads columns B–O. Skips entirely empty rows.
        - Normalizes `gioiTinh` (`"Nam"` → `true`, `"Nữ"` → `false`).
        - Parses `ngaySinh`.
        - Looks up an existing student by `{ maSV, hoTen, ngaySinh }`. If found → counts as duplicate, skips.
        - Otherwise inserts a new `SinhVien` with `trangThai: "Đang học"` and the row's fields.
        - On any error (validation, type-coerce, etc.) — adds `{ row, message }` to `errors` and continues.
   - Attempts to delete the temp file from `.tmp/` at the end of the function body (not in a `finally` block). If the function throws early — e.g., on file-parse failure — the file is orphaned in `.tmp/` until the cleanup cron runs.
   - Returns `{ inserted, duplicates, createdDaiDoi, errors }`.
6. **Frontend** shows a result modal: "Imported X, skipped Y duplicates, created Z battalions, errors: …". User can re-import after fixing the errors.

## Side-effects

- New rows in `sinh_vien` (one per `inserted`).
- New rows in `dai_doi` (one per `createdDaiDoi`).
- **No** writes to `khoa`, `quyet_dinh`, `ho_so_suc_khoe`, `users`.
- The temp file is deleted on success and on failure. If the process crashes mid-import, an orphan file remains in `.tmp/`; the cron job in [`deployment.md`](../guides/deployment.md) cleans `.tmp/` files older than 60 minutes.

## Failure modes

| Scenario | What happens | Recovery |
|---|---|---|
| No `file` field | `400 { error: { message: "Vui lòng chọn file Excel" } }`. **Note** this uses the ad-hoc envelope, not the standard `{ error: { code, message } }`. | Re-submit with a file. |
| Wrong file extension | Multer rejects with `Chỉ chấp nhận file Excel (.xls, .xlsx)`. Reaches the global error handler as a 500. | Re-save as `.xlsx` and try again. |
| Workbook has a typo in a `daiDoi` sheet name | A new `DaiDoi` with the typo'd name is created. Students land under it. | Rename the daiDoi after import; or merge with the correct one and delete the typo'd row. |
| Row missing required column (e.g. no `maSV`) | Row contributes an `errors` entry; other rows still apply. | Fix the cell, re-import the same workbook — duplicates won't double-insert, only the previously-failed row will succeed. |
| Row's `(maSV, hoTen, ngaySinh)` already exists | Counts as `duplicates`. Existing student is **not** updated. | If you want to update an existing student, do it through the UI, not the importer. |
| Two rows in the same workbook have the same `(maSV, hoTen, ngaySinh)` | First wins on the unique index; second counts as `duplicates` on the same run. | Deduplicate the workbook before importing. |
| Workbook is huge (>10k students) | Imports run synchronously on the request thread. The HTTP request may time out at Nginx (default 60 s) or PM2 may flag the worker. | Split the workbook into chunks of ~2000 students per file. |

## Manual test recipe

- [ ] Prepare a small `.xlsx` with two sheets: `D1` (3 valid rows) and `SỐ LƯỢNG` (an extra summary sheet). Put the `STT / CCCD / Mã SV / …` header on any row you like — the parser auto-detects it; data follows on the next row.
- [ ] Open Cơ sở dữ liệu sinh viên → Nhập từ Excel → pick khoa K47, upload.
- [ ] Expected response: `inserted: 3, duplicates: 0, createdDaiDoi: 1, errors: []`.
- [ ] Reload the page; confirm 3 new students appear under daiDoi `D1` in `khoa K47`. `SỐ LƯỢNG` is absent.
- [ ] Re-upload the same file. Expected: `inserted: 0, duplicates: 3, createdDaiDoi: 0`.
- [ ] Edit one row's `ngaySinh` to an invalid string (`"not a date"`), upload again. Expected: 2 duplicates, 0 inserted, 0 new daiDoi, 1 error row.
- [ ] Confirm the temp file under `backend/uploads/.tmp/` is empty after each run.
