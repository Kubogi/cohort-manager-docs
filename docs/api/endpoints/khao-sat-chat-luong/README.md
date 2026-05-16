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

**Roles**: `admin`, `staff`.

Multipart upload of a survey-result spreadsheet exported from Google Forms (or a compatible source). The controller processes the uploaded file and streams back a **binary Excel workbook** — it does NOT return JSON and does NOT write aggregate scores into `GiangVienKhaoSat` rows.

**Request**: `multipart/form-data` with the following fields:

| Field | Type | Required | Description |
|---|---|---|---|
| `file` | file | yes | `.xls` or `.xlsx`, max 20 MB. Multer rejects other extensions before the controller runs. |
| `type` | string | yes | Processing path selector. Must be one of `gvTuDanhGia` \| `phanHoiGV` \| `phanHoiTrungTam`. An invalid or missing value returns `400 INVALID_TYPE`. |
| `totalInstructors` | number | no | Used only by the `gvTuDanhGia` processing path. |

**Response**: `200` with `Content-Type: application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`. The response body is the generated Excel workbook — callers should save it as a `.xlsx` file. Processing statistics are returned in the custom response header `X-KhaoSat-Stats` (URL-encoded JSON).

**Source**: [khaoSatChatLuong.route.js](../../../../backend/src/routes/khaoSatChatLuong.route.js) (line 40), [khaoSatChatLuongProcess.service.js](../../../../backend/src/services/khaoSatChatLuongProcess.service.js).

**Notes**:
- The temp file is written to `backend/uploads/.tmp/` and deleted in a `finally` block.
- Expected sheet shape is owned by the processing service; consult `khaoSatChatLuongProcess.service.js` for the column map before exporting your form template.
- This endpoint is **not** wired into a frontend page today — it is invoked manually via cURL/Postman during a survey cycle.

## Cross-cutting notes

- **Instructor names are *not* linked to `CanBoQuanLy`.** This is a separate, plain-string per-survey roster. Renaming a `CanBoQuanLy` doesn't propagate; renaming a `KhaoSatGiangVien` doesn't touch `CanBoQuanLy`.
- **Year is an integer.** Don't send `"2025"`.
- **The `giangVien` collection name collides visually with the frontend's "Giảng viên" page.** See the [GiangVienKhaoSat schema](../../../backend/schemas/GiangVienKhaoSat.md) for the naming history.

## Related

- [KhaoSatChatLuongNam schema](../../../backend/schemas/KhaoSatChatLuongNam.md)
- [GiangVienKhaoSat schema](../../../backend/schemas/GiangVienKhaoSat.md)
- [`docs/frontend/pages/quan-ly-tai-khoan/quan-ly-link.md`](../../../frontend/pages/quan-ly-tai-khoan/quan-ly-link.md) — admin UI (sections 2 + 3)
- [`docs/frontend/pages/khao-sat-chat-luong/`](../../../frontend/pages/khao-sat-chat-luong/) — consumer pages
