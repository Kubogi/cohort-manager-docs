# GET /api/khoa-hoc

**Endpoint**: `GET /api/khoa-hoc`  

**Authentication**: ✅ Required  

**Roles**: admin, staff, viewer, teacher

**Unit Scoping**: ✅ Applied (staff: `allowedUnits`; teacher: `teacherScope`)

**Last Verified**: 2026-05-16

---

## Description

Lists courses/training programs (khóa học) with pagination.

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
| limit | number | 20 | Items per page |

---

## Response

### Success (200 OK)

```json
{
  "data": [
    {
      "_id": "507f1f77bcf86cd799439011",
      "ten": "K58",
      "daiDoi": ["64a1b2c3d4e5f6a7b8c9d0e1"],
      "createdAt": "2023-01-15T00:00:00.000Z",
      "updatedAt": "2023-01-15T00:00:00.000Z"
    }
  ],
  "meta": {
    "page": 1,
    "limit": 20,
    "total": 12
  }
}
```

---

## Related

- [GET /api/khoa-hoc/:id](./get-one.md)
- [POST /api/khoa-hoc](./post.md)
- [Khoa Schema](../../../backend/schemas/Khoa.md)
