# GET /api/khoa-hoc/:id

**Endpoint**: `GET /api/khoa-hoc/:id`  

**Authentication**: ✅ Required  

**Roles**: admin, staff, viewer, teacher  

**Unit Scoping**: ✅ Applied (staff: `allowedUnits`; teacher: `teacherScope`)  

**Last Verified**: 2026-05-16

---

## Description

Retrieves a single course by ID.

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

### Success (200 OK)

```json
{
  "data": {
    "_id": "507f1f77bcf86cd799439011",
    "ten": "K58",
    "daiDoi": ["64a1b2c3d4e5f6a7b8c9d0e1"],
    "createdAt": "2023-01-15T00:00:00.000Z",
    "updatedAt": "2023-01-15T00:00:00.000Z"
  }
}
```

---

## Error Responses

### 404 Not Found
Course doesn't exist.

---

## Related

- [GET /api/khoa-hoc](./get-list.md)
- [PATCH /api/khoa-hoc/:id](./patch.md)
- [Khoa Schema](../../../backend/schemas/Khoa.md)
