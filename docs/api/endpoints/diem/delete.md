# DELETE /api/diem/:id

**Endpoint**: `DELETE /api/diem/:id`  
**Authentication**: ✅ Required  
**Roles**: admin, staff
**Last Verified**: 2026-05-23

---

## Description

Deletes a grade (removes from SinhVien.diem array). **Admin and staff only** — teachers are read-only on grades.

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
