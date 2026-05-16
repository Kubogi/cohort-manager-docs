# GET /api/can-bo-quan-ly/:id

**Endpoint**: `GET /api/can-bo-quan-ly/:id`  
**Authentication**: ✅ Required  

**Roles**: admin, staff, viewer  
**Unit Scoping**: ❌ Not applied — all permitted roles see all records  

**Last Verified**: 2026-05-16

---

## Description

Retrieves a single management staff member by ID.

---

## Request

### Headers
```
Authorization: Bearer <access_token>
```

### URL Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| id | string | Yes | CanBoQuanLy MongoDB ObjectId |

---

## Response

### Success (200 OK)

```json
{
  "data": {
    "_id": "507f1f77bcf86cd799439011",
    "hoTen": "Nguyễn Văn A",
    "capBac": "Thiếu tá",
    "donViQL": "Bộ Quốc phòng",
    "chucVu": "Trưởng khoa",
    "soDienThoai": "0912345678",
    "ghiChu": "",
    "soQD": "QĐ-001/2024",
    "ngayRaQD": "01/01/2024",
    "hieuLuc": "Còn hiệu lực",
    "taiKhoan": "64a1b2c3d4e5f6a7b8c9d0e1",
    "createdAt": "2023-01-15T00:00:00.000Z",
    "updatedAt": "2023-01-15T00:00:00.000Z"
  }
}
```

---

## Error Responses

### 404 Not Found
Staff member doesn't exist.

---

## Related

- [GET /api/can-bo-quan-ly](./get-list.md)
- [PATCH /api/can-bo-quan-ly/:id](./patch.md)
- [CanBoQuanLy Schema](../../../backend/schemas/CanBoQuanLy.md)
