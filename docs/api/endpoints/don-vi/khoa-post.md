# POST /api/don-vi/khoa

**Endpoint**: `POST /api/don-vi/khoa`  
**Authentication**: ✅ Required  

**Roles**: admin  

**Last Verified**: 2025-12-31

---

## Description

Creates a new khoa (faculty/division). **Admin-only operation**.

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
  "ten": "K58",
  "daiDoi": ["64a1b2c3d4e5f6a7b8c9d0e1"]
}
```

### Required Fields

- `ten` (string, max 200 chars) - Name of the khoa

### Optional Fields

- `daiDoi` (array of ObjectId strings) - Array of DaiDoi references

---

## Response

### Success (201 Created)

```json
{
  "data": {
    "_id": "64a1b2c3d4e5f6a7b8c9d0e1",
    "ten": "K58",
    "daiDoi": ["64a1b2c3d4e5f6a7b8c9d0e1"],
    "createdAt": "2025-12-31T10:00:00.000Z",
    "updatedAt": "2025-12-31T10:00:00.000Z"
  }
}
```

---

## Error Responses

### 409 Conflict — Duplicate name

A khoa with the same `ten` already exists.

```json
{ "error": "DUPLICATE_TEN" }
```

### 409 Conflict — DaiDoi already assigned

One or more `daiDoi` ObjectIds in the body are already assigned to another khoa.

```json
{ "error": "DAIDOI_ALREADY_ASSIGNED" }
```

---

## Related

- [GET /api/don-vi/khoa](./khoa-get-list.md)
- [PATCH /api/don-vi/khoa/:id](./khoa-patch.md)
- [Khoa Schema](../../../backend/schemas/Khoa.md)
