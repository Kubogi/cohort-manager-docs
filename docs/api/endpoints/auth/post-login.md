# POST /api/auth/login

**Endpoint**: `POST /api/auth/login`  
**Authentication**: ❌ Not Required (Public Endpoint)  

**Roles**: N/A  

**Source**: [auth.route.js](../../../../backend/src/routes/auth.route.js), [auth.controller.js](../../../../backend/src/controllers/auth.controller.js)  

**Last Verified**: 2026-05-16

---

## Description

Authenticates a user with username and password, returning JWT access and refresh tokens.

---

## Request

### Headers

```
Content-Type: application/json
```

### Body

```json
{
  "username": "string (required)",
  "password": "string (required)"
}
```

### Body Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| username | string | Yes | Username (case-insensitive, will be lowercased) |
| password | string | Yes | User password (plain text) |

---

## Response

### Success Response (200 OK)

```json
{
  "data": {
    "accessToken": "eyJhbGciOiJIUzI1NiIs...",
    "refreshToken": "eyJhbGciOiJIUzI1NiIs...",
    "user": {
      "id": "507f1f77bcf86cd799439011",
      "username": "admin",
      "role": "admin",
      "allowedUnits": {
        "daiDoi": [],
        "khoa": [],
        "truong": [],
        "allowAll": {
          "daiDoi": true,
          "khoa": true,
          "truong": true
        }
      }
    }
  }
}
```

### Response Fields

| Field | Type | Description |
|-------|------|-------------|
| data.accessToken | string | JWT access token (short-lived, for API requests) |
| data.refreshToken | string | JWT refresh token (long-lived, for getting new access tokens) |
| data.user.id | string | User ID |
| data.user.username | string | Username (lowercase) |
| data.user.role | string | User role: `'admin'`, `'staff'`, `'viewer'`, or `'teacher'` |
| data.user.allowedUnits | object | Unit-level permissions structure |
| data.user.teacherScope | array | (teacher role only) List of `(khoa, daiDoi)` access pairs. Absent for non-teacher roles. |

### JWT Payload Structure

**Access Token** contains:
```json
{
  "sub": "507f1f77bcf86cd799439011",
  "role": "admin",
  "allowedUnits": { ... },
  "iat": 1704067200,
  "exp": 1704070800
}
```

> `username` is **not** included in the JWT payload. Read `data.user.username` from the login response body instead.
>
> For `teacher` role tokens, an additional `teacherScope` array is present in the payload:
> ```json
> {
>   "sub": "...",
>   "role": "teacher",
>   "allowedUnits": { ... },
>   "teacherScope": [{ "khoa": "<id>|null", "daiDoi": "<id>|null", "allKhoa": false, "allDaiDoi": false }],
>   "iat": ..., "exp": ...
> }
> ```

---

## Error Responses

### 401 Unauthorized - Invalid Credentials

```json
{
  "error": {
    "code": "INVALID_CREDENTIALS",
    "message": "Tên đăng nhập hoặc mật khẩu không đúng"
  }
}
```

**When**: Username doesn't exist, password is incorrect, or user status is 'disabled'

### 400 Bad Request - Validation Error

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Validation failed",
    "details": [
      {
        "field": "username",
        "message": "\"username\" is required"
      }
    ]
  }
}
```

**When**: Missing required fields or invalid format

---

## Side Effects

- Updates `User.lastLogin` field to current timestamp
- Does not invalidate any existing tokens

---

## Notes

1. **Public Endpoint**: Does NOT require authentication
2. **Case-Insensitive Username**: Username is automatically lowercased before lookup
3. **Disabled Users**: Cannot log in even with correct credentials
4. **Token Types**: 
   - Access token: Short-lived (e.g., 1 hour), used for API requests
   - Refresh token: Long-lived (e.g., 7 days), used to get new access tokens
5. **Password**: Sent as plain text over HTTPS, verified against bcrypt hash

---

## Example Request

```bash
curl -X POST https://api.example.com/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "username": "admin",
    "password": "MySecurePassword123"
  }'
```

---

## Related Endpoints

- [POST /api/auth/register](./post-register.md) - Register new user
- [POST /api/auth/refresh](./post-refresh.md) - Refresh access token
- [POST /api/auth/logout](./post-logout.md) - Logout user
