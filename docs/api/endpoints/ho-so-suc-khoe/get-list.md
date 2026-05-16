# GET /api/ho-so-suc-khoe

**Endpoint**: `GET /api/ho-so-suc-khoe`  

**Authentication**: ✅ Required  

**Roles**: admin, staff, viewer, teacher

**Unit Scoping**: ✅ Applied (staff: `allowedUnits`; teacher: `teacherScope`)  

**Last Verified**: 2026-05-16

---

## Description

Lists health records (hospital visit tracking) with pagination and filtering. **Unit scoping automatically applied** based on user's allowedUnits.

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
| khoa | string | - | Filter by Khoa ID |
| daiDoi | string | - | Filter by DaiDoi ID |
| trangThai | string | - | Filter by status: "Bình thường" or "Viện" |

---

## Response

### Success (200 OK)

```json
{
  "data": [
    {
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
      "updatedAt": "2023-10-15T00:00:00.000Z",
      "attachments": []
    }
  ],
  "meta": {
    "page": 1,
    "limit": 20,
    "total": 78
  }
}
```

---

## Notes

- `attachments` is added to each item by `decorateWithAttachments()`. It is **only present on list responses** — the single-record `GET /:id` endpoint does not include it.

## Unit Scoping

Staff are filtered by `allowedUnits`; teachers by `teacherScope`. Viewer and admin see all records their route allows.

---

## Notes

- **NOT health measurements** - tracks hospital visits/appointments
- thoiGianDi/thoiGianVe are nested objects with gio (string) and ngay (Date)
- khung: "Chính trị" or "Quân sự"
- trangThai: "Bình thường" or "Viện"
- nguoiDiKem: "Sinh viên" or "Người thân"

---

## Related

- [GET /api/ho-so-suc-khoe/:id](./get-one.md)
- [GET /api/ho-so-suc-khoe/stats](./get-stats.md)
- [POST /api/ho-so-suc-khoe](./post.md)
- [HoSoSucKhoe Schema](../../../backend/schemas/HoSoSucKhoe.md)
