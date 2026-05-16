# PATCH /api/ho-so-suc-khoe/:id

**Endpoint**: `PATCH /api/ho-so-suc-khoe/:id`  

**Authentication**: ✅ Required  

**Roles**: admin  

**HTTP Method**: **PATCH** (not PUT)  

**Last Verified**: 2026-05-16

---

## Description

Updates an existing health record. **Admin-only operation**. Partial updates supported.

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
| id | string | Yes | HoSoSucKhoe MongoDB ObjectId |

### Body

Send only the fields you want to update:

```json
{
  "thoiGianVe": {
    "gio": "14:00",
    "ngay": "2023-10-15"
  },
  "chuanDoanBenhVien": "Cần theo dõi",
  "trangThai": "Viện"
}
```

All fields are **optional** in update.

---

## Response

### Success (200 OK)

```json
{
  "data": {
    "_id": "507f1f77bcf86cd799439011",
    ...updated health record fields...,
    "updatedAt": "2025-12-31T10:00:00.000Z"
  }
}
```

---

## Error Responses

### 404 Not Found
Health record doesn't exist.

### 403 Forbidden
Only admin role can update health records.

### 400 Bad Request
Invalid enum values or malformed data.

---

## Related

- [GET /api/ho-so-suc-khoe/:id](./get-one.md)
- [DELETE /api/ho-so-suc-khoe/:id](./delete.md)
- [HoSoSucKhoe Schema](../../../backend/schemas/HoSoSucKhoe.md)
