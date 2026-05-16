# PATCH /api/can-bo-quan-ly/:id

**Endpoint**: `PATCH /api/can-bo-quan-ly/:id`  
**Authentication**: ✅ Required  

**Roles**: admin  
**HTTP Method**: **PATCH** (not PUT)  

**Last Verified**: 2026-05-16

---

## Description

Updates an existing management staff member. **Admin-only operation**. Partial updates supported.

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

Send only the fields you want to update:

```json
{
  "capBac": "Đại tá",
  "soDienThoai": "0987654321",
  "donViQL": "Bộ Quốc phòng"
}
```

**Updatable fields**: `hoTen`, `capBac`, `donViQL`, `chucVu`, `soDienThoai`, `ghiChu`, `soQD`, `ngayRaQD`, `hieuLuc`, `taiKhoan`

---

## Response

### Success (200 OK)

```json
{
  "data": {
    "_id": "507f1f77bcf86cd799439011",
    "hoTen": "Nguyễn Văn A",
    "capBac": "Đại tá",
    "donViQL": "Bộ Quốc phòng",
    "chucVu": "Trưởng khoa",
    "soDienThoai": "0987654321",
    "ghiChu": "",
    "soQD": "QĐ-001/2024",
    "ngayRaQD": "01/01/2024",
    "hieuLuc": "Còn hiệu lực",
    "taiKhoan": "64a1b2c3d4e5f6a7b8c9d0e1",
    "createdAt": "2023-01-15T00:00:00.000Z",
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

### 409 Conflict — Account already linked
The `taiKhoan` ObjectId is already assigned to another staff member.

```json
{ "error": "DUPLICATE_TAIKHOAN" }
```

---

## Related

- [GET /api/can-bo-quan-ly/:id](./get-one.md)
- [DELETE /api/can-bo-quan-ly/:id](./delete.md)
- [CanBoQuanLy Schema](../../../backend/schemas/CanBoQuanLy.md)
