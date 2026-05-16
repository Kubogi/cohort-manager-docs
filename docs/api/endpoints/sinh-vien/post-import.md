# POST /api/sinh-vien/import

**Endpoint**: `POST /api/sinh-vien/import`

**Authentication**: ✅ Required (Bearer)

**Roles**: `admin`

**Source**: [sinhVien.route.js](../../../../backend/src/routes/sinhVien.route.js), [sinhVien.controller.js](../../../../backend/src/controllers/sinhVien.controller.js), [sinhVien.service.js](../../../../backend/src/services/sinhVien.service.js)

**Last verified**: 2026-05-16

---

## Description

Imports a list of students from an Excel file. The backend parses the file with the **`xlsx` (SheetJS)** library, resolves each **sheet name** to an existing or newly-created battalion inside the supplied `khoa`, inserts new students, and reports per-row errors.

The workbook may contain multiple sheets — each sheet's name is treated as a `đại đội` (battalion). A sheet literally named `SỐ LƯỢNG` is skipped. Within each sheet, **row 5 is the header** and data begins on row 6. The expected column layout matches the import templates the operator uses; it is **not** the same layout as the export templates in `backend/forms/Hệ {ĐH,CĐ}/`.

---

## Request

### Headers

```
Content-Type: multipart/form-data
Authorization: Bearer <accessToken>
```

### Body (multipart fields)

| Field | Type | Required | Description |
|---|---|---|---|
| `file` | File (`.xls` or `.xlsx`) | Yes | The roster file. Max 20 MB; other extensions rejected at multer's `fileFilter`. |
| `khoa` | ObjectId string | No | Default `khoa` (faculty/cohort) for rows that don't carry their own. If a row references a `daiDoi` that doesn't exist under this `khoa`, the battalion is created automatically. |
| `truong` | ObjectId string | No | Default `truong` (đơn vị liên kết) for rows that don't carry their own. |

### Column expectations (per sheet; row 5 = header, data from row 6)

The `daiDoi` is taken from the sheet **name**, not a column. Columns read from each data row:

| Column | Field | Required | Notes |
|---|---|---|---|
| A | (STT) | no | Sequence number; ignored by the parser. |
| B | `cccd` | no | Citizen-ID. |
| C | `maSV` | yes | |
| D | `hoTen` | yes | |
| E | `ngaySinh` | yes | Accepts an Excel date cell, an Excel date-serial number, or text in `dd/mm/yyyy`, `dd-mm-yyyy`, `dd.mm.yyyy`, or ISO format. |
| F | `lop` | no | |
| G | `nganh` | no | |
| H | `noiSinh` | no | Place of birth. |
| I | `gioiTinh` | no | The Vietnamese strings `"Nam"` → `true` and `"Nữ"` → `false`. Anything else is unset. |
| J | `danToc` | no | Ethnicity. |
| K | `soDienThoai` | no | |
| L | `trangThai` | no | Matched case-insensitively against the enum (`'Đang học'`, `'Hoãn học'`, `'Thôi học'`, `'Đình chỉ'`, `'Miễn học'`, `'Không tham gia học'`). Empty or unmatched values default to `'Đang học'`. |
| M | `ghiChu` | no | |

If column L is empty or contains an unrecognized value, the inserted student receives `trangThai: 'Đang học'`.

---

## Response

### 200 OK

```json
{
  "data": {
    "inserted": 42,
    "duplicates": 3,
    "createdDaiDoi": ["d1", "d2"],
    "errors": [
      { "row": 17, "sheet": "D1", "maSV": "SV001", "reason": "Thiếu/không hợp lệ: Ngày sinh không hợp lệ (\"31/02/2024\")" },
      { "row": 21, "sheet": "D1", "maSV": "SV007", "reason": "Thiếu/không hợp lệ: Họ tên" }
    ]
  }
}
```

| Field | Type | Meaning |
|---|---|---|
| `inserted` | number | New `SinhVien` documents written. |
| `duplicates` | number | Rows that matched an existing student by `(maSV, hoTen, ngaySinh)` and were skipped (detected via the sparse unique index). |
| `createdDaiDoi` | string[] | Lowercased battalion names auto-created under the chosen `khoa` because they didn't exist yet. |
| `errors` | object[] | Per-row errors. Each entry has `row` (1-indexed Excel row, `0` for Mongo-level failures with no row context), `sheet`, `maSV`, and a Vietnamese `reason`. Other rows still commit — this is **not** an atomic batch. |

### 400 Bad Request — no file

```json
{ "error": { "message": "Vui lòng chọn file Excel" } }
```

(Note: emitted ad-hoc by the controller; does not use the standard `{ error: { code, message } }` envelope. Frontend reads `error.message`.)

### 400 Bad Request — wrong file type

Multer's `fileFilter` rejects non-Excel files with the message `Chỉ chấp nhận file Excel (.xls, .xlsx)`. The error reaches the global `errorHandler` and is returned as a 500 with that message. Treat as user error in the UI.

### 401 / 403

Standard auth envelope. Non-admin callers get 403 `FORBIDDEN`.

---

## Behavior notes

1. **Atomicity** — none. Rows are processed sequentially; errors collect, valid rows commit. There is no transactional rollback.
2. **Duplicates** — detected by the `{ maSV, hoTen, ngaySinh }` sparse unique index. Existing students are **not** updated; they're counted in `duplicates` and skipped.
3. **`daiDoi` auto-creation** — if a row's `daiDoi` cell holds a name that doesn't exist under the chosen `khoa`, the service creates a fresh `DaiDoi` with `{ ten: <name>, khoa: <khoa> }` and uses it for that row. This makes onboarding new battalions painless but can also produce typo-driven duplicates — confirm `createdDaiDoi` matches your intent after each import.
4. **Temp file cleanup** — multer drops the upload into `backend/uploads/.tmp/`; the service attempts to delete it at the end of a successful or partially-successful run (`fs.unlink` at the end of the function body, not in a `finally` block). If the function throws early — e.g., on file-parse failure or battalion creation failure — the unlink is never reached and the temp file is **orphaned** in `.tmp/`. See [`docs/guides/backups.md`](../../../guides/backups.md) for the cron that cleans up stale `.tmp/` files.
5. **Per-unit scoping** — the import does **not** consult `req.user.allowedUnits` (the route is admin-only and admin sees everything). Adding finer-grained importer roles would require service-level scoping additions.

---

## Example

```bash
curl -X POST http://localhost:3000/api/sinh-vien/import \
  -H "Authorization: Bearer $TOKEN" \
  -F "file=@./rosters/k47.xlsx" \
  -F "khoa=64a1b2c3d4e5f6a7b8c9d0e1" \
  -F "truong=64a1b2c3d4e5f6a7b8c9d0f2"
```

---

## Related

- [POST /api/sinh-vien](./post.md) — create a single student
- [GET /api/sinh-vien/export/all](./get-export.md) — the export side (still a stub)
- [`docs/architecture/excel-pipeline.md`](../../../architecture/excel-pipeline.md) — column-by-column template reference
- [`docs/workflows/excel-import-students.md`](../../../workflows/excel-import-students.md) — end-to-end import walkthrough
- [SinhVien schema](../../../backend/schemas/SinhVien.md)
