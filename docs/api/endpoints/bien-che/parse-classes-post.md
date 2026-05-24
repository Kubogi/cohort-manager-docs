# POST /api/bien-che/parse-classes

**Endpoint**: `POST /api/bien-che/parse-classes`
**Authentication**: ✅ Required
**Roles**: admin, staff
**Unit Scoping**: ❌ Not applied (pure compute, no DB read/write)
**Last Verified**: 2026-05-23

---

## Description

Parses a user-uploaded Excel workbook and returns the class roster as `{ ten, soSinhVien }[]`. Iterates every sheet, locates the `Lớp` column by **header name** (so it tolerates differing column positions across templates), counts students per class, and sums any duplicates across sheets into a single row. No records are read from or written to the database — this is purely a parser feeding the [Biên chế đại đội tự động](../../../frontend/pages/quan-ly-sinh-vien/bien-che-dai-doi-tu-dong.md) planning tool.

---

## Request

### Headers
```
Authorization: Bearer <access_token>
Content-Type: multipart/form-data; boundary=…
```

### Body (multipart)

| Field | Type | Required | Description |
|---|---|---|---|
| `file` | file (`.xls` / `.xlsx`) | Yes | Max 20 MB. Rejected by multer if extension is anything else. |

---

## Response

### Success (200 OK)

```json
{
  "data": {
    "classes": [
      { "ten": "QH-2025-X-CTXH", "soSinhVien": 67 },
      { "ten": "QH-2025-X-LS",   "soSinhVien": 90 }
    ]
  }
}
```

- `classes` is sorted by `ten` ascending (deterministic).
- `soSinhVien` is a positive integer per row (zero/blank entries are not emitted).
- A `ten` that appears across multiple sheets is summed into a single row.

---

## Parsing rules

- **Sheet selection**: every sheet is inspected. A sheet is **skipped silently** if no cell in its first 30 rows matches the Lớp header pattern. This naturally excludes summary tabs like `SỐ LƯỢNG` and blank sheets.
- **Header detection**: a cell is considered the Lớp header if, after trimming + stripping diacritics + lowercasing, its content is `"lop"` (matches `"Lớp"`, `"lớp"`, `"LỚP"`, `" Lop "`, `"L"` is also accepted). Cells longer than 6 characters are ignored to avoid matching things like `"Họ và tên"`.
- **Header row**: the first row containing a Lớp header cell. The column index of that cell becomes the Lớp column for the rest of the sheet.
- **Data rows**: every row after the header row. Blank Lớp cells are skipped. Names are trimmed.
- **Cross-sheet aggregation**: a single `Map<ten, count>` accumulates across all sheets.

---

## Error Responses

### 400 `NO_FILE`
No file uploaded in the `file` field.

### 400 `INVALID_FILE`
The file exists but `XLSX.readFile` threw (corrupt / not a real spreadsheet).

### 400 `NO_CLASSES_FOUND`
The file parsed but no sheet had a Lớp header **and** at least one non-blank data row. Message: `Không tìm thấy lớp nào trong file. Kiểm tra cột "Lớp" có trong tiêu đề bảng.`

### 400 (multer)
File extension not `.xls` / `.xlsx`, or file exceeds 20 MB.

### 401 Unauthorized
Missing or invalid bearer token.

### 403 Forbidden
Role is not `admin` or `staff` (e.g. `viewer`, `teacher`).

---

## Implementation notes

- The uploaded file is written to `UPLOADS_ROOT/.tmp/` by multer, parsed, then unlinked in a `finally` so failures don't leak temp files.
- The parser itself is a pure function: `parseClassCountsXlsx(filePath)` in `backend/src/utils/parseClassCountsXlsx.js`. 12 unit tests cover header-by-name detection at columns E (K178 template) and I (coSoDuLieuSinhVien template), cross-sheet summation, summary-sheet skipping, header variants, and the empty-file 400.
- Once the response lands on the frontend, the [Biên chế đại đội tự động](../../../frontend/pages/quan-ly-sinh-vien/bien-che-dai-doi-tu-dong.md) page replaces the editable table with these rows. The next click of the **Biên chế** button posts them straight to [`POST /api/bien-che/partition`](./partition-post.md) — no DB write happens in either step.

## Related

- [POST /api/bien-che/partition](./partition-post.md)
- [Biên chế đại đội tự động page](../../../frontend/pages/quan-ly-sinh-vien/bien-che-dai-doi-tu-dong.md)
- Source: `backend/src/utils/parseClassCountsXlsx.js`, `backend/src/controllers/bienCheImport.controller.js`, `backend/src/routes/bienChe.route.js`
