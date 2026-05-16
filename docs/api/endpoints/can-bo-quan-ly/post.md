# POST /api/can-bo-quan-ly

**Endpoint**: `POST /api/can-bo-quan-ly`  
**Authentication**: ✅ Required  

**Roles**: admin  

**Last Verified**: 2026-05-16

---

## Description

Creates a new management staff member. **Admin-only operation**.

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
  "hoTen": "Nguyễn Văn A",
  "capBac": "Thiếu tá",
  "donViQL": "Bộ Quốc phòng",
  "chucVu": "Trưởng khoa",
  "soDienThoai": "0912345678",
  "ghiChu": "",
  "soQD": "QĐ-001/2024",
  "ngayRaQD": "01/01/2024",
  "hieuLuc": "Còn hiệu lực",
  "taiKhoan": "64a1b2c3d4e5f6a7b8c9d0e1"
}
```

### Required Fields

- `hoTen` (string, max 200 chars) - Full name

### Optional Fields

- `capBac` (string) - Military rank (e.g. "Thiếu tá", "Đại tá")
- `donViQL` (string) - Managing unit / workplace assignment (plain string, not an ObjectId)
- `chucVu` (string) - Position / role title
- `soDienThoai` (string) - Phone number
- `ghiChu` (string) - Free-text notes
- `soQD` (string) - Decision reference number
- `ngayRaQD` (string) - Decision issue date (free-text string)
- `hieuLuc` (string) - Effective-status text (e.g. "Còn hiệu lực", "Hết hiệu lực")
- `taiKhoan` (ObjectId string) - ObjectId of an existing `User` to link this staff member to

---

## Response

### Success (201 Created)

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
    "createdAt": "2025-12-31T10:00:00.000Z",
    "updatedAt": "2025-12-31T10:00:00.000Z"
  }
}
```

---

## Error Responses

### 403 Forbidden
Only admin role can create staff.

### 400 Bad Request
Missing required field `hoTen`.

### 409 Conflict — Duplicate name
A staff member with the same `hoTen` already exists.

```json
{ "error": { "code": "DUPLICATE_CAN_BO", "message": "Cán bộ đã tồn tại" } }
```

### 409 Conflict — Account already linked
The `taiKhoan` ObjectId is already assigned to another staff member.

```json
{ "error": "DUPLICATE_TAIKHOAN" }
```

---

## Related

- [PATCH /api/can-bo-quan-ly/:id](./patch.md)
- [GET /api/can-bo-quan-ly](./get-list.md)
- [CanBoQuanLy Schema](../../../backend/schemas/CanBoQuanLy.md)
