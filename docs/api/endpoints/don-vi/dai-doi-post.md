# POST /api/don-vi/dai-doi

**Endpoint**: `POST /api/don-vi/dai-doi`  
**Authentication**: ✅ Required  

**Roles**: admin  

**Last Verified**: 2026-05-16

---

## Description

Creates a new dai-doi (battalion/squad). **Admin-only operation**.

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
  "ten": "Đại đội 1",
  "khoa": "64a1b2c3d4e5f6a7b8c9d0e1",
  "donViLienKet": "64a1b2c3d4e5f6a7b8c9d0e3",
  "ngayBatDau": "2024-01-15",
  "ngayKetThuc": "2024-12-31",
  "quanSo": 45,
  "canBo": ["64a1b2c3d4e5f6a7b8c9d0e4"],
  "soQD": ["QĐ-001/2024"],
  "ngayQD": ["2024-01-15"],
  "hieuLuc": [
    {
      "batDau": "2024-01-15",
      "ketThuc": "2024-12-31"
    }
  ]
}
```

### Required Fields

- `ten` (string, max 200 chars) - Platoon name
- `khoa` (ObjectId string) - Parent khoa reference

### Optional Fields

- `donViLienKet` (ObjectId string) - Partner institution reference
- `ngayBatDau` (ISO date string) - Platoon start date
- `ngayKetThuc` (ISO date string) - Platoon end date
- `quanSo` (number) - Platoon strength/headcount
- `canBo` (array of ObjectId strings) - Staff member references
- `soQD` (array of strings) - Decision numbers
- `ngayQD` (array of ISO date strings) - Decision dates
- `hieuLuc` (array of objects) - Validity periods with `batDau` and `ketThuc` dates

**IMPORTANT**: Arrays `canBo`, `soQD`, `ngayQD`, and `hieuLuc` must have equal lengths (validated by schema).

---

## Response

### Success (201 Created)

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
    "createdAt": "2025-12-31T10:00:00.000Z",
    "updatedAt": "2025-12-31T10:00:00.000Z"
  }
}
```

---

## Error Responses

### 409 Conflict
A dai-doi with the same combination of `ten`, `khoa`, and `donViLienKet` already exists.

```json
{ "error": { "code": "DUPLICATE_DAIDOI", "message": "..." } }
```

---

## Related

- [GET /api/don-vi/dai-doi](./dai-doi-get-list.md)
- [PATCH /api/don-vi/dai-doi/:id](./dai-doi-patch.md)
- [DaiDoi Schema](../../../backend/schemas/DaiDoi.md)
