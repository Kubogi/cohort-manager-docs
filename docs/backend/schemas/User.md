# User Schema

**Source**: [backend/src/models/User.js](../../../backend/src/models/User.js)

**Collection**: `users` (primary cluster)

**Last verified**: 2026-05-16

---

## Overview

User accounts for authentication and authorization. Uses a complex nested allowedUnits structure for fine-grained unit-level permissions.

---

## Fields

| Field | Type | Required | Unique | Default | Description |
|-------|------|----------|--------|---------|-------------|
| username | String | Yes | Yes | - | Username (lowercase, trimmed) |
| passwordHash | String | Yes | No | - | Hashed password (bcrypt cost 10) |
| role | String | No (default) | No | 'viewer' | One of `admin`, `staff`, `viewer`, `teacher` |
| allowedUnits | IAllowedUnits | No | No | {} | Unit-level scope for staff (see [auth.md](../../architecture/auth.md#5-per-unit-scoping--allowedunits-staff)) |
| teacherScope | TeacherScopeEntry[] | No | No | [] | Per-teacher (khoa, đại đội) entries (see [auth.md](../../architecture/auth.md#6-per-unit-scoping--teacherscope-teacher)) |
| status | String | No (default) | No | 'active' | One of `active`, `disabled` |
| lastLogin | Date | No | No | - | Last successful login timestamp |
| createdAt | Date | Auto | No | - | Document creation timestamp |
| updatedAt | Date | Auto | No | - | Document last update timestamp |

---

## Role Enum

- **admin**: Full system access. Only role that can manage users + system links.
- **staff**: Can read/write within `allowedUnits`. No user management.
- **viewer**: Read-only access to all records their route gate allows. **`allowedUnits` is not applied to the viewer role** — `applyUnitScope` is a no-op for non-staff users, so viewers see the same records an unscoped staff user with `allowAll: true` would see. Verify this is intentional before deploying widely.
- **teacher**: Limited menu; reads gated by `teacherScope` per-(khoa, đại đội). Can edit grades for in-scope students.

---

## Status Enum

- **active**: User can log in
- **disabled**: User cannot log in

---

## AllowedUnits Structure

Complex nested schema for unit-level permissions:

```typescript
{
  daiDoi: string[],      // Array of allowed DaiDoi IDs
  khoa: string[],        // Array of allowed Khoa IDs
  truong: string[],      // Array of allowed DonViLienKet IDs
  allowAll: {
    daiDoi: boolean,     // If true, access ALL daiDoi
    khoa: boolean,       // If true, access ALL khoa
    truong: boolean      // If true, access ALL truong
  }
}
```

### Example

```json
{
  "daiDoi": ["id1", "id2"],
  "khoa": [],
  "truong": [],
  "allowAll": {
    "daiDoi": false,
    "khoa": true,
    "truong": false
  }
}
```

This means:
- User can access daiDoi id1 and id2
- User can access **ALL** khoa (allowAll.khoa = true)
- User has no truong access

`allowedUnits` is only applied to **staff** today — see `applyUnitScope` in [`utils/permissions.js`](../../../backend/src/utils/permissions.js). Viewer and teacher use different mechanisms.

---

## TeacherScope Structure (teacher role only)

Embedded array on the User document. Only relevant when `role === 'teacher'`.

```typescript
teacherScope: Array<{
  khoa: ObjectId | null,    // restricted to this khoa
  daiDoi: ObjectId | null,  // and this đại đội (ANDed)
  allKhoa: boolean,         // wildcard on khoa axis (ignores `khoa`)
  allDaiDoi: boolean        // wildcard on đại đội axis (ignores `daiDoi`)
}>
```

Each entry contributes one `$or` branch in queries; the user can access the union of their entries. An empty array means "no rows for you" (sentinel filter `{ _id: { $in: [] } }`). An entry with both `allKhoa: true` and `allDaiDoi: true` contributes a wildcard.

`teacherScope` is included in the JWT access token for client-side UI gating. Cross-DB validity (every khoa + daiDoi exists) is checked at write time by `validateTeacherScopeAgainstDB` in `services/user.service.js`.

---

## Indexes

| Fields | Type | Unique |
|--------|------|--------|
| username | Single | Yes |

---

## Important Notes

1. **Field name** — `passwordHash` not `password`.
2. **Username** is automatically lowercased and trimmed by Mongoose; never store a mixed-case username.
3. **Two scope mechanisms** — `allowedUnits` (staff) and `teacherScope` (teacher) are independent; admin and viewer use neither.
4. **`allowAll` flags override the arrays** — `allowAll.khoa: true` means "any khoa", regardless of `allowedUnits.khoa` contents.
5. **Empty allowedUnits array + no allowAll** means *no* access on that axis (effectively zero rows).
6. **JWT access-token payload** carries `{ sub, role, allowedUnits, teacherScope? }`. See [`/docs/architecture/auth.md`](../../architecture/auth.md).

---

## API Endpoints

- [`/api/auth`](../../api/endpoints/auth/) — login, register (admin), refresh, logout, change-password
- [`/api/users`](../../api/endpoints/users/) — admin-only user CRUD
