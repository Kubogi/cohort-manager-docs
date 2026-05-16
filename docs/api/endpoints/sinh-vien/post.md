# POST /api/sinh-vien

**Endpoint**: `POST /api/sinh-vien`  

**Authentication**: ✅ Required  

**Roles**: admin  

**Last Verified**: 2026-05-16

---

## Description

Creates a new student. **Admin-only operation**.

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
  "maSV": "SV001",
  "hoTen": "Nguyen Van A",
  "ngaySinh": "2000-01-15",
  "nganh": "Công nghệ thông tin",
  "noiSinh": "Hà Nội",
  "gioiTinh": true,
  "soDienThoai": "0912345678",
  "lop": "K58-CNTT",
  "daiDoi": "64a1b2c3d4e5f6a7b8c9d0e1",
  "khoa": "64a1b2c3d4e5f6a7b8c9d0e2",
  "truong": "64a1b2c3d4e5f6a7b8c9d0e3",
  "ngayNhapHoc": "2023-09-01",
  "trangThai": "Đang học",
  "ghiChu": "",
  "trangThaiSucKhoe": "Bình thường"
}
```

### Required Fields

- `maSV` (string, max 100 chars)
- `hoTen` (string, max 200 chars)
- `ngaySinh` (ISO date string)

### Optional Fields

All other fields from SinhVien schema are optional.

**Important**: `gioiTinh` is **Boolean** (true=Male, false=Female), not string.

- `trangThai` (string) — Valid values: `'Đang học'`, `'Hoãn học'`, `'Thôi học'`, `'Đình chỉ'`, `'Miễn học'` (enforced by Joi validator).
- `ghiChuYTe` (string, allow empty, max 2000 chars) — Medical notes.

---

## Response

### Success (201 Created)

```json
{
  "data": {
    "_id": "507f1f77bcf86cd799439011",
    "maSV": "SV001",
    "hoTen": "Nguyen Van A",
    ...all fields...,
    "diem": [],
    "createdAt": "2023-09-01T00:00:00.000Z",
    "updatedAt": "2023-09-01T00:00:00.000Z"
  }
}
```

---

## Error Responses

### 403 Forbidden
Only admin role can create students.

### 400 Bad Request - Validation
Missing required fields or invalid data types.

### 409 Conflict - Duplicate
If compound unique index (maSV, hoTen, ngaySinh) already exists.

---

## Notes

- diem array starts empty (add grades via /api/diem endpoints)
- gioiTinh: true=Male, false=Female (Boolean, not string!)
- Compound unique index on (maSV, hoTen, ngaySinh) is sparse

---

## Related

- [PATCH /api/sinh-vien/:id](./patch.md)
- [GET /api/sinh-vien](./get-list.md)
- [SinhVien Schema](../../../backend/schemas/SinhVien.md)
