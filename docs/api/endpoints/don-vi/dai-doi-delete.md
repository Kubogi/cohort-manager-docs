# DELETE /api/don-vi/dai-doi/:id

**Endpoint**: `DELETE /api/don-vi/dai-doi/:id`  
**Authentication**: ✅ Required  

**Roles**: admin  

**Last Verified**: 2026-05-16

---

## Description

Deletes a dai-doi. **Admin-only operation**.

---

## Request

### Headers
```
Authorization: Bearer <access_token>
```

### URL Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| id | string | Yes | DaiDoi MongoDB ObjectId |

---

## Response

### Success (204 No Content)

**Empty response body**

---

## Notes

- **Cascade behavior not verified** - related students/staff may become orphaned

---

## Related

- [GET /api/don-vi/dai-doi/:id](./dai-doi-get-one.md)
- [PATCH /api/don-vi/dai-doi/:id](./dai-doi-patch.md)
- [DaiDoi Schema](../../../backend/schemas/DaiDoi.md)
