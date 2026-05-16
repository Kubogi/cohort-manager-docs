# DELETE /api/diem/:id

**Endpoint**: `DELETE /api/diem/:id`  
**Authentication**: ✅ Required  

**Roles**: admin, staff, teacher (teacher writes restricted to students within `teacherScope`)

**Last Verified**: 2025-12-31

---

## Description

Deletes a grade (removes from SinhVien.diem array). Teachers can only delete grades for students within their `teacherScope`.

---

## Request

### Headers
```
Authorization: Bearer <access_token>
```

### URL Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| id | string | Yes | Grade subdocument _id |

---

## Response

### Success (204 No Content)

**Empty response body**

---

## Error Responses

### 404 Not Found
Grade doesn't exist.

### 403 Forbidden
Only admin/staff roles can delete grades.

---

## Notes

- Returns 204 (No Content) on success
- Removes subdocument from SinhVien.diem array
- Does NOT delete parent SinhVien document

---

## Related

- [GET /api/diem/:id](./get-one.md)
- [PATCH /api/diem/:id](./patch.md)
- [SinhVien Schema](../../../backend/schemas/SinhVien.md)
