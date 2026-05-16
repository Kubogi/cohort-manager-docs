# POST /api/ho-so-suc-khoe

**Endpoint**: `POST /api/ho-so-suc-khoe`  

**Authentication**: ✅ Required  

**Roles**: admin  

**Last Verified**: 2026-05-16

---

## Description

Creates a new health record (hospital visit). **Admin-only operation**.

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
  "thayQuanLy": "Nguyễn Văn B",
  "khung": "Chính trị",
  "thoiGianDi": {
    "gio": "08:00",
    "ngay": "2023-10-15"
  },
  "thoiGianVe": {
    "gio": "12:00",
    "ngay": "2023-10-15"
  },
  "benhVien": "BV 108",
  "lyDo": "Khám định kỳ",
  "chuanDoanBenhVien": "Sức khỏe tốt",
  "nguoiDiKem": "Sinh viên",
  "thuocDTCC": "A1-Class",
  "sinhVienDiCung": "SV002, SV003",
  "trangThai": "Bình thường",
  "khoa": "64a1b2c3d4e5f6a7b8c9d0e2",
  "daiDoi": "64a1b2c3d4e5f6a7b8c9d0e3",
  "truong": "64a1b2c3d4e5f6a7b8c9d0e4"
}
```

### Required Fields

- `sinhVien` (ObjectId string)

### Optional Fields

All other fields optional. See [HoSoSucKhoe Schema](../../../backend/schemas/HoSoSucKhoe.md) for field details.

**Important enum values:**
- `khung`: "Chính trị" or "Quân sự"
- `nguoiDiKem`: "Sinh viên" or "Người thân"
- `trangThai`: "Bình thường" or "Viện"

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
Only admin role can create health records.

### 400 Bad Request
Missing required fields, invalid enums, or malformed data.

---

## Related

- [PATCH /api/ho-so-suc-khoe/:id](./patch.md)
- [GET /api/ho-so-suc-khoe](./get-list.md)
- [HoSoSucKhoe Schema](../../../backend/schemas/HoSoSucKhoe.md)
