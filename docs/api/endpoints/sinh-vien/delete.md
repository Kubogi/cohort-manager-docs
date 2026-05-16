# DELETE /api/sinh-vien/:id

**Endpoint**: `DELETE /api/sinh-vien/:id`  

**Authentication**: ✅ Required  

**Roles**: admin  

**Last Verified**: 2026-05-16

---

## Description

Deletes a student. **Admin-only operation**.

---

## Request

### Headers
```
Authorization: Bearer <access_token>
```

### URL Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| id | string | Yes | Student MongoDB ObjectId |

---

## Response

### Success (204 No Content)

**Empty response body**

---

## Error Responses

### 404 Not Found

```json
{
  "error": {
    "code": "NOT_FOUND",
    "message": "Sinh vien not found or not permitted"
  }
}
```

### 403 Forbidden
Only admin role can delete students.

---

## Notes

- Returns 204 (No Content) on success.
- **No cascade.** Only the `SinhVien` document is deleted. Related `QuyetDinh` and `HoSoSucKhoe` documents that reference this student are **not** deleted — they become orphaned. Embedded `diem[]` grades are deleted with the student document.
- The 404 response covers both "not found" and "scoped-out" cases (admin-only route, so in practice only "not found" applies).

---

## Related

- [GET /api/sinh-vien/:id](./get-one.md)
- [PATCH /api/sinh-vien/:id](./patch.md)
- [SinhVien Schema](../../../backend/schemas/SinhVien.md)
