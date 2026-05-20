# GET /api/diem/xep-loai-by-unit

**Endpoint**: `GET /api/diem/xep-loai-by-unit`
**Authentication**: ✅ Required

**Roles**: admin, staff, viewer, teacher

**Last Verified**: 2026-05-19

**Unit Scoping**: ✅ Applied (staff: `allowedUnits`; teacher: `teacherScope`)

---

## Description

Returns a grade-classification breakdown per battalion (đại đội). Each row represents one battalion and counts how many of its students fall into each classification bucket based on their average grade (`diemTB`), along with `tong` — the total number of students that contributed to the row.

The classification buckets are: **Xuất sắc** (9–10), **Giỏi** (8–9), **Khá** (7–8), **Trung bình** (5–7), **Nợ môn** (< 5 or missing required subjects). Students with no grades yet, or whose `trangThai` is a suppress-rollup status (`Học ghép` / `Học lẻ`), do not enter any bucket but still count toward `tong`.

Classification is computed using the student's average across all subjects belonging to their school's education system (`heDaoTao`). Students whose school has an ambiguous or unrecognised `heDaoTao` are counted in `meta.unknownHe` and excluded from both buckets and `tong`.

---

## Request

### Headers
```
Authorization: Bearer <access_token>
```

### Query Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| khoa | string | No | Filter battalions by Khoa ObjectId. |
| daiDoi | string | No | Return only this DaiDoi ObjectId's row. |
| truong | string | No | Filter students by DonViLienKet ObjectId (partner school). |

---

## Response

### Success (200 OK)

```json
{
  "data": [
    {
      "khoaId": "64a1b2c3d4e5f6a7b8c9d0e1",
      "khoaTen": "Khóa 47",
      "daiDoiId": "64a1b2c3d4e5f6a7b8c9d0e2",
      "daiDoiTen": "D1",
      "truongId": "64a1b2c3d4e5f6a7b8c9d0e3",
      "truongTen": "Đại học ABC",
      "xuatSac": 5,
      "gioi": 12,
      "kha": 20,
      "trungBinh": 8,
      "noMon": 2,
      "tong": 50
    }
  ],
  "meta": {
    "unknownHe": 1
  }
}
```

### Response Fields

| Field | Type | Description |
|-------|------|-------------|
| data[].khoaId | string | Khoa ObjectId |
| data[].khoaTen | string | Khoa name |
| data[].daiDoiId | string | Battalion ObjectId |
| data[].daiDoiTen | string | Battalion name |
| data[].truongId | string | Partner school ObjectId |
| data[].truongTen | string | Partner school name |
| data[].xuatSac | number | Students with diemTB ≥ 9.0 |
| data[].gioi | number | Students with 8.0 ≤ diemTB < 9.0 |
| data[].kha | number | Students with 7.0 ≤ diemTB < 8.0 |
| data[].trungBinh | number | Students with 5.0 ≤ diemTB < 7.0 |
| data[].noMon | number | Students with diemTB < 5.0 |
| data[].tong | number | Total students in this row (sum of bucket counts + students with no grades + students whose `trangThai` suppresses the rollup, e.g. `Học ghép` / `Học lẻ`). Excludes students counted in `meta.unknownHe`. |
| meta.unknownHe | number | Students excluded because their school's `heDaoTao` was ambiguous or unrecognised |

Results are sorted by `khoaTen → truongTen → daiDoiTen` (Vietnamese locale).

---

## Related

- [GET /api/diem/summary](./get-summary.md)
- [GET /api/diem/export-so-tong](./get-export-so-tong.md)
- [SinhVien Schema](../../../backend/schemas/SinhVien.md)
