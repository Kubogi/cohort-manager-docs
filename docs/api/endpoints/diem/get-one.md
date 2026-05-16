# GET /api/diem/:id

**Endpoint**: `GET /api/diem/:id`  
**Authentication**: ✅ Required  

**Roles**: admin, staff, viewer, teacher
**Unit Scoping**: ✅ Applied (staff: `allowedUnits`; teacher: `teacherScope`, via parent SinhVien)  

**Last Verified**: 2025-12-31

---

## Description

Retrieves a single grade by subdocument ID. **Unit scoping applied via parent student**.

---

## Request

### Headers
```
Authorization: Bearer <access_token>
```

### URL Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| id | string | Yes | Grade subdocument _id |

---

## Response

### Success (200 OK)

```json
{
  "data": {
    "_id": "64a1b2c3d4e5f6a7b8c9d0e4",
    "sinhVien": "64a1b2c3d4e5f6a7b8c9d0e1",
    "mon": "Chính trị 1",
    "thuongXuyen": 8,
    "mieng": 7,
    "giuaHP": 8,
    "hetHP": 9,
    "tbMon": 8.5
  }
}
```

> `createdAt`/`updatedAt` exist on the embedded subdocument but are **not projected** by this endpoint.
```

---

## Error Responses

### 404 Not Found
Grade doesn't exist OR user doesn't have access to parent student.

---

## Related

- [GET /api/diem](./get-list.md)
- [PATCH /api/diem/:id](./patch.md)
- [SinhVien Schema](../../../backend/schemas/SinhVien.md)
