# POST /api/diem

**Endpoint**: `POST /api/diem`  
**Authentication**: ✅ Required  

**Roles**: admin, staff, teacher (teacher writes are restricted to students within `teacherScope`)

**Last Verified**: 2026-01-02

---

## Description

Creates a new grade (adds to SinhVien.diem array). Admin, staff, and teacher can create grades. Teachers can only write grades for students within their `teacherScope`.

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
  "mon": "Chính trị 1",
  "thuongXuyen": 8,
  "mieng": 7,
  "giuaHP": 8,
  "hetHP": 9
}
```

### Required Fields

- `sinhVien` (ObjectId string)
- `mon` (string from MON_ENUM - see schema)
- `thuongXuyen` (number, 0-10) - Required
- `giuaHP` (number, 0-10) - Required
- `hetHP` (number, 0-10) - Required

### Optional Fields

- `mieng` (number, 0-10) - Only used for last 3 subjects

**Note**: `tbMon` is **auto-calculated** from component grades using different formulas:

**Formula 1** (First 4 subjects: Đường lối..., Công tác..., Quân sự chung, Kỹ thuật...):
- `tbMon = thuongXuyen * 0.1 + giuaHP * 0.3 + hetHP * 0.6`
- Note: `mieng` score is NOT used for these subjects

**Formula 2** (Last 3 subjects: Chính trị 1, Chính trị 2, Quân sự):
- `tbMon = thuongXuyen * 0.1 + mieng * 0.1 + giuaHP * 0.2 + hetHP * 0.6`

---

## Response

### Success (201 Created)

```json
{
  "data": {
    "_id": "64a1b2c3d4e5f6a7b8c9d0e4",
    "sinhVien": "64a1b2c3d4e5f6a7b8c9d0e1",
    "mon": "Chính trị 1",
    "thuongXuyen": 8,
    "mieng": 7,
    "giuaHP": 8,
    "hetHP": 9,
    "tbMon": 8.5
  }
}
```

---

## Error Responses

### 403 Forbidden
Only admin, staff, and teacher roles can create grades. Teachers can only create grades for students in their `teacherScope`; writing outside scope returns 403.

### 400 Bad Request
Missing required fields, invalid mon value, or grades out of range.

### 409 Conflict — `SUBJECT_EXEMPT`
Returned when `student.monMienHoc` includes the requested `mon`. Exempt students are not allowed to accumulate grades for that subject; the cross-subject `diemTB` rollup naturally skips them (per-subject `tbMon` would never be written). Clear the exemption in the student modal first, then retry.

```json
{ "error": { "code": "SUBJECT_EXEMPT", "message": "Sinh viên này được miễn học môn này" } }
```

---

## Notes

- Adds to SinhVien.diem subdocument array
- tbMon auto-calculated using different formulas based on subject
- mon must be one of 7 courses from MON_ENUM
- thuongXuyen, giuaHP, hetHP are required fields (not optional)

---

## Related

- [PATCH /api/diem/:id](./patch.md)
- [GET /api/diem](./get-list.md)
- [SinhVien Schema - MON_ENUM](../../../backend/schemas/SinhVien.md)
