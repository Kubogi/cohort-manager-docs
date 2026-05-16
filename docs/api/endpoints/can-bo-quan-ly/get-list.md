# GET /api/can-bo-quan-ly

**Endpoint**: `GET /api/can-bo-quan-ly`  
**Authentication**: ✅ Required  

**Roles**: admin, staff, viewer  
**Unit Scoping**: ❌ Not applied — `CanBoQuanLy` has no khoa/daiDoi fields; all permitted roles see all records  

**Last Verified**: 2026-05-16

---

## Description

Lists management staff (cán bộ quản lý) with pagination.

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
      "soQD": "QĐ-001/2024",
      "ngayRaQD": "01/01/2024",
      "hieuLuc": "Còn hiệu lực",
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

### Response Fields

| Field | Type | Description |
|-------|------|-------------|
| hoTen | string | Full name |
| capBac | string | Military rank (e.g. "Thiếu tá") |
| donViQL | string | Managing unit / workplace assignment (plain string) |
| chucVu | string | Position / role title |
| soDienThoai | string | Phone number |
| ghiChu | string | Free-text notes |
| soQD | string | Decision reference number |
| ngayRaQD | string | Decision issue date (free-text string, not a Date) |
| hieuLuc | string | Effective status (e.g. "Còn hiệu lực", "Hết hiệu lực") |
| taiKhoan | ObjectId \| null | ObjectId reference to linked `User` account, or null |

---

## Related

- [GET /api/can-bo-quan-ly/:id](./get-one.md)
- [POST /api/can-bo-quan-ly](./post.md)
- [CanBoQuanLy Schema](../../../backend/schemas/CanBoQuanLy.md)
