# PATCH /api/quyet-dinh/:id

**Endpoint**: `PATCH /api/quyet-dinh/:id`  

**Authentication**: ✅ Required  

**Roles**: admin  

**HTTP Method**: **PATCH** (not PUT)  

**Last Verified**: 2026-05-16

---

## Description

Updates an existing decision. **Admin-only operation**. Partial updates supported.

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
| id | string | Yes | QuyetDinh MongoDB ObjectId |

### Body

Send only the fields you want to update:

```json
{
  "loaiQD": "Kỷ luật",
  "lyDo": "Updated reason",
  "trangThai": "Đã xử lý"
}
```

All fields are **optional** in update. Available fields: `soQD`, `ngayKiQD`, `loaiQD`, `lyDo`, `khoa`, `daiDoi`, `trangThai`.

> There is no `file` field on QuyetDinh. Attachments are managed separately.

---

## Response

### Success (200 OK)

```json
{
  "data": {
    "_id": "507f1f77bcf86cd799439011",
    ...updated decision fields...,
    "updatedAt": "2025-12-31T10:00:00.000Z"
  }
}
```

---

## Error Responses

### 404 Not Found
Decision doesn't exist.

### 403 Forbidden
Only admin role can update decisions.

---

## Related

- [GET /api/quyet-dinh/:id](./get-one.md)
- [DELETE /api/quyet-dinh/:id](./delete.md)
- [QuyetDinh Schema](../../../backend/schemas/QuyetDinh.md)
