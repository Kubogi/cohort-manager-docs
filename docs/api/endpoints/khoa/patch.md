# PATCH /api/khoa-hoc/:id

**Endpoint**: `PATCH /api/khoa-hoc/:id`  

**Authentication**: ✅ Required  

**Roles**: admin  

**HTTP Method**: **PATCH** (not PUT)  

**Last Verified**: 2026-05-16

---

## Description

Updates an existing course. **Admin-only operation**. Partial updates supported.

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
| id | string | Yes | Khoa MongoDB ObjectId |

### Body

Send only the fields you want to update:

```json
{
  "ten": "K59 - Updated",
  "daiDoi": ["64a1b2c3d4e5f6a7b8c9d0e1", "64a1b2c3d4e5f6a7b8c9d0e2"]
}
```

### Updatable Fields

- `ten` (string, max 200 chars)
- `daiDoi` (array of ObjectId strings)

---

## Response

### Success (200 OK)

```json
{
  "data": {
    "_id": "507f1f77bcf86cd799439011",
    ...updated course fields...,
    "updatedAt": "2025-12-31T10:00:00.000Z"
  }
}
```

---

## Error Responses

### 404 Not Found
Course doesn't exist.

### 403 Forbidden
Only admin role can update courses.

---

## Related

- [GET /api/khoa-hoc/:id](./get-one.md)
- [DELETE /api/khoa-hoc/:id](./delete.md)
- [Khoa Schema](../../../backend/schemas/Khoa.md)
