# PATCH /api/sinh-vien/:id

**Endpoint**: `PATCH /api/sinh-vien/:id`  

**Authentication**: ✅ Required  

**Roles**: admin  

**HTTP Method**: **PATCH** (not PUT)  

**Last Verified**: 2026-05-16

---

## Description

Updates an existing student. **Admin-only operation**. Partial updates supported.

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
| id | string | Yes | Student MongoDB ObjectId |

### Body

Send only the fields you want to update:

```json
{
  "soDienThoai": "0987654321",
  "trangThai": "Tốt nghiệp",
  "ngayVe": "2025-06-30"
}
```

All fields from SinhVien schema are **optional** in update.

---

## Response

### Success (200 OK)

```json
{
  "data": {
    "_id": "507f1f77bcf86cd799439011",
    ...updated student fields...,
    "updatedAt": "2025-12-31T10:00:00.000Z"
  }
}
```

---

## Error Responses

### 404 Not Found
Student doesn't exist.

### 403 Forbidden
Only admin role can update students.

### 400 Bad Request
Invalid field values or data types.

---

## Notes

- **PATCH semantics**: Only provided fields are updated
- Cannot update `_id`, `createdAt`
- `updatedAt` automatically updated
- To update grades, use /api/diem endpoints (grades are embedded)
- gioiTinh is Boolean if provided

---

## Related

- [GET /api/sinh-vien/:id](./get-one.md)
- [DELETE /api/sinh-vien/:id](./delete.md)
- [POST /api/diem](../../diem/post.md) - To update grades
- [SinhVien Schema](../../../backend/schemas/SinhVien.md)
