# DELETE /api/can-bo-quan-ly/:id

**Endpoint**: `DELETE /api/can-bo-quan-ly/:id`  
**Authentication**: ✅ Required  

**Roles**: admin  

**Last Verified**: 2025-12-31

---

## Description

Deletes a management staff member. **Admin-only operation**.

---

## Request

### Headers
```
Authorization: Bearer <access_token>
```

### URL Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| id | string | Yes | CanBoQuanLy MongoDB ObjectId |

---

## Response

### Success (204 No Content)

**Empty response body**

---

## Error Responses

### 404 Not Found
Staff member doesn't exist.

### 403 Forbidden
Only admin role can delete staff.

---

## Notes

- Returns 204 (No Content) on success
- Does NOT delete referenced unit (khoa/daiDoi/donViLienKet)

---

## Related

- [GET /api/can-bo-quan-ly/:id](./get-one.md)
- [PATCH /api/can-bo-quan-ly/:id](./patch.md)
- [CanBoQuanLy Schema](../../../backend/schemas/CanBoQuanLy.md)
