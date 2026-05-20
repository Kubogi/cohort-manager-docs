# PATCH /api/can-bo-quan-ly/:id

**Endpoint**: `PATCH /api/can-bo-quan-ly/:id`
**Authentication**: ✅ Required
**Roles**: admin
**HTTP Method**: **PATCH** (not PUT)
**Last Verified**: 2026-05-19

---

## Description

Updates an existing management staff member. **Admin-only operation**. Partial updates supported.

When `phanCong` is included in the body, it **replaces** the entire array (Mongoose `$set`). The khoaHoc frontend reads the full current `phanCong[]`, mutates the entry for `selectedKhoa`, and sends back the full array — that way it never clobbers entries belonging to other khoa.

---

## Request

### Headers
```
Content-Type: application/json
Authorization: Bearer <access_token>
```

### URL Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| id | string | Yes | CanBoQuanLy MongoDB ObjectId |

### Body

Send only the fields you want to update. To rewrite a phanCong entry:

```json
{
  "phanCong": [
    {
      "khoa": "64a1b2c3d4e5f6a7b8c9d0e1",
      "daiDoi": null,
      "soQD": "QĐ-002/2024",
      "ngayRaQD": "2024-06-15",
      "hieuLuc": { "batDau": "2024-06-15", "ketThuc": "2024-12-31" },
      "tkpk": "Phó khung",
      "ghiChu": "Mới gán"
    }
  ]
}
```

**Updatable top-level fields**: `hoTen`, `capBac`, `donViQL`, `chucVu`, `soDienThoai`, `ghiChu`, `taiKhoan`, `phanCong`.

`phanCong[]` validation rules are identical to POST: per-khoa uniqueness + cross-khoa daiDoi consistency.

---

## Response

### Success (200 OK)

```json
{
  "data": {
    "_id": "507f1f77bcf86cd799439011",
    "hoTen": "Nguyễn Văn A",
    "phanCong": [
      {
        "_id": "60a1b2c3d4e5f6a7b8c9d0f1",
        "khoa": "64a1b2c3d4e5f6a7b8c9d0e1",
        "daiDoi": null,
        "soQD": "QĐ-002/2024",
        "ngayRaQD": "2024-06-15T00:00:00.000Z",
        "hieuLuc": {
          "batDau": "2024-06-15T00:00:00.000Z",
          "ketThuc": "2024-12-31T00:00:00.000Z"
        },
        "tkpk": "Phó khung",
        "ghiChu": "Mới gán"
      }
    ],
    "updatedAt": "2025-12-31T10:00:00.000Z"
  }
}
```

---

## Error Responses

### 404 Not Found
Staff member doesn't exist.

### 403 Forbidden
Only admin role can update staff.

### 400 Bad Request — VALIDATION_ERROR | DUPLICATE_KHOA_IN_PHANCONG | KHOA_DAIDOI_MISMATCH

### 409 Conflict — DUPLICATE_TAIKHOAN

---

## Related

- [GET /api/can-bo-quan-ly/:id](./get-one.md)
- [DELETE /api/can-bo-quan-ly/:id](./delete.md)
- [CanBoQuanLy Schema](../../../backend/schemas/CanBoQuanLy.md)
