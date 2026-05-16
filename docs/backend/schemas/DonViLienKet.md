# DonViLienKet (Partner Institution) Schema

**Source**: [backend/src/models/DonViLienKet.js](../../backend/src/models/DonViLienKet.js)

**Collection**: `schools` (in secondary database)

**Database**: MONGO_URI2 (SECONDARY CONNECTION)

**Last Verified**: 2026-05-16

---

## Overview

**CRITICAL**: This model uses a SEPARATE DATABASE CONNECTION (MONGO_URI2).

It reads from an existing "schools" collection in a partner database. Field names in code use aliases to map to existing database column names.

---

## Fields

| Field (Code) | Field (DB) | Type | Description |
|--------------|------------|------|-------------|
| ten | ten_truong | String | Institution name |
| heDaoTao | he_dao_tao | String | Training system/program level |

**No timestamps** (timestamps: false)

---

## Field Aliases

The schema uses Mongoose aliases to map cleaner field names to existing database columns:

- **In code**: `ten`  
  **In database**: `ten_truong`

- **In code**: `heDaoTao`  
  **In database**: `he_dao_tao`

---

## Database Connection

Uses mongoose.createConnection(process.env.MONGO_URI2)

**SEPARATE** from main database connection (MONGO_URI)

**Application will crash** if MONGO_URI2 environment variable is not set.

---

## heDaoTao Examples

Typical values (not enforced by enum):
- **Đại học** - University
- **Cao đẳng** - College  
- **Trung cấp** - Vocational

---

## Important Notes

1. **Minimal Schema**: Only 2 fields by design
2. **Secondary Database**: Uses MONGO_URI2, not primary MONGO_URI
3. **Collection Name**: `schools`, not `don_vi_lien_ket`
4. **No Timestamps**: timestamps: false
5. **Field Aliases**: Code uses ten/heDaoTao, database has ten_truong/he_dao_tao
6. **All Optional**: All fields are optional in schema definition
7. **Likely Read-Only**: References external database owned by another system
8. **Environment Required**: MONGO_URI2 must be set or application crashes

---

## Relationships

Referenced by:
- **SinhVien.truong** (which students come from which partner schools)
- **DaiDoi.donViLienKet** (organizational hierarchy)
- **HoSoSucKhoe.truong** (unit scoping)

---

## Organizational Hierarchy

```
Truong (Partner Institution) <- THIS LEVEL (External DB)
  └── Khoa (Course/Department)
      └── DaiDoi (Platoon)
          └── SinhVien (Students)
```

---

## Migration from Hallucinated Documentation

**FIELDS THAT DO NOT EXIST**:
- ❌ tenDonVi (unit name) - Actual field is "ten" (aliased to ten_truong)
- ❌ loaiDonVi (unit type) - Does not exist
- ❌ diaChi (address) - Does not exist
- ❌ nguoiLienHe (contact person) - Does not exist
- ❌ soDienThoai (phone) - Does not exist
- ❌ email - Does not exist
- ❌ ghiChu (notes) - Does not exist

**ACTUAL STRUCTURE**:
- Only 2 fields: ten (aliased), heDaoTao (aliased)
- Separate database connection
- Collection name "schools"
- No timestamps

---

## API Endpoints

**Base Path**: `/api/don-vi/don-vi-lien-ket`

- `GET /don-vi-lien-ket` - List partner institutions (from secondary DB)

**Note**: Likely read-only - no create/update/delete operations documented.

---

## Environment Setup

Requires MONGO_URI2 environment variable:

```
MONGO_URI2=mongodb://localhost:27017/partner_schools_db
```
