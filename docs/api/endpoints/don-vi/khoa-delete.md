# DELETE /api/don-vi/khoa/:id

**Endpoint**: `DELETE /api/don-vi/khoa/:id`  
**Authentication**: ✅ Required  

**Roles**: admin  

**Last Verified**: 2025-12-31

---

## Description

Deletes a khoa. **Admin-only operation**.

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

---

## Response

### Success (204 No Content)

**Empty response body**

---

## Notes

- No cascade — deletes only the Khoa document. Related SinhVien and DaiDoi records are not deleted by this route.

---

## Related

- [GET /api/don-vi/khoa](./khoa-get-list.md)
- [PATCH /api/don-vi/khoa/:id](./khoa-patch.md)
- [DonViLienKet Schema](../../../backend/schemas/DonViLienKet.md)
