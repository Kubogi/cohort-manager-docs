# GET /api/don-vi/don-vi-lien-ket/:id

**Endpoint**: `GET /api/don-vi/don-vi-lien-ket/:id`
**Authentication**: ✅ Required
**Roles**: admin, staff, viewer, teacher
**Unit Scoping**: ✅ Applied (staff: `allowedUnits.truong` via `applyUnitScope`)
**Last Verified**: 2026-05-22

---

## Description

Retrieves a single `DonViLienKet` (linked partner school) by ID. Backs the frontend supplement path `dataProvider.getMany('don-vi/don-vi-lien-ket', { ids })`, which surfaces school names in báo cáo / thống kê / hồ sơ sức khỏe when the school is past the 200-row catalog cap of `useUnitLookups`.

Document lives on the secondary cluster (see [DonViLienKet schema](../../../backend/schemas/DonViLienKet.md)).

---

## Request

### Headers
```
Authorization: Bearer <access_token>
```

### URL Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| id | string | Yes | DonViLienKet ObjectId (secondary cluster) |

---

## Response

### Success (200 OK)

```json
{
  "data": {
    "_id": "64a1b2c3d4e5f6a7b8c9d0e3",
    "ten": "Đại học Công nghệ",
    "heDaoTao": "Đại học"
  }
}
```

---

## Error Responses

### 404 Not Found
Returned when the id is malformed, has no matching document, or the caller's `applyUnitScope` excludes the document. 404 is used uniformly (not 400/403) to avoid leaking existence information across scope boundaries.

---

## Related

- [GET /api/don-vi/don-vi-lien-ket](./don-vi-lien-ket-get-list.md)
- [DonViLienKet Schema](../../../backend/schemas/DonViLienKet.md)
