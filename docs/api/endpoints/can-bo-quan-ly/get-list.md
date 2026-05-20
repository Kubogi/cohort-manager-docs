# GET /api/can-bo-quan-ly

**Endpoint**: `GET /api/can-bo-quan-ly`
**Authentication**: ✅ Required
**Roles**: admin, staff, viewer
**Unit Scoping**: ❌ Not applied — `CanBoQuanLy` has no top-level `khoa` field; permitted roles see every record (but the `?khoa=` filter does scope by `phanCong[].khoa`)
**Last Verified**: 2026-05-19

---

## Description

Lists management staff (cán bộ quản lý) with pagination. Each record carries its `phanCong[]` — per-khoa assignments holding `soQD`, `ngayRaQD`, `hieuLuc`, `tkpk`, `ghiChu`, and optional `daiDoi`.

---

## Request

### Headers
```
Authorization: Bearer <access_token>
```

### Query Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| page | number | 1 | Page number (1-indexed) |
| limit | number | 20 | Items per page |
| hoTen | string | — | Case-insensitive substring filter on full name. Metachars are escaped before regex use. |
| donVi | string | — | Case-insensitive substring filter on `donViQL`. |
| chucVu | string | — | Case-insensitive substring filter on position/role. |
| khoa | ObjectId | — | Returns only CBQL with at least one `phanCong` entry for that khoa. |
| unattached | `true` | — | Combined with `khoa`: returns only CBQL whose `phanCong` entry for that khoa has `daiDoi: null` (khoa-only, no battalion). |

All filter params combine with AND semantics. Empty strings are ignored.

### Default sort

Results are sorted by `{ hoTen: 1, _id: 1 }`.

---

## Response

### Success (200 OK)

```json
{
  "data": [
    {
      "_id": "507f1f77bcf86cd799439011",
      "hoTen": "Nguyễn Văn A",
      "capBac": "Thiếu tá",
      "donViQL": "Bộ Quốc phòng",
      "chucVu": "Trưởng khoa",
      "soDienThoai": "0912345678",
      "ghiChu": "",
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
      "createdAt": "2023-01-15T00:00:00.000Z",
      "updatedAt": "2023-01-15T00:00:00.000Z"
    }
  ],
  "meta": {
    "page": 1,
    "limit": 20,
    "total": 35
  }
}
```

---

## Related

- [GET /api/can-bo-quan-ly/:id](./get-one.md)
- [POST /api/can-bo-quan-ly](./post.md)
- [CanBoQuanLy Schema](../../../backend/schemas/CanBoQuanLy.md)
