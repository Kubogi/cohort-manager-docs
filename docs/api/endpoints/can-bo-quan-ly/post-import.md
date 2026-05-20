# POST /api/can-bo-quan-ly/import

**Endpoint**: `POST /api/can-bo-quan-ly/import`
**Authentication**: ✅ Required
**Roles**: admin
**Content-Type**: `multipart/form-data`
**Last Verified**: 2026-05-19

---

## Description

Bulk-imports management staff (`CanBoQuanLy`) from the K181 Excel template. Each row inserts a CBQL (deduped by `hoTen`) **and** appends a `phanCong[]` entry pointing at the matched DaiDoi under the selected Khóa. **DaiDoi codes that don't yet exist under the chosen khóa are auto-created** (mirroring the sinhVien importer), so an admin can drop in a brand-new roster without pre-seeding battalions.

Admin-only.

---

## Request

### Headers
```
Authorization: Bearer <access_token>
Content-Type: multipart/form-data
```

### Form fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `file` | File | Yes | Excel file (`.xls` / `.xlsx`), max 20 MB |
| `khoa` | string (ObjectId) | **Yes** | The Khóa under which to insert phanCong entries and auto-create missing DaiDoi. `400 KHOA_REQUIRED` if omitted. |

---

## Excel format

### Sheet matching

The importer picks the first worksheet whose name, lowercased and stripped of whitespace, contains `mẫucbql` or `cbql`. Examples that match: `"Mẫu CB QL"`, `"CBQL Khóa 175"`, `"DS CBQL"`. Workbooks with no matching sheet return `400 NO_CBQL_SHEET`.

### Row structure

Header on **row 1**; data starts at **row 2**. **One row per CBQL** (flat layout — the old multi-row continuation pattern is gone). A row is treated as a CBQL when column B (`Họ và tên`) is non-empty; column A (`TT`) is a free-form serial number with no parsing significance.

### Column mapping (K181 template — `Mẫu CB QL` sheet)

| Col | Header | Maps to | Notes |
|-----|--------|---------|-------|
| A | `TT` | — | Free-form serial. Ignored. |
| B | `Họ và tên` | `CanBoQuanLy.hoTen` | Row marker. Empty → row skipped. |
| C | `Đại Đội` | `phanCong[].daiDoi` | Lowercased; looked up by `(ten, khoa)`. Auto-created if missing. **Rows whose value doesn't start with `c` are treated as junk and skipped entirely** (no CBQL is inserted for them) — this filters out section headers, typos, and stray cells. |
| D | `TK/PK` | `phanCong[].tkpk` | Free text — stored verbatim (e.g. `"Trưởng Khung Nhà D2"`). |
| E | `Số QĐ` | `phanCong[].soQD` | Free text. |
| F | `Ngày ra QĐ` | `phanCong[].ngayRaQD` | Excel-typed Date / serial number / `dd/mm/yyyy` text. |
| G | `Hiệu lực từ ngày` | `phanCong[].hieuLuc.batDau` | Same date parsing. |
| H | `Hiệu lực đến ngày` | `phanCong[].hieuLuc.ketThuc` | Same date parsing. |
| I | `Ghi chú` | `phanCong[].ghiChu` | Free text (max 500 chars). |

Fields **not in this template** (left empty on insert): `capBac`, `chucVu`, `donViQL`, `soDienThoai`. Edit them in the UI afterwards.

### Auto-create DaiDoi

When a row's `Đại Đội` value has no matching `DaiDoi` under the chosen Khóa, the importer creates one:

```js
DaiDoi.create({
  ten: code,
  khoa: <selectedKhoa>,
  donViLienKet: <inherited>   // from any existing DaiDoi under this khoa, falling back to any DaiDoi globally
});
```

A `donViLienKet` reference must exist somewhere in the database (the schema requires it). If the system has zero DaiDois at all, the importer throws `400 NO_DON_VI_AVAILABLE` — the admin must create at least one DonViLienKet + DaiDoi via the UI before re-importing.

Each auto-created code is reported once under `createdDaiDoi` in the response.

### Dedup

By `hoTen` only. A name already in the DB → counted under `duplicates`, no re-insert. Re-importing the same workbook is **idempotent**: existing CBQL with a phanCong entry for the chosen khóa are skipped (no duplicate phanCong appends).

---

## Response

### Success (200 OK)

```json
{
  "data": {
    "inserted": 9,
    "duplicates": 2,
    "attached": 11,
    "createdDaiDoi": ["c99"]
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `inserted` | number | New CBQL rows created. |
| `duplicates` | number | Rows skipped because `hoTen` already exists. |
| `attached` | number | `phanCong[]` entries appended this run. Includes both new and pre-existing CBQL whose name matched a row. |
| `createdDaiDoi` | string[] | Distinct daiDoi codes that were auto-created under the selected Khóa during this import. |

---

## Error responses

| Status | Code | Cause |
|--------|------|-------|
| 400 | `KHOA_REQUIRED` | The `khoa` form field is missing or invalid. |
| 400 | `KHOA_NOT_FOUND` | The supplied `khoa` ObjectId doesn't reference a real Khóa. |
| 400 | `NO_CBQL_SHEET` | No worksheet matched `mẫucbql` / `cbql`. |
| 400 | `NO_DON_VI_AVAILABLE` | An auto-create was needed but the system has no DonViLienKet to inherit from. |
| 400 | (no code) | No file in the request. |
| 403 | (auth) | Caller is not admin. |

---

## Notes

- The temp upload file is always cleaned up in a `finally` block, even when the service throws.
- After a successful import, the service re-runs `syncTeacherScope` for any User linked to a mutated CBQL — so teacher dashboards stay in sync with the new phanCong entries.
- The previous version of this endpoint expected the legacy `Danh sách GV.xlsx` layout (A=TT marker, B=name, C=capBac, D=chucVu, E=ghiChu, F=daiDoi code, multi-row continuations). That contract is **removed** as of 2026-05-19 — see commit history for migration notes.

---

## Related

- [POST /api/can-bo-quan-ly](./post.md)
- [GET /api/can-bo-quan-ly](./get-list.md)
- [CanBoQuanLy Schema](../../../backend/schemas/CanBoQuanLy.md)
- [Khóa học page](../../../frontend/pages/thong-tin-chung/khoa-hoc.md) — the UI that calls this endpoint
