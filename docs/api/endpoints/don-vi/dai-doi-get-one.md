# GET /api/don-vi/dai-doi/:id

**Endpoint**: `GET /api/don-vi/dai-doi/:id`  
**Authentication**: ✅ Required  

**Roles**: admin, staff, viewer, teacher  
**Unit Scoping**: ✅ Applied (staff: `allowedUnits`; teacher: `teacherScope`)  

**Last Verified**: 2026-01-02

---

## Description

Retrieves a single dai-doi by ID.

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

### Success (200 OK)

```json
{
  "data": {
    "_id": "64a1b2c3d4e5f6a7b8c9d0e2",
    "ten": "Đại đội 1",
    "khoa": "64a1b2c3d4e5f6a7b8c9d0e1",
    "donViLienKet": "64a1b2c3d4e5f6a7b8c9d0e3",
    "ngayBatDau": "2024-01-15T00:00:00.000Z",
    "ngayKetThuc": "2024-12-31T00:00:00.000Z",
    "quanSo": 45,
    "canBo": ["64a1b2c3d4e5f6a7b8c9d0e4"],
    "soQD": ["QĐ-001/2024"],
    "ngayQD": ["2024-01-15T00:00:00.000Z"],
    "hieuLuc": [
      {
        "batDau": "2024-01-15T00:00:00.000Z",
        "ketThuc": "2024-12-31T00:00:00.000Z"
      }
    ],
    "createdAt": "2023-01-15T00:00:00.000Z",
    "updatedAt": "2023-01-15T00:00:00.000Z"
  }
}
```

---

## Error Responses

### 404 Not Found
Dai-doi doesn't exist.

---

## Related

- [GET /api/don-vi/dai-doi](./dai-doi-get-list.md)
- [PATCH /api/don-vi/dai-doi/:id](./dai-doi-patch.md)
- [DaiDoi Schema](../../../backend/schemas/DaiDoi.md)
