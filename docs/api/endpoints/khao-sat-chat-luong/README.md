# /api/khao-sat-chat-luong

Per-year quality-survey configuration. Two sub-resources:

1. **`/nam`** — `KhaoSatChatLuongNam` rows. One per survey year, with the two center-level link fields (`linkGiangVienTuDanhGia`, `linkPhanHoiHoatDongTrungTam`).
2. **`/giang-vien`** — `GiangVienKhaoSat` rows. One per `(nam, hoTen)` instructor entry, with a `linkPhanHoiNguoiHoc` URL.

Role gating is **inside** the route file. Reads are typically `admin`/`staff`/`viewer`/`teacher`. Most write operations (`POST /nam`, `PATCH /nam/:id`, `POST /giang-vien`, `PATCH /giang-vien/:id`, `DELETE /giang-vien/:id`) are `admin`-only. `POST /process` is authorized for `['admin', 'staff']` — staff can also use it.

**Source files**: [khaoSatChatLuong.route.js](../../../../backend/src/routes/khaoSatChatLuong.route.js), [khaoSatChatLuong.controller.js](../../../../backend/src/controllers/khaoSatChatLuong.controller.js), [khaoSatChatLuong.service.js](../../../../backend/src/services/khaoSatChatLuong.service.js), [KhaoSatChatLuongNam schema](../../../backend/schemas/KhaoSatChatLuongNam.md), [GiangVienKhaoSat schema](../../../backend/schemas/GiangVienKhaoSat.md).

## `/nam` (per-year config)

### `GET /api/khao-sat-chat-luong/nam`

List configured years.

**Response**: `{ data: KhaoSatNam[], meta: { total: N } }`. Item shape: `{ _id, nam, linkGiangVienTuDanhGia, linkPhanHoiHoatDongTrungTam }`.

### `POST /api/khao-sat-chat-luong/nam`

Create a year row.

**Body**: `{ nam: number }`. Both link fields default to `""`. Unique on `nam` — duplicates return `409`.

**Response 201**: `{ data: KhaoSatNam }`.

### `PATCH /api/khao-sat-chat-luong/nam/:id`

Update either link field for a given year.

**Body**: `{ linkGiangVienTuDanhGia?: string, linkPhanHoiHoatDongTrungTam?: string }`.

**Response**: `{ data: KhaoSatNam }`.

> **No `DELETE`** is exposed. To "deactivate" a year, blank both URLs.

## `/giang-vien` (per-instructor feedback links)

### `GET /api/khao-sat-chat-luong/giang-vien`

List per-instructor entries.

**Query**: `nam` (optional — filter to one year).

**Response**: `{ data: KhaoSatGiangVien[], meta: { total: N } }`. Item shape: `{ _id, nam, hoTen, linkPhanHoiNguoiHoc }`.

### `POST /api/khao-sat-chat-luong/giang-vien`

Create an instructor entry.

**Body**: `{ nam: number, hoTen: string, linkPhanHoiNguoiHoc?: string }`. Unique on `(nam, hoTen)` — duplicates return `409`. Returns `404 YEAR_NOT_FOUND` if the referenced `nam` has no corresponding year row.

### `PATCH /api/khao-sat-chat-luong/giang-vien/:id`

Update name or URL.

**Body**: `{ hoTen?: string, linkPhanHoiNguoiHoc?: string }`.

### `DELETE /api/khao-sat-chat-luong/giang-vien/:id`

Remove an instructor entry.

**Response**: `204 No Content`.

## Survey-result ingestion

### `POST /api/khao-sat-chat-luong/process`

**Roles**: route-level `admin`, `staff`. With `nam` provided, additionally **admin-only** at the service layer (consistent with the admin-only KhaoSat ownerTypes).

Multipart upload of a survey-result spreadsheet exported from Google Forms (or a compatible source). The controller processes the uploaded file and streams back a **binary Excel workbook** — it does NOT return JSON and does NOT write aggregate scores into `GiangVienKhaoSat` rows.

When `nam` is provided, the endpoint additionally **persists two Attachment rows** against that year — the uploaded source file and the generated report — keyed by the per-tab ownerType pair (see [Attachment schema](../../../backend/schemas/Attachment.md#ownertype--backing-model)). Re-uploading for the same `(nam, type)` replaces both rows and unlinks the old disk files.

**Request**: `multipart/form-data` with the following fields:

| Field | Type | Required | Description |
|---|---|---|---|
| `file` | file | yes | `.xls` or `.xlsx`, max 20 MB. Multer rejects other extensions before the controller runs. |
| `type` | string | yes | Processing path selector. Must be one of `gvTuDanhGia` \| `phanHoiGV` \| `phanHoiTrungTam`. An invalid or missing value returns `400 INVALID_TYPE`. |
| `totalInstructors` | number | no | Used only by the `gvTuDanhGia` processing path. |
| `nam` | ObjectId | no | `_id` of a `KhaoSatChatLuongNam` row. When present, the source + generated report are persisted as Attachments against that year; admin-only when present. Omitting it preserves the pre-archive behavior (no persistence, staff still allowed). |

**Response**: `200` with `Content-Type: application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`. The response body is the generated Excel workbook — callers should save it as a `.xlsx` file. Processing statistics are returned in the custom response header `X-KhaoSat-Stats` (URL-encoded JSON).

**Errors when `nam` is provided**:
- `400 BAD_NAM` — not a valid ObjectId
- `400 INVALID_TYPE` — `type` is not one of the three accepted values
- `400 UNBUCKETED_TEACHER_SHEETS` — **v2026 phanHoiGV only**; one or more teacher sheets in the upload have no recognized tab color (see *Per-năm pipeline branches* below)
- `403 FORBIDDEN` — caller is not admin
- `404 YEAR_NOT_FOUND` — no `KhaoSatChatLuongNam` row with that id

**Per-năm pipeline branches**

The service dispatches on the integer `nam.nam` value read from the year doc:

- **`Number(nam.nam) === 2026`** → v2026 pipeline (different template + handler per tab).
- **Anything else (including the no-`nam` legacy path)** → v2024 pipeline (the original handler set).

Tab-by-tab behavior:

| Tab | v2024 output | v2026 output |
|---|---|---|
| `gvTuDanhGia` | one sheet `Biểu mẫu câu trả lời` with 25 questions | three sheets — `2025-2026` (29 questions), `Khoa QS` (per-teacher matrix for Quân sự), `Khoa CT` (per-teacher matrix for Chính trị). Khoa bucketing reads source col E ("4. Khoa") |
| `phanHoiGV` | per-teacher sheets only (10 questions each), upload must have `Gốc.*` sheet prefix | per-teacher sheets (16 questions each) + 3 aggregates (`KQ Khoa QS`, `KQ Khoa CT`, `Toàn TT`); upload sheets may carry the `Gốc.` prefix OR be bare-named; **teacher→khoa is encoded by sheet tab color** (see below) |
| `phanHoiTrungTam` | one sheet `Thống kê kết quả trả lời` with 26 questions | reuses the v2024 handler unchanged |

**Tab-color khoa convention (v2026 phanHoiGV uploads)**

Each per-teacher sheet in the upload must carry one of three explicit ARGB tab colors. The admin sets these via Excel → right-click tab → **Tab Color → More Colors → Custom** (Excel's "Theme Colors" picker is rejected — only explicit ARGB is read):

| Tab color | Bucket | Effect |
|---|---|---|
| `FF002060` (navy) | Khoa Quân sự (QS) | Teacher report emitted + counts flow into `KQ Khoa QS` aggregate |
| `FF00B050` (green) | Khoa Chính trị (CT) | Teacher report emitted + counts flow into `KQ Khoa CT` aggregate |
| `FFFF0000` (red) | EXCLUDE | Sheet skipped silently; name listed in `stats.excluded[]` |
| anything else / no color | (unbucketed) | **Upload rejected with `400 UNBUCKETED_TEACHER_SHEETS`** listing the offending sheet names |

The denylist `NON_TEACHER_SHEET_NAMES` (in `khaoSatChatLuong/shared.js`) excludes known scratch/aggregate sheet names (`KQ Khoa QS`, `Data gốc`, `Sheet1`, `Tính theo % TT`, etc.) so a previously-generated report can be re-uploaded without false positives.

**Source**: [khaoSatChatLuong.route.js](../../../../backend/src/routes/khaoSatChatLuong.route.js) (line 40), [khaoSatChatLuong/index.js](../../../../backend/src/services/khaoSatChatLuong/index.js) (dispatcher), [khaoSatChatLuong/v2024.*](../../../../backend/src/services/khaoSatChatLuong/), [khaoSatChatLuong/v2026.*](../../../../backend/src/services/khaoSatChatLuong/), [khaoSatChatLuongProcess.service.js](../../../../backend/src/services/khaoSatChatLuongProcess.service.js) (thin re-export shim).

**Notes**:
- The temp file is written to `backend/uploads/.tmp/` and deleted in a `finally` block. When `nam` is provided, the source is also copied into `backend/uploads/attachments/<KhaoSat*Source>/` before the temp is unlinked.
- Expected sheet shape is owned by the processing service; consult `khaoSatChatLuongProcess.service.js` for the column map before exporting your form template.
- The frontend ketQuaKhaoSat page (admin/staff/viewer) is the production caller — it sends `nam` whenever the admin has picked a year, which is required to enable the upload button.

## Cross-cutting notes

- **Instructor names are *not* linked to `CanBoQuanLy`.** This is a separate, plain-string per-survey roster. Renaming a `CanBoQuanLy` doesn't propagate; renaming a `KhaoSatGiangVien` doesn't touch `CanBoQuanLy`.
- **Year is an integer.** Don't send `"2025"`.
- **The `giangVien` collection name collides visually with the frontend's "Giảng viên" page.** See the [GiangVienKhaoSat schema](../../../backend/schemas/GiangVienKhaoSat.md) for the naming history.

## Related

- [KhaoSatChatLuongNam schema](../../../backend/schemas/KhaoSatChatLuongNam.md)
- [GiangVienKhaoSat schema](../../../backend/schemas/GiangVienKhaoSat.md)
- [`docs/frontend/pages/quan-ly-tai-khoan/quan-ly-link.md`](../../../frontend/pages/quan-ly-tai-khoan/quan-ly-link.md) — admin UI (sections 2 + 3)
- [`docs/frontend/pages/khao-sat-chat-luong/`](../../../frontend/pages/khao-sat-chat-luong/) — consumer pages
