# GET /api/diem/summary

**Endpoint**: `GET /api/diem/summary`
**Authentication**: ✅ Required
**Roles**: admin, staff, viewer, teacher
**Unit Scoping**: ✅ Applied (staff: `allowedUnits`; teacher: `teacherScope`)
**Last Verified**: 2026-05-23

---

## Description

Returns per-student grade summaries for all subjects in the specified education system (`he`). **One row per in-scope student** — students with no `diem` entries are still included; their `grades` array contains one entry per subject with `tbMon: null`. The roster matches `/api/sinh-vien` for the same filters, so the Tổng kết cuối khóa tab shows the full cohort even when nobody has been graded yet.

---

## Request

### Headers
```
Authorization: Bearer <access_token>
```

### Query Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| he | string | **Yes** | Education system. Must be `"Đại học"` or `"Cao đẳng"`. Subjects included in the summary depend on this value: `"Đại học"` uses the first 4 subjects of `MON_ENUM`; `"Cao đẳng"` uses the remaining 3. |
| khoa | string | No | Filter by Khoa ObjectId. |
| daiDoi | string | No | Filter by DaiDoi ObjectId. |
| truong | string | No | Filter by DonViLienKet ObjectId (partner school). |

---

## Response

### Success (200 OK)

```json
{
  "data": [
    {
      "_id": "64a1b2c3d4e5f6a7b8c9d0e1",
      "maSV": "SV001",
      "hoTen": "Nguyễn Văn A",
      "ngaySinh": "2000-01-15T00:00:00.000Z",
      "lop": "K58-CNTT",
      "trangThai": "Đang học",
      "grades": [
        { "mon": "Đường lối quốc phòng và an ninh của ĐCSVN", "tbMon": 8.5 },
        { "mon": "Công tác Đảng, công tác chính trị trong QĐND VN", "tbMon": 7.0 }
      ]
    }
  ]
}
```

Each item in `data` is one student. The `grades` array contains **one entry per subject in the chosen `he`** (4 for Đại học, 3 for Cao đẳng), with the latest `tbMon` for that subject — or `tbMon: null` when the student has no grade recorded for that subject yet.

`data` is ordered by the canonical student sort: `khoaSortKey ASC → daiDoiSortKey ASC → _id ASC`. Within a single (khoa, đại đội) filter that reduces to `_id ASC`, which matches Mongo insertion order and the order used by `/api/sinh-vien` and the Excel exports.

---

## Error Responses

### 400 Bad Request — missing `he`

```json
{
  "error": {
    "code": "INVALID_QUERY",
    "message": "Hệ is required"
  }
}
```

**When**: `he` query parameter is absent or empty.

---

## Notes

- `meta.pages` is **not** returned; this endpoint does not paginate.
- Unit scoping is applied via `scopedStudentIds(user)` — staff/teacher see only students within their scope.

---

## Related

- [GET /api/diem](./get-list.md)
- [GET /api/diem/transcripts](./get-transcripts.md)
- [SinhVien Schema](../../../backend/schemas/SinhVien.md)
