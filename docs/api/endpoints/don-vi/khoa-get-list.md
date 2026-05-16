# GET /api/don-vi/khoa

**Endpoint**: `GET /api/don-vi/khoa`  
**Authentication**: ✅ Required  

**Roles**: admin, staff, viewer, teacher  

**Last Verified**: 2026-01-02

---

## Description

Lists khoa (faculties/divisions).

---

## Request

### Headers
```
Authorization: Bearer <access_token>
```

### Query Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| page | number | 1 | Page number (1-indexed) |
| limit | number | 50 | Items per page |

---

## Response

### Success (200 OK)

```json
{
  "data": [
    {
      "_id": "64a1b2c3d4e5f6a7b8c9d0e1",
      "ten": "K58",
      "daiDoi": ["64a1b2c3d4e5f6a7b8c9d0e1"],
      "createdAt": "2023-01-15T00:00:00.000Z",
      "updatedAt": "2023-01-15T00:00:00.000Z"
    }
  ],
  "meta": {
    "page": 1,
    "limit": 50,
    "total": 8
  }
}
```

---

## Notes

- Requires authentication with admin, staff, viewer, or teacher role
- Used for organizational hierarchy reference

---

## Related

- [POST /api/don-vi/khoa](./khoa-post.md)
- [PATCH /api/don-vi/khoa/:id](./khoa-patch.md)
- [Khoa Schema](../../../backend/schemas/Khoa.md)
