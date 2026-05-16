# GET /api/sinh-vien

**Endpoint**: `GET /api/sinh-vien`  

**Authentication**: ✅ Required  

**Roles**: admin, staff, viewer, teacher

**Unit Scoping**: ✅ Applied (staff: `allowedUnits`; teacher: `teacherScope`)

**Last Verified**: 2026-05-16

---

## Description

Lists students with pagination and filtering. Unit scoping is automatically applied: staff are filtered by `allowedUnits`, teachers are filtered by `teacherScope`.

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
| q | string | - | Text search in maSV, hoTen fields |
| khoa | string | - | Filter by Khoa ID |
| daiDoi | string | - | Filter by DaiDoi ID |
| donViLienKet | string | - | Filter by DonViLienKet ID (school). `truong` is accepted as an alias; `donViLienKet` takes priority. |
| truong | string | - | Alias for `donViLienKet`. |
| status | string | - | Filter by status (maps to `trangThai` field) |
| maSV | string | - | Regex filter on student ID |
| cccd | string | - | Regex filter on citizen-ID number |
| hoTen | string | - | Regex filter on full name |
| ngaySinh | string | - | Date filter. Accepts `dd/mm/yyyy` or ISO `yyyy-mm-dd` format. (`dd-mm-yyyy` is not supported — it is parsed as US `mm-dd-yyyy` by `new Date()`.) |
| nganh | string | - | Regex filter on major |
| noiSinh | string | - | Regex filter on birthplace |
| soDienThoai | string | - | Regex filter on phone number |
| lop | string | - | Regex filter on class designation |

---

## Response

### Success (200 OK)

```json
{
  "data": [
    {
      "_id": "507f1f77bcf86cd799439011",
      "maSV": "SV001",
      "hoTen": "Nguyen Van A",
      "ngaySinh": "2000-01-15T00:00:00.000Z",
      "nganh": "Công nghệ thông tin",
      "noiSinh": "Hà Nội",
      "gioiTinh": true,
      "soDienThoai": "0912345678",
      "lop": "K58-CNTT",
      "daiDoi": "64a1b2c3d4e5f6a7b8c9d0e1",
      "khoa": "64a1b2c3d4e5f6a7b8c9d0e2",
      "truong": "64a1b2c3d4e5f6a7b8c9d0e3",
      "ngayNhapHoc": "2023-09-01T00:00:00.000Z",
      "ngayVe": null,
      "trangThai": "Đang học",
      "ghiChu": "",
      "trangThaiSucKhoe": "Bình thường",
      "diem": [],
      "createdAt": "2023-09-01T00:00:00.000Z",
      "updatedAt": "2023-09-01T00:00:00.000Z"
    }
  ],
  "meta": {
    "page": 1,
    "limit": 20,
    "total": 150
  }
}
```

---

## Unit Scoping

- **Admin**: Sees all students.
- **Staff only**: Filtered by `allowedUnits` (`applyUnitScope`). Example: `allowedUnits.khoa = ['K47']` restricts results to K47 students.
- **Teacher**: Filtered by `teacherScope` (`applyTeacherScope`). Only students in the teacher's `(khoa, daiDoi)` scope entries are returned.
- **Viewer**: No unit restriction — `applyUnitScope` is a no-op for non-staff users. Viewers see all students their route gate allows.

---

## Notes

- gioiTinh is **Boolean** (true=Male, false=Female), not string
- diem array may be empty if student has no grades yet
- Unit scoping is automatic and cannot be bypassed
- Search is case-insensitive and matches partial strings

---

## Related

- [GET /api/sinh-vien/:id](./get-one.md)
- [POST /api/sinh-vien](./post.md)
- [SinhVien Schema](../../../backend/schemas/SinhVien.md)
