# GET /api/sinh-vien/:id

**Endpoint**: `GET /api/sinh-vien/:id`  

**Authentication**: ✅ Required  

**Roles**: admin, staff, viewer, teacher

**Unit Scoping**: ✅ Applied (staff: `allowedUnits`; teacher: `teacherScope`)

**Last Verified**: 2026-05-16

---

## Description

Retrieves a single student by ID. **Unit scoping automatically applied**.

---

## Request

### Headers
```
Authorization: Bearer <access_token>
```

### URL Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| id | string | Yes | Student MongoDB ObjectId |

---

## Response

### Success (200 OK)

```json
{
  "data": {
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
    "diem": [
      {
        "_id": "64a1b2c3d4e5f6a7b8c9d0e4",
        "mon": "Chính trị 1",
        "thuongXuyen": 8,
        "mieng": 7,
        "giuaHP": 8,
        "hetHP": 9,
        "tbMon": 8.5,
        "createdAt": "2023-10-01T00:00:00.000Z",
        "updatedAt": "2023-10-01T00:00:00.000Z"
      }
    ],
    "createdAt": "2023-09-01T00:00:00.000Z",
    "updatedAt": "2023-09-01T00:00:00.000Z"
  }
}
```

---

## Error Responses

### 404 Not Found

```json
{
  "error": {
    "code": "NOT_FOUND",
    "message": "Sinh vien not found"
  }
}
```

**When**: Student doesn't exist OR user doesn't have access due to unit scoping (both cases return 404 to avoid leaking the existence of records).

---

## Notes

- Returns embedded `diem` array (grades as subdocuments)
- 404 returned even if student exists but user lacks unit access (security)
- gioiTinh is Boolean (true=Male, false=Female)

---

## Related

- [GET /api/sinh-vien](./get-list.md)
- [PATCH /api/sinh-vien/:id](./patch.md)
- [SinhVien Schema](../../../backend/schemas/SinhVien.md)
