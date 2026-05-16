# GET /api/don-vi/don-vi-lien-ket

**Endpoint**: `GET /api/don-vi/don-vi-lien-ket`  
**Authentication**: ✅ Required  

**Roles**: admin, staff, viewer, teacher  

**Last Verified**: 2026-05-16

---

## Description

Lists partner institutions/schools (đơn vị liên kết). Records are read from a **secondary database connection** (`MONGO_URI2`).

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

---

## Response

### Success (200 OK)

```json
{
  "data": [
    {
      "_id": "64a1b2c3d4e5f6a7b8c9d0e3",
      "ten": "Trường Cao đẳng Kỹ thuật",
      "heDaoTao": "Cao đẳng"
    }
  ],
  "meta": {
    "page": 1,
    "limit": 50,
    "total": 12
  }
}
```

### Response Fields

| Field | Type | Description |
|-------|------|-------------|
| ten | string | Institution name (aliased to `ten_truong` in the secondary DB) |
| heDaoTao | string | Training system (aliased to `he_dao_tao` in the secondary DB) |

**Note**: `createdAt` and `updatedAt` are **not** present — the model uses `timestamps: false`.

---

## Notes

- No write operations (POST/PATCH/DELETE) are exposed for this resource — it is managed directly in the secondary database.

---

## Related

- [DonViLienKet Schema](../../../backend/schemas/DonViLienKet.md)
