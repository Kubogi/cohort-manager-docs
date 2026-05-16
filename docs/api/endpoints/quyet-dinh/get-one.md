# GET /api/quyet-dinh/:id

**Endpoint**: `GET /api/quyet-dinh/:id`  

**Authentication**: ✅ Required  

**Roles**: admin, staff, viewer, teacher

**Unit Scoping**: ✅ Applied (staff: `allowedUnits`; teacher: `teacherScope`)  

**Last Verified**: 2026-05-16

---

## Description

Retrieves a single decision by ID. **Unit scoping automatically applied**.

---

## Request

### Headers
```
Authorization: Bearer <access_token>
```

### URL Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| id | string | Yes | QuyetDinh MongoDB ObjectId |

---

## Response

### Success (200 OK)

```json
{
  "data": {
    "_id": "507f1f77bcf86cd799439011",
    "sinhVien": "64a1b2c3d4e5f6a7b8c9d0e1",
    "soQD": "QD-123/2023",
    "ngayKiQD": "2023-10-15T00:00:00.000Z",
    "loaiQD": "Khen thưởng",
    "lyDo": "Hoàn thành xuất sắc nhiệm vụ",
    "khoa": "64a1b2c3d4e5f6a7b8c9d0e2",
    "daiDoi": "64a1b2c3d4e5f6a7b8c9d0e3",
    "trangThai": "Chưa xử lý",
    "createdAt": "2023-10-15T00:00:00.000Z",
    "updatedAt": "2023-10-15T00:00:00.000Z"
  }
}
```

---

## Notes

- `attachments` is **not** included in single-record responses. Only `GET /api/quyet-dinh` (list) decorates items with `attachments[]`. Use the Attachments API to fetch attachments for a specific decision.
- There is no `file` field on `QuyetDinh` documents.

## Error Responses

### 404 Not Found
Decision doesn't exist OR user doesn't have access due to unit scoping.

---

## Related

- [GET /api/quyet-dinh](./get-list.md)
- [PATCH /api/quyet-dinh/:id](./patch.md)
- [QuyetDinh Schema](../../../backend/schemas/QuyetDinh.md)
