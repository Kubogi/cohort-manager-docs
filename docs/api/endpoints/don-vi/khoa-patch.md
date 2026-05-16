# PATCH /api/don-vi/khoa/:id

**Endpoint**: `PATCH /api/don-vi/khoa/:id`  
**Authentication**: ✅ Required  

**Roles**: admin  
**HTTP Method**: **PATCH** (not PUT)  

**Last Verified**: 2025-12-31

---

## Description

Updates an existing khoa. **Admin-only operation**. Partial updates supported.

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

```json
{
  "ten": "K58 - Updated",
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
    "_id": "64a1b2c3d4e5f6a7b8c9d0e1",
    "ten": "K58 - Updated",
    "daiDoi": ["64a1b2c3d4e5f6a7b8c9d0e1", "64a1b2c3d4e5f6a7b8c9d0e2"],
    "createdAt": "2023-01-15T00:00:00.000Z",
    "updatedAt": "2025-12-31T10:00:00.000Z"
  }
}
```

---

## Related

- [GET /api/don-vi/khoa](./khoa-get-list.md)
- [DELETE /api/don-vi/khoa/:id](./khoa-delete.md)
- [Khoa Schema](../../../backend/schemas/Khoa.md)
