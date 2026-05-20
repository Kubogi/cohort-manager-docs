# POST /api/don-vi/dai-doi

**Endpoint**: `POST /api/don-vi/dai-doi`
**Authentication**: ✅ Required
**Roles**: admin
**Last Verified**: 2026-05-19

---

## Description

Creates a new dai-doi (battalion/squad). **Admin-only operation**.

Note: As of May 2026, CBQL ↔ DaiDoi assignments are stored on `CanBoQuanLy.phanCong[]`, not on DaiDoi. This endpoint no longer accepts the legacy `canBo / soQD / ngayQD / hieuLuc / tkpk / ghiChu` arrays — Joi's `stripUnknown` drops them silently. To attach a CBQL, PATCH the CBQL with the new `phanCong[]` entry.

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
  "quanSo": 45
}
```

### Required Fields

- `ten` (string, max 200 chars) — Platoon name
- `khoa` (ObjectId string) — Parent khoa reference
- `donViLienKet` (ObjectId string) — Partner institution reference

### Optional Fields

- `ngayBatDau` (ISO date string) — Platoon start date
- `ngayKetThuc` (ISO date string) — Platoon end date
- `quanSo` (number) — Platoon strength/headcount

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
- [CanBoQuanLy Schema](../../../backend/schemas/CanBoQuanLy.md) — assignment fields live here now
