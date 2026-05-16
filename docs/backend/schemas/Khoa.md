# Khoa (Course/Department) Schema

**Source**: [backend/src/models/Khoa.js](../../backend/src/models/Khoa.js)

**Collection**: `khoa`

**Last Verified**: 2026-05-16

---

## Overview

Represents a Khoa (course/department/cohort) which contains multiple DaiDoi (platoons).

**IMPORTANT**: Only has 2 fields - ten (name) and daiDoi (array of platoon refs). Does NOT have maKhoa or moTa fields.

---

## Fields

| Field | Type | Required | Unique | Description |
|-------|------|----------|--------|-------------|
| ten | String | No | Yes* | Name of khoa/course/cohort (e.g. "K58", "Khóa 2023-2025") |
| daiDoi | ObjectId[] | No | No | Array of DaiDoi references |
| createdAt | Date | Auto | No | Document creation timestamp |
| updatedAt | Date | Auto | No | Document last update timestamp |

\* Unique constraint with sparse index on `ten`

---

## Indexes

| Fields | Type | Unique | Sparse |
|--------|------|--------|--------|
| ten | Single | Yes | Yes |

---

## Relationships

- **daiDoi** → DaiDoi model (One-to-Many) - Array of platoon references

---

## Organizational Hierarchy

```
Truong (Partner Institution)
  └── Khoa (Course/Department) <- THIS LEVEL
      └── DaiDoi (Platoon)
          └── SinhVien (Students)
```

---

## Important Notes

1. **Only 2 Fields**: Schema has only `ten` and `daiDoi`
2. **All Optional**: All fields are optional in schema definition
3. **Unique Name**: `ten` is unique when present (sparse index)
4. **Array Reference**: `daiDoi` is an array of ObjectId references, not single reference
5. **Used as Organizational Level**: Above DaiDoi (platoon)

---

## Migration from Hallucinated Documentation

**FIELDS THAT DO NOT EXIST**:
- ❌ maKhoa (course code/ID) - Does not exist, only has ten (name)
- ❌ moTa (description) - Not in schema
- ❌ tenKhoa (course name) - Actual field is just "ten", not "tenKhoa"

**ACTUAL FIELDS**:
- ✅ ten (name)
- ✅ daiDoi (array of ObjectId references)

---

## API Endpoints

**Base Path**: `/api/khoa-hoc` ⚠️ **NOT** `/api/khoa`

- `GET /` - List khoa/courses
- `GET /:id` - Get khoa by ID
- `POST /` - Create new khoa
- `PATCH /:id` - Update khoa (**NOT PUT**)
- `DELETE /:id` - Delete khoa

**Alternative Path**: `/api/don-vi/khoa` — handled by `donVi.controller.js`, which exposes list/create/update/delete but does **NOT** expose `GET /khoa/:id`. By contrast, `/api/khoa-hoc` is handled by `khoa.controller.js` and exposes full CRUD including `GET /:id`. These are separate controllers with different route sets and are not interchangeable.
