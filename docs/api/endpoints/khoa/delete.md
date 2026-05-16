# DELETE /api/khoa-hoc/:id

**Endpoint**: `DELETE /api/khoa-hoc/:id`  

**Authentication**: ✅ Required  

**Roles**: admin  

**Last Verified**: 2026-05-16

---

## Description

Deletes a course. **Admin-only operation**.

---

## Request

### Headers
```
Authorization: Bearer <access_token>
```

### URL Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| id | string | Yes | Khoa MongoDB ObjectId |

### Query Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| cascade | boolean | false | Pass `true` or `1` to delete all related students and platoons |

---

## Response

### Success — No cascade (204 No Content)

**Empty response body**

### Success — With cascade (200 OK)

```json
{
  "data": {
    "deletedSinhVien": 42,
    "deletedDaiDoi": 3
  }
}
```

> ⚠️ **Destructive**: When `?cascade=true` is passed, ALL SinhVien and DaiDoi documents belonging to this Khoa are permanently deleted. This cannot be undone.

---

## Error Responses

### 404 Not Found
Course doesn't exist.

### 403 Forbidden
Only admin role can delete courses.

---

## Notes

- Without `?cascade`, returns 204 and deletes only the Khoa document. Related SinhVien and DaiDoi records are left intact (but may reference a deleted khoa).
- With `?cascade=true` or `?cascade=1`, returns 200 with counts of deleted related documents.

---

## Related

- [GET /api/khoa-hoc/:id](./get-one.md)
- [PATCH /api/khoa-hoc/:id](./patch.md)
- [Khoa Schema](../../../backend/schemas/Khoa.md)
