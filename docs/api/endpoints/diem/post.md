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
- `thuongXuyen` (number 0-10, or `null` for miễn thành phần) - Required to be present in the body
- `giuaHP` (number 0-10, or `null` for miễn thành phần) - Required to be present in the body
- `hetHP` (number 0-10, or `null` for miễn thành phần) - Required to be present in the body

### Optional Fields

- `mieng` (number 0-10, or `null` for miễn thành phần) - Only used for last 3 subjects

### Per-component exemption (`null`)

A component value of `null` means the student is exempt from that specific exam ("miễn thành phần"). The `tbMon` formula then **skips that component from both numerator and denominator** and rescales the remaining weights to sum to the full curriculum weight. Use this when a student is excused from one cell (e.g. Giữa HP) but still participates in the others; for whole-subject exemption use `monMienHoc[]` on the student instead.

**Note**: `tbMon` is **auto-calculated** from component grades using different weight sets per subject:

**Đại học** (first 4 subjects): weights `10% + 30% + 60%` on `thuongXuyen + giuaHP + hetHP`. `mieng` is ignored.

**Cao đẳng** (last 3 subjects): weights `10% + 10% + 20% + 60%` on `thuongXuyen + mieng + giuaHP + hetHP`.

Rescaling examples (Đại học):
- Full row tx=8, giuaHP=9, hetHP=7 → `(8·1 + 9·3 + 7·6) / 10 = 7.7`
- `giuaHP: null`, tx=10, hetHP=8 → `(10·1 + 8·6) / 7 ≈ 8.3`
- All three null → `tbMon: null` (no division by zero).

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
- `thuongXuyen`, `giuaHP`, `hetHP` must be present in the body (Joi `required()`); their value may be a number 0-10 or `null` (miễn thành phần)

---

## Related

- [PATCH /api/diem/:id](./patch.md)
- [GET /api/diem](./get-list.md)
- [SinhVien Schema - MON_ENUM](../../../backend/schemas/SinhVien.md)
