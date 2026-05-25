# GET /api/quyet-dinh

**Endpoint**: `GET /api/quyet-dinh`  

**Authentication**: ✅ Required  

**Roles**: admin, staff, viewer, teacher

**Unit Scoping**: ✅ Applied (staff: `allowedUnits`; teacher: `teacherScope`)

**Last Verified**: 2026-05-25

---

## Description

Lists decisions (quyết định) with pagination and filtering. Unit scoping is applied via the referenced student's unit membership.

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
| khoa | string | - | Filter by Khoa ID. Resolved through `SinhVien.khoa` → `QuyetDinh.sinhVien $in` (the `QuyetDinh.khoa` denormalized field is not populated on create, so a direct field filter would return zero rows). |
| daiDoi | string | - | Filter by DaiDoi ID. Same indirection as `khoa` — resolved through `SinhVien.daiDoi`. |
| type | string | - | Filter by decision type (maps to loaiQD field) |
| status | string | - | Filter by status (maps to trangThai field) |
| from | string | - | Filter by start date (ngayKiQD >= from, ISO date) |
| to | string | - | Filter by end date (ngayKiQD <= to, ISO date) |

---

## Response

### Success (200 OK)

```json
{
  "data": [
    {
      "_id": "507f1f77bcf86cd799439011",
      "sinhVien": "64a1b2c3d4e5f6a7b8c9d0e1",
      "soQD": "QD-123/2023",
      "ngayKiQD": "2023-10-15T00:00:00.000Z",
      "loaiQD": "Khen thưởng",
      "lyDo": "Hoàn thành xuất sắc nhiệm vụ",
      "khoa": "64a1b2c3d4e5f6a7b8c9d0e2",
      "daiDoi": "64a1b2c3d4e5f6a7b8c9d0e3",
      "trangThai": "Chưa xử lý",
      "attachments": [],
      "createdAt": "2023-10-15T00:00:00.000Z",
      "updatedAt": "2023-10-15T00:00:00.000Z"
    }
  ],
  "meta": {
    "page": 1,
    "limit": 20,
    "total": 45
  }
}
```

---

## Notes

- The `attachments` array is added to each item by `decorateWithAttachments()` (populated from the `Attachment` collection). It is **only present on list responses** — the single-record `GET /:id` endpoint does not include it.
- There is no `file` field on `QuyetDinh` documents. Attachments are managed separately via the Attachments API.
- `trangThai` values: `"Chưa xử lý"` and `"Đã xử lý"`.

## Unit Scoping

Staff users are filtered by `allowedUnits` scoped to the referenced student's unit. Teachers are filtered by `teacherScope`. Viewer and admin see all records their route allows.

---

## Related

- [GET /api/quyet-dinh/:id](./get-one.md)
- [POST /api/quyet-dinh](./post.md)
- [QuyetDinh Schema](../../../backend/schemas/QuyetDinh.md)
