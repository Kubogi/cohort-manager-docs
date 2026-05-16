# POST /api/khoa-hoc

**Endpoint**: `POST /api/khoa-hoc`  

**Authentication**: ✅ Required  

**Roles**: admin  

**Last Verified**: 2026-05-16

---

## Description

Creates a new course/training program. **Admin-only operation**.

---

## Request

### Headers
```
Content-Type: application/json
Authorization: Bearer <access_token>
```

### Body

```json
{
  "ten": "K59",
  "daiDoi": ["64a1b2c3d4e5f6a7b8c9d0e1", "64a1b2c3d4e5f6a7b8c9d0e2"]
}
```

### Required Fields

- `ten` (string, max 200 chars) - Name of the khoa/course

### Optional Fields

- `daiDoi` (array of ObjectId strings) - Array of DaiDoi (platoon) references

---

## Response

### Success (201 Created)

```json
{
  "data": {
    "_id": "507f1f77bcf86cd799439012",
    "ten": "K59",
    "daiDoi": ["64a1b2c3d4e5f6a7b8c9d0e1", "64a1b2c3d4e5f6a7b8c9d0e2"],
    "createdAt": "2025-12-31T10:00:00.000Z",
    "updatedAt": "2025-12-31T10:00:00.000Z"
  }
}
```

---

## Error Responses

### 403 Forbidden
Only admin role can create courses.

### 409 Conflict
A khoa with the same `ten` already exists.

```json
{ "error": { "code": "DUPLICATE_KHOA", "message": "..." } }
```

---

## Related

- [PATCH /api/khoa-hoc/:id](./patch.md)
- [GET /api/khoa-hoc](./get-list.md)
- [Khoa Schema](../../../backend/schemas/Khoa.md)
