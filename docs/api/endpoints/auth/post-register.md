# POST /api/auth/register

**Endpoint**: `POST /api/auth/register`  
**Authentication**: ✅ Required  

**Roles**: admin  

**Source**: [auth.route.js](../../../../backend/src/routes/auth.route.js), [auth.controller.js](../../../../backend/src/controllers/auth.controller.js)  

**Last Verified**: 2026-05-16

---

## Description

Registers a new user account. **Admin-only operation**.

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
  "username": "string (required)",
  "password": "string (required)",
  "role": "string (optional, default: 'viewer')",
  "allowedUnits": {
    "daiDoi": ["string"],
    "khoa": ["string"],
    "truong": ["string"],
    "allowAll": {
      "daiDoi": false,
      "khoa": false,
      "truong": false
    }
  }
}
```

### Body Parameters

| Parameter | Type | Required | Default | Enum | Description |
|-----------|------|----------|---------|------|-------------|
| username | string | Yes | - | - | Username (will be lowercased and trimmed). Min 3 chars, max 50 chars. |
| password | string | Yes | - | - | User password (plain text, will be hashed). Min 6 chars. |
| role | string | No | 'viewer' | 'admin', 'staff', 'viewer', 'teacher' | User role |
| teacherScope | array | No | [] | - | Required when `role` is `'teacher'`. Array of `{ khoa, daiDoi, allKhoa, allDaiDoi }` access entries. Each entry specifies one `(khoa, daiDoi)` pair the teacher can access; use `allKhoa: true` or `allDaiDoi: true` for wildcard axes. |
| allowedUnits | object | No | {} | - | Unit-level permissions (applies to `staff` role only) |
| allowedUnits.daiDoi | string[] | No | [] | - | Array of DaiDoi IDs user can access |
| allowedUnits.khoa | string[] | No | [] | - | Array of Khoa IDs user can access |
| allowedUnits.truong | string[] | No | [] | - | Array of DonViLienKet IDs user can access |
| allowedUnits.allowAll | object | No | {} | - | Allow all flags for each unit type |
| allowedUnits.allowAll.daiDoi | boolean | No | false | - | If true, user can access ALL daiDoi |
| allowedUnits.allowAll.khoa | boolean | No | false | - | If true, user can access ALL khoa |
| allowedUnits.allowAll.truong | boolean | No | false | - | If true, user can access ALL truong |

---

## Response

### Success Response (201 Created)

```json
{
  "data": {
    "id": "507f1f77bcf86cd799439011",
    "username": "newuser",
    "role": "staff"
  }
}
```

### Response Fields

| Field | Type | Description |
|-------|------|-------------|
| data.id | string | Created user ID |
| data.username | string | Username (lowercase) |
| data.role | string | User role |

**Note**: Does NOT return password or tokens. User must login separately.

---

## Error Responses

### 401 Unauthorized - Not Authenticated

```json
{
  "error": {
    "code": "UNAUTHORIZED",
    "message": "Missing or invalid authentication token"
  }
}
```

**When**: No Authorization header or invalid token

### 403 Forbidden - Not Admin

```json
{
  "error": {
    "code": "FORBIDDEN",
    "message": "Insufficient permissions"
  }
}
```

**When**: Authenticated user does not have 'admin' role

### 409 Conflict - Username Exists

```json
{
  "error": {
    "code": "USER_EXISTS",
    "message": "Username already taken"
  }
}
```

**When**: Username already exists in database

### 400 Bad Request - Validation Error

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Validation failed",
    "details": [
      {
        "field": "role",
        "message": "\"role\" must be one of [admin, staff, viewer, teacher]"
      }
    ]
  }
}
```

**When**: Invalid field values or format

---

## Notes

1. **Admin Only**: Only users with 'admin' role can create new users
2. **Password Hashing**: Password is hashed with bcrypt before storage (field: passwordHash)
3. **Username Normalization**: Username is automatically lowercased and trimmed
4. **Default Role**: If not specified, defaults to 'viewer'
5. **Empty AllowedUnits**: If not specified, user starts with no unit access (except admins have implicit full access)
6. **AllowAll Flags**: Override corresponding arrays (if allowAll.khoa = true, khoa[] is ignored)
7. **No Auto-Login**: User is created but NOT logged in - must call /login separately

---

## Example Request

### Create Staff User with Limited Access

```bash
curl -X POST https://api.example.com/api/auth/register \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <admin_token>" \
  -d '{
    "username": "staffuser",
    "password": "SecurePass123",
    "role": "staff",
    "allowedUnits": {
      "daiDoi": ["64a1b2c3d4e5f6a7b8c9d0e1"],
      "khoa": [],
      "truong": [],
      "allowAll": {
        "daiDoi": false,
        "khoa": true,
        "truong": false
      }
    }
  }'
```

This creates a staff user who can:
- Access specific daiDoi: 64a1b2c3d4e5f6a7b8c9d0e1
- Access ALL khoa (allowAll.khoa = true)
- No truong access

### Create Teacher User with Scoped Access

```bash
curl -X POST https://api.example.com/api/auth/register \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <admin_token>" \
  -d '{
    "username": "teacher01",
    "password": "SecurePass123",
    "role": "teacher",
    "teacherScope": [
      { "khoa": "64a1b2c3d4e5f6a7b8c9d0e1", "daiDoi": "64a1b2c3d4e5f6a7b8c9d0e2", "allKhoa": false, "allDaiDoi": false },
      { "khoa": "64a1b2c3d4e5f6a7b8c9d0e1", "daiDoi": null, "allKhoa": false, "allDaiDoi": true }
    ]
  }'
```

The teacher can access battalion D2 within khoa K1, and all battalions within khoa K1.

---

## Related Endpoints

- [POST /api/auth/login](./post-login.md) - Login user
- [POST /api/auth/refresh](./post-refresh.md) - Refresh access token
- [POST /api/auth/logout](./post-logout.md) - Logout user

---

## Schema Reference

See [User Schema](../../../backend/schemas/User.md) for complete User model documentation.
