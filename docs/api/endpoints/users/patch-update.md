# PATCH /api/users/:id

Update user role and/or password.

## Authorization

**Required Role:** Admin only

## URL Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `id` | string | Yes | MongoDB ObjectId of the user |

## Request Body

At least one field must be provided.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `role` | string | No | New role: `admin`, `staff`, `viewer`, or `teacher` |
| `password` | string | No | New password (min 6 characters). If empty string or omitted, password is not changed |
| `allowedUnits` | object | No | New unit permissions configuration (applies to `staff` role) |
| `teacherScope` | array | No | New **manual** teacher scope entries. Each entry: `{ khoa, daiDoi, allKhoa, allDaiDoi }`. Validated against actual Khoa/DaiDoi records before saving. |

> **`teacherScopeSynced` is not accepted here.** It is system-managed (rewritten by [`syncTeacherScope`](../../workflows/teacher-scope-sync.md) whenever the linked CBQL.phanCong[] or CBQL.taiKhoan changes). Joi `stripUnknown` silently drops it from the request body — sending it has no effect.

### Example Request Body

```json
{
  "role": "staff",
  "password": "newpassword123"
}
```

Or update only role:
```json
{
  "role": "admin"
}
```

Or update only password:
```json
{
  "password": "newsecurepass"
}
```

## Response

### Success (200 OK)

```json
{
  "data": {
    "_id": "60f7b3b3b3b3b3b3b3b3b3b3",
    "username": "staff01",
    "role": "staff",
    "allowedUnits": {
      "daiDoi": [],
      "khoa": [],
      "truong": [],
      "allowAll": {
        "daiDoi": false,
        "khoa": false,
        "truong": false
      }
    },
    "status": "active",
    "teacherScope": [],
    "lastLogin": "2026-02-22T10:30:00.000Z",
    "createdAt": "2026-01-01T00:00:00.000Z",
    "updatedAt": "2026-02-22T11:00:00.000Z"
  }
}
```

### Error Responses

#### 400 Bad Request - Validation Error
```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Mật khẩu phải có ít nhất 6 ký tự"
  }
}
```

#### 401 Unauthorized
No valid authentication token provided.

#### 403 Forbidden
User does not have admin role.

#### 404 Not Found
```json
{
  "error": {
    "code": "USER_NOT_FOUND",
    "message": "Không tìm thấy người dùng"
  }
}
```

## Examples

### cURL
```bash
curl -X PATCH "http://localhost:5000/api/users/60f7b3b3b3b3b3b3b3b3b3b3" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"role": "staff", "password": "newpass123"}'
```

### JavaScript (fetch)
```javascript
const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
  method: 'PATCH',
  headers: {
    'Authorization': `Bearer ${accessToken}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({ role: 'staff', password: 'newpass123' })
});
const { data } = await response.json();
```

### React Admin
```typescript
import { userAPI } from '@/api/client';

// Update role only
const updated = await userAPI.update(userId, { role: 'staff' });

// Update password only
const updated = await userAPI.update(userId, { password: 'newpass123' });

// Update both
const updated = await userAPI.update(userId, { role: 'admin', password: 'newpass123' });
```

## Security Considerations

- Admins can change any user's password without knowing the old password
- Password is optional - if omitted or empty string, password remains unchanged
- Password is hashed with bcrypt before storage
- Updated `passwordHash` is never returned in the response
- Username cannot be changed via this endpoint

## Notes

- At least one field must be provided in the request body
- Password field is optional in edit mode (empty = no change)
- `updatedAt` timestamp is automatically updated
