# GET /api/diem

**Endpoint**: `GET /api/diem`  
**Authentication**: ✅ Required  
**Roles**: admin, staff, viewer, teacher
**Unit Scoping**: ✅ Applied (staff: `allowedUnits`; teacher: `teacherScope`, via parent SinhVien)  
**Last Verified**: 2025-12-31

---

## Description

Lists grades (điểm) with pagination. Manages embedded subdocuments from SinhVien.diem array. **Unit scoping applied via parent student**.

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
| sinhVien | string | - | Filter by SinhVien ID. **Accepts a comma-separated list** (e.g. `?sinhVien=id1,id2`) → matched with `$in`. Single value behaves as exact match. |
| mon | string | - | Filter by course name. When provided, each item also includes `maSV`, `hoTen`, `ngaySinh`, `lop` from the parent student (different response shape from the no-`mon` case). |
| khoa | string | - | Filter by Khoa ID |
| daiDoi | string | - | Filter by DaiDoi ID |
| truong | string | - | Filter by DonViLienKet ID (partner school) |
| trangThai | string | - | Filter by student status |
| maSV | string | - | Filter on student code. **Accepts a comma-separated list** (e.g. `?maSV=A,B,C`) → matched with `$in` (exact equality per item). A single value (no comma) is still treated as a case-insensitive regex substring — that's how the advanced "Tìm kiếm theo mã SV" partial-match filter on the Nhập điểm tab works. |
| hoTen | string | - | Regex filter on student name |
| lop | string | - | Regex filter on class designation |

**Default sort**:
- When `mon` is provided: rows are sorted by the canonical student ordering on the parent students — `{ khoaSortKey: 1, daiDoiSortKey: 1, _id: 1 }`. Within each `(khoa, daiDoi)` group rows appear in insertion order. See [SinhVien get-list](../sinh-vien/get-list.md) for the meaning of each key.
- When `mon` is not provided: the response shape is grade-centric (one row per embedded grade) and the sort stays `{ _id: -1 }` on the embedded grade documents (newest grade first).

Both are stable for pagination.

---

## Response

### Success (200 OK)

```json
{
  "data": [
    {
      "_id": "64a1b2c3d4e5f6a7b8c9d0e4",
      "sinhVien": "64a1b2c3d4e5f6a7b8c9d0e1",
      "mon": "Chính trị 1",
      "thuongXuyen": 8,
      "mieng": 7,
      "giuaHP": 8,
      "hetHP": 9,
      "tbMon": 8.5
    }
  ],
  "meta": {
    "page": 1,
    "limit": 20,
    "total": 120
  }
}
```

---

## Notes

- Grades are embedded subdocuments in `SinhVien.diem[]`; `createdAt`/`updatedAt` exist on the subdocument but are **not projected** by this endpoint.
- `tbMon` is auto-calculated from component grades.
- `mon` must be one of the 7 values in `MON_ENUM` — see [SinhVien Schema](../../../backend/schemas/SinhVien.md).
- Unit scoping is applied to the parent student query.

---

## Related

- [GET /api/diem/:id](./get-one.md)
- [GET /api/diem/transcripts](./get-transcripts.md)
- [POST /api/diem](./post.md)
- [SinhVien Schema](../../../backend/schemas/SinhVien.md)
