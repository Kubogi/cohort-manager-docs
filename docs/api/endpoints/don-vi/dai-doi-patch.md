# PATCH /api/don-vi/dai-doi/:id

**Endpoint**: `PATCH /api/don-vi/dai-doi/:id`  
**Authentication**: ✅ Required  
**Roles**: admin  
**HTTP Method**: **PATCH** (not PUT)  
**Last Verified**: 2026-05-16

---

## Description

Updates an existing dai-doi. **Admin-only operation**. Partial updates supported.

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
| id | string | Yes | DaiDoi MongoDB ObjectId |

### Body

```json
{
  "ten": "Đại đội 1 - Updated",
  "quanSo": 48,
  "ngayKetThuc": "2025-06-30",
  "canBo": ["64a1b2c3d4e5f6a7b8c9d0e4", "64a1b2c3d4e5f6a7b8c9d0e5"],
  "soQD": ["QĐ-001/2024", "QĐ-002/2024"],
  "ngayQD": ["2024-01-15", "2024-06-01"],
  "hieuLuc": [
    {
      "batDau": "2024-01-15",
      "ketThuc": "2024-05-31"
    },
    {
      "batDau": "2024-06-01",
      "ketThuc": "2025-06-30"
    }
  ],
  "tkpk": ["Trưởng khung", "Phó khung"],
  "ghiChu": ["Phụ trách khung D2", ""]
}
```

### Updatable Fields

- `ten` (string, max 200 chars)
- `khoa` (ObjectId string)
- `donViLienKet` (ObjectId string)
- `ngayBatDau` (ISO date string)
- `ngayKetThuc` (ISO date string)
- `quanSo` (number)
- `canBo` (array of ObjectId strings)
- `soQD` (array of strings)
- `ngayQD` (array of ISO date strings)
- `hieuLuc` (array of validity period objects)
- `tkpk` (array of strings) - per-assignment TK/PK role (`""` | `"Trưởng khung"` | `"Phó khung"`)
- `ghiChu` (array of strings, max 500 chars each) - per-assignment notes

**Note**: When updating parallel arrays, all six arrays (canBo, soQD, ngayQD, hieuLuc, tkpk, ghiChu) must maintain equal lengths.

---

## Response

### Success (200 OK)

```json
{
  "data": {
    "_id": "64a1b2c3d4e5f6a7b8c9d0e2",
    ...updated fields...,
    "updatedAt": "2025-12-31T10:00:00.000Z"
  }
}
```

---

## Related

- [GET /api/don-vi/dai-doi/:id](./dai-doi-get-one.md)
- [DELETE /api/don-vi/dai-doi/:id](./dai-doi-delete.md)
- [DaiDoi Schema](../../../backend/schemas/DaiDoi.md)
