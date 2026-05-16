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

All fields are **optional** in update.

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

---

## Related

- [GET /api/diem/:id](./get-one.md)
- [DELETE /api/diem/:id](./delete.md)
- [SinhVien Schema](../../../backend/schemas/SinhVien.md)
