# DELETE /api/quyet-dinh/:id

**Endpoint**: `DELETE /api/quyet-dinh/:id`  

**Authentication**: ✅ Required  

**Roles**: admin  

**Last Verified**: 2026-05-16

---

## Description

Deletes a decision. **Admin-only operation**.

---

## Request

### Headers
```
Authorization: Bearer <access_token>
```

### URL Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| id | string | Yes | QuyetDinh MongoDB ObjectId |

---

## Response

### Success (204 No Content)

**Empty response body**

---

## Error Responses

### 404 Not Found
Decision doesn't exist.

### 403 Forbidden
Only admin role can delete decisions.

---

## Notes

- Returns 204 (No Content) on success
- Does NOT delete referenced SinhVien document

---

## Related

- [GET /api/quyet-dinh/:id](./get-one.md)
- [PATCH /api/quyet-dinh/:id](./patch.md)
- [QuyetDinh Schema](../../../backend/schemas/QuyetDinh.md)
