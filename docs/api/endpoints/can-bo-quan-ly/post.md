# POST /api/can-bo-quan-ly

**Endpoint**: `POST /api/can-bo-quan-ly`
**Authentication**: ✅ Required
**Roles**: admin
**Last Verified**: 2026-05-19

---

## Description

Creates a new management staff member. **Admin-only operation**.

The legacy top-level `soQD`, `ngayRaQD`, `hieuLuc` fields are no longer part of the schema — those values now live per-khoa under `phanCong[]`.

---

## Request

### Headers
```
Content-Type: application/json
Authorization: Bearer <access_token>
```

### Body

```json
{
  "hoTen": "Nguyễn Văn A",
  "capBac": "Thiếu tá",
  "donViQL": "Bộ Quốc phòng",
  "chucVu": "Trưởng khoa",
  "soDienThoai": "0912345678",
  "ghiChu": "Notes about the person (not the assignment)",
  "phanCong": [
    {
      "khoa": "64a1b2c3d4e5f6a7b8c9d0e1",
      "daiDoi": "64a1b2c3d4e5f6a7b8c9d0e2",
      "soQD": "QĐ-001/2024",
      "ngayRaQD": "2024-01-15",
      "hieuLuc": { "batDau": "2024-01-15", "ketThuc": "2024-12-31" },
      "tkpk": "Trưởng khung",
      "ghiChu": "Phụ trách khung D2"
    }
  ],
  "taiKhoan": "64a1b2c3d4e5f6a7b8c9d0e1"
}
```

### Required Fields

- `hoTen` (string, max 200 chars) — Full name

### Optional Fields

- `capBac` / `donViQL` / `chucVu` / `soDienThoai` / `ghiChu` (string)
- `taiKhoan` (24-char hex ObjectId) — link to an existing `User`
- `phanCong[]` — per-khoa assignments. Each entry:
  - `khoa` (24-char hex ObjectId) — **required** within the entry
  - `daiDoi` (24-char hex ObjectId | `null`) — optional; if set, the DaiDoi's khoa must equal this entry's `khoa` (otherwise `400 KHOA_DAIDOI_MISMATCH`)
  - `soQD` (string, max 100)
  - `ngayRaQD` (ISO date | `null`)
  - `hieuLuc` (`{ batDau, ketThuc }` of ISO dates | `null`)
  - `tkpk` (`""` | `"Trưởng khung"` | `"Phó khung"`)
  - `ghiChu` (string, max 500)

A CBQL may have **at most one** `phanCong` entry per khoa. Duplicates are rejected with `400 DUPLICATE_KHOA_IN_PHANCONG`.

---

## Response

### Success (201 Created)

```json
{
  "data": {
    "_id": "507f1f77bcf86cd799439011",
    "hoTen": "Nguyễn Văn A",
    "capBac": "Thiếu tá",
    "donViQL": "Bộ Quốc phòng",
    "chucVu": "Trưởng khoa",
    "soDienThoai": "0912345678",
    "ghiChu": "Notes about the person",
    "phanCong": [
      {
        "_id": "60a1b2c3d4e5f6a7b8c9d0f1",
        "khoa": "64a1b2c3d4e5f6a7b8c9d0e1",
        "daiDoi": "64a1b2c3d4e5f6a7b8c9d0e2",
        "soQD": "QĐ-001/2024",
        "ngayRaQD": "2024-01-15T00:00:00.000Z",
        "hieuLuc": {
          "batDau": "2024-01-15T00:00:00.000Z",
          "ketThuc": "2024-12-31T00:00:00.000Z"
        },
        "tkpk": "Trưởng khung",
        "ghiChu": "Phụ trách khung D2"
      }
    ],
    "taiKhoan": "64a1b2c3d4e5f6a7b8c9d0e1",
    "createdAt": "2025-12-31T10:00:00.000Z",
    "updatedAt": "2025-12-31T10:00:00.000Z"
  }
}
```

---

## Error Responses

### 403 Forbidden
Only admin role can create staff.

### 400 Bad Request — VALIDATION_ERROR
Missing required field `hoTen`, or invalid `phanCong` entry.

### 400 Bad Request — DUPLICATE_KHOA_IN_PHANCONG
The `phanCong[]` array contains two entries with the same `khoa`.

### 400 Bad Request — KHOA_DAIDOI_MISMATCH
A `phanCong` entry has `daiDoi` set, but the `DaiDoi` belongs to a different khoa.

### 409 Conflict — DUPLICATE_CAN_BO
A staff member with the same `hoTen` already exists.

### 409 Conflict — DUPLICATE_TAIKHOAN
The `taiKhoan` ObjectId is already assigned to another staff member.

---

## Related

- [PATCH /api/can-bo-quan-ly/:id](./patch.md)
- [GET /api/can-bo-quan-ly](./get-list.md)
- [CanBoQuanLy Schema](../../../backend/schemas/CanBoQuanLy.md)
