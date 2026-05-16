# GET /api/diem/transcripts

**Endpoint**: `GET /api/diem/transcripts`
**Authentication**: ✅ Required

**Roles**: admin, staff, viewer, teacher

**Last Verified**: 2026-05-16

**Unit Scoping**: ❌ Not applied (this endpoint queries by student identity, not by unit)

---

## Description

Returns the grade transcript for a specific student. The student is identified by one of two lookup strategies: `(hoTen + ngaySinh)` for name/date-of-birth lookup, or `(maSinhVien + donViLienKet)` for student-ID/school lookup.

---

## Request

### Headers
```
Authorization: Bearer <access_token>
```

### Query Parameters

Exactly one of the following lookup pairs is required. If neither pair is fully provided, the endpoint returns `400 INVALID_QUERY`.

**Lookup by name and date of birth:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| hoTen | string | Yes (for this pair) | Student full name (exact match). |
| ngaySinh | string | Yes (for this pair) | Date of birth. Parsed as a JavaScript `Date` — accepts ISO 8601 and common date formats. |

**Lookup by student ID and school:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| maSinhVien | string | Yes (for this pair) | Student ID (`maSV` field). |
| donViLienKet | string | Yes (for this pair) | Partner school ObjectId (`truong` field). |

**Pagination:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| page | number | 1 | Page number (1-indexed). Applied to the student's grade array. |
| limit | number | 50 | Items per page. |

---

## Response

### Success (200 OK)

```json
{
  "data": [
    {
      "_id": "64a1b2c3d4e5f6a7b8c9d0e4",
      "sinhVien": "64a1b2c3d4e5f6a7b8c9d0e1",
      "mon": "Chính trị 1",
      "thuongXuyen": 8,
      "mieng": 7,
      "giuaHP": 8,
      "hetHP": 9,
      "tbMon": 8.5
    }
  ],
  "meta": {
    "total": 5,
    "page": 1,
    "limit": 50
  }
}
```

Each item is a single grade row (one subject). The `sinhVien` field is the parent student's ObjectId. Note that `createdAt`/`updatedAt` are **not** projected.

---

## Error Responses

### 400 Bad Request — missing lookup pair

```json
{
  "error": {
    "code": "INVALID_QUERY",
    "message": "Provide hoTen + ngaySinh or maSinhVien + donViLienKet"
  }
}
```

**When**: Neither valid lookup pair is provided.

### 404 Not Found — student not found

```json
{
  "error": {
    "code": "STUDENT_NOT_FOUND",
    "message": "Student not found for transcript lookup"
  }
}
```

**When**: No student matches the supplied lookup parameters.

---

## Notes

- No unit scoping is applied — any authenticated role can look up transcripts by identity.
- `tongDiem` and `diemTB` are **not** returned; compute them client-side if needed.

---

## Related

- [GET /api/diem](./get-list.md)
- [GET /api/diem/summary](./get-summary.md)
- [SinhVien Schema](../../../backend/schemas/SinhVien.md)
