# PATCH /api/don-vi/dai-doi/:id

**Endpoint**: `PATCH /api/don-vi/dai-doi/:id`
**Authentication**: ✅ Required
**Roles**: admin
**HTTP Method**: **PATCH** (not PUT)
**Last Verified**: 2026-05-19

---

## Description

Updates an existing dai-doi. **Admin-only operation**. Partial updates supported.

Note: As of May 2026, CBQL ↔ DaiDoi assignments are stored on `CanBoQuanLy.phanCong[]`, not on DaiDoi. This endpoint no longer accepts the legacy `canBo / soQD / ngayQD / hieuLuc / tkpk / ghiChu` arrays — Joi's `stripUnknown` drops them silently. To attach/detach a CBQL, PATCH the CBQL with the new `phanCong[]` entry.

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
  "ngayKetThuc": "2025-06-30"
}
```

### Updatable Fields

- `ten` (string, max 200 chars)
- `khoa` (ObjectId string)
- `donViLienKet` (ObjectId string)
- `ngayBatDau` (ISO date string)
- `ngayKetThuc` (ISO date string)
- `quanSo` (number)

---

## Response

### Success (200 OK)

```json
{
  "data": {
    "_id": "64a1b2c3d4e5f6a7b8c9d0e2",
    "...updated fields...": "...",
    "updatedAt": "2025-12-31T10:00:00.000Z"
  }
}
```

---

## Related

- [GET /api/don-vi/dai-doi/:id](./dai-doi-get-one.md)
- [DELETE /api/don-vi/dai-doi/:id](./dai-doi-delete.md)
- [DaiDoi Schema](../../../backend/schemas/DaiDoi.md)
- [CanBoQuanLy Schema](../../../backend/schemas/CanBoQuanLy.md) — assignment fields live here now
