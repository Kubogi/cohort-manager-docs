# PATCH /api/diem/:id

**Endpoint**: `PATCH /api/diem/:id`  
**Authentication**: ✅ Required  
**Roles**: admin, staff, teacher (teacher writes restricted to students within `teacherScope`)
**HTTP Method**: **PATCH** (not PUT)
**Last Verified**: 2025-12-31

---

## Description

Updates an existing grade. Partial updates supported. Teachers can only update grades for students within their `teacherScope`.

---

## Request

### Headers
```
Content-Type: application/json
Authorization: Bearer <access_token>
```

### URL Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| id | string | Yes | Grade subdocument _id |

### Body

Send only the fields you want to update:

```json
{
  "thuongXuyen": 9,
  "hetHP": 10
}
```

All fields are **optional** in update. A component field set to `null` flips it to "miễn thành phần" (exempt) — `tbMon` is recomputed using the remaining components' rescaled weights. See [POST /api/diem](./post.md#per-component-exemption-null) for the formula.

---

## Response

### Success (200 OK)

```json
{
  "data": {
    "_id": "64a1b2c3d4e5f6a7b8c9d0e4",
    ...updated grade fields...,
    "tbMon": 9.0,
    "updatedAt": "2025-12-31T10:00:00.000Z"
  }
}
```

**Note**: `tbMon` is **recalculated** automatically when component grades change.

---

## Error Responses

### 404 Not Found
Grade doesn't exist.

### 403 Forbidden
Only admin/staff roles can update grades.

### 409 Conflict — `SUBJECT_EXEMPT`
Returned when the targeted grade's `mon` (either the existing value or a new `mon` in the payload) is in `student.monMienHoc`. Clear the exemption first, then retry. Same shape as [POST /api/diem](./post.md#409-conflict--subject_exempt).

---

## Related

- [GET /api/diem/:id](./get-one.md)
- [DELETE /api/diem/:id](./delete.md)
- [SinhVien Schema](../../../backend/schemas/SinhVien.md)
