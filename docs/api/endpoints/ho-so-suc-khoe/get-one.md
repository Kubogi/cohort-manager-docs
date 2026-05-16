# GET /api/ho-so-suc-khoe/:id

**Endpoint**: `GET /api/ho-so-suc-khoe/:id`  

**Authentication**: ✅ Required  

**Roles**: admin, staff, viewer, teacher

**Unit Scoping**: ✅ Applied (staff: `allowedUnits`; teacher: `teacherScope`)  

**Last Verified**: 2026-05-16

---

## Description

Retrieves a single health record by ID. **Unit scoping automatically applied**.

---

## Request

### Headers
```
Authorization: Bearer <access_token>
```

### URL Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| id | string | Yes | HoSoSucKhoe MongoDB ObjectId |

---

## Response

### Success (200 OK)

```json
{
  "data": {
    "_id": "507f1f77bcf86cd799439011",
    "sinhVien": "64a1b2c3d4e5f6a7b8c9d0e1",
    "thayQuanLy": "Nguyễn Văn B",
    "khung": "Chính trị",
    "thoiGianDi": {
      "gio": "08:00",
      "ngay": "2023-10-15T00:00:00.000Z"
    },
    "thoiGianVe": {
      "gio": "12:00",
      "ngay": "2023-10-15T00:00:00.000Z"
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
    "truong": "64a1b2c3d4e5f6a7b8c9d0e4",
    "createdAt": "2023-10-15T00:00:00.000Z",
    "updatedAt": "2023-10-15T00:00:00.000Z"
  }
}
```

---

## Notes

- `attachments` is **not** included in single-record responses. Only the list endpoint decorates items with `attachments[]`.

## Error Responses

### 404 Not Found
Health record doesn't exist OR user doesn't have access due to unit scoping.

---

## Related

- [GET /api/ho-so-suc-khoe](./get-list.md)
- [PATCH /api/ho-so-suc-khoe/:id](./patch.md)
- [HoSoSucKhoe Schema](../../../backend/schemas/HoSoSucKhoe.md)
