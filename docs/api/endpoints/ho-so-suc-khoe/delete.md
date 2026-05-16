# DELETE /api/ho-so-suc-khoe/:id

**Endpoint**: `DELETE /api/ho-so-suc-khoe/:id`  

**Authentication**: ✅ Required  

**Roles**: admin  

**Last Verified**: 2026-05-16

---

## Description

Deletes a health record. **Admin-only operation**.

---

## Request

### Headers
```
Authorization: Bearer <access_token>
```

### URL Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| id | string | Yes | HoSoSucKhoe MongoDB ObjectId |

---

## Response

### Success (204 No Content)

**Empty response body**

---

## Error Responses

### 404 Not Found
Health record doesn't exist.

### 403 Forbidden
Only admin role can delete health records.

---

## Notes

- Returns 204 (No Content) on success
- Does NOT delete referenced SinhVien document

---

## Related

- [GET /api/ho-so-suc-khoe/:id](./get-one.md)
- [PATCH /api/ho-so-suc-khoe/:id](./patch.md)
- [HoSoSucKhoe Schema](../../../backend/schemas/HoSoSucKhoe.md)
