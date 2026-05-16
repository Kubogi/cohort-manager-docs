# POST /api/quyet-dinh

**Endpoint**: `POST /api/quyet-dinh`  

**Authentication**: ✅ Required  

**Roles**: admin  

**Last Verified**: 2026-05-16

---

## Description

Creates a new decision. **Admin-only operation**.

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
  "sinhVien": "64a1b2c3d4e5f6a7b8c9d0e1",
  "soQD": "QD-123/2023",
  "ngayKiQD": "2023-10-15",
  "loaiQD": "Khen thưởng",
  "lyDo": "Hoàn thành xuất sắc nhiệm vụ",
  "khoa": "64a1b2c3d4e5f6a7b8c9d0e2",
  "daiDoi": "64a1b2c3d4e5f6a7b8c9d0e3",
  "trangThai": "Chưa xử lý"
}
```

### Required Fields

- `sinhVien` (ObjectId string)
- `soQD` (string, max 100 chars)

### Optional Fields

- `ngayKiQD` (ISO date string)
- `loaiQD` (string, max 100 chars)
- `lyDo` (string, max 500 chars)
- `khoa` (ObjectId string) — cohort scope for unit scoping
- `daiDoi` (ObjectId string) — battalion scope for unit scoping
- `trangThai` (string, max 100 chars) — decision status; accepts any string up to 100 characters. There is no enum restriction and no default value enforced by the schema or validator.

> There is no `file` field on QuyetDinh. Attachments are managed separately through the Attachments API.

---

## Response

### Success (201 Created)

```json
{
  "data": {
    "_id": "507f1f77bcf86cd799439011",
    ...all fields...,
    "createdAt": "2023-10-15T00:00:00.000Z",
    "updatedAt": "2023-10-15T00:00:00.000Z"
  }
}
```

---

## Error Responses

### 403 Forbidden
Only admin role can create decisions.

### 400 Bad Request
Missing required fields or invalid data.

---

## Related

- [PATCH /api/quyet-dinh/:id](./patch.md)
- [GET /api/quyet-dinh](./get-list.md)
- [QuyetDinh Schema](../../../backend/schemas/QuyetDinh.md)
