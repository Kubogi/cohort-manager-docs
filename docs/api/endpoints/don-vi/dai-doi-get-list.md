# GET /api/don-vi/dai-doi

**Endpoint**: `GET /api/don-vi/dai-doi`  
**Authentication**: ✅ Required  

**Roles**: admin, staff, viewer, teacher  
**Unit Scoping**: ✅ Applied (staff: `allowedUnits`; teacher: `teacherScope`)  

**Last Verified**: 2026-01-02

---

## Description

Lists dai-doi (battalions/squads).

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
| khoa | string | - | Filter by parent Khoa ID |
| khoaId | string | - | Alias for khoa parameter |
| canBo | string | - | Filter by staff member ObjectId |
| giangVien | string | - | Alias for canBo parameter |

---

## Response

### Success (200 OK)

```json
{
  "data": [
    {
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
  ],
  "meta": {
    "page": 1,
    "limit": 50,
    "total": 25
  }
}
```

---

## Related

- [GET /api/don-vi/dai-doi/:id](./dai-doi-get-one.md)
- [POST /api/don-vi/dai-doi](./dai-doi-post.md)
- [DaiDoi Schema](../../../backend/schemas/DaiDoi.md)
