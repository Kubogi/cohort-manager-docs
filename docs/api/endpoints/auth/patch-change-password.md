# PATCH /api/auth/change-password

**Endpoint**: `PATCH /api/auth/change-password`  
**Authentication**: ✅ Required  

**Roles**: `admin`, `staff`, `viewer` — **`teacher` is excluded** (see Gotchas below)  

**Source**: [auth.route.js](../../../../backend/src/routes/auth.route.js), [auth.controller.js](../../../../backend/src/controllers/auth.controller.js)  

**Last Verified**: 2026-02-22

---

## Description

Allows an authenticated user to change their password by providing their current password and a new password. The user's identity is determined from the JWT token, not from the request body, ensuring users can only change their own passwords.

---

## Request

### Headers

```
Authorization: Bearer <accessToken>
Content-Type: application/json
```

### Body

```json
{
  "oldPassword": "string (required)",
  "newPassword": "string (required, min 6 chars)"
}
```

### Body Parameters

| Parameter | Type | Required | Constraints | Description |
|-----------|------|----------|-------------|-------------|
| oldPassword | string | Yes | - | Current password for verification |
| newPassword | string | Yes | min 6 characters | New password to set |

**Note**: Password confirmation is typically handled on the frontend. The backend only requires the new password once.

---

## Response

### Success Response (200 OK)

```json
{
  "data": {
    "message": "Password changed successfully"
  }
}
```

### Response Fields

| Field | Type | Description |
|-------|------|-------------|
| data.message | string | Success confirmation message |

---

## Error Responses

### 401 Unauthorized - Missing Token

```json
{
  "error": {
    "code": "UNAUTHORIZED",
    "message": "Missing authorization token"
  }
}
```

**Cause**: No `Authorization` header in the request.

---

### 401 Unauthorized - Invalid or Expired Token

```json
{
  "error": {
    "code": "UNAUTHORIZED",
    "message": "Invalid or expired token"
  }
}
```

**Cause**: The token is malformed, has expired, or was signed with a different secret.

---

### 401 Unauthorized - Invalid Old Password

```json
{
  "error": {
    "code": "INVALID_OLD_PASSWORD",
    "message": "Old password is incorrect"
  }
}
```

**Cause**: The oldPassword provided does not match the user's current password

---

### 400 Bad Request - Same Password

```json
{
  "error": {
    "code": "SAME_PASSWORD",
    "message": "New password must be different from old password"
  }
}
```

**Cause**: The newPassword is identical to the current password

---

### 400 Bad Request - Validation Error

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Mật khẩu mới phải có ít nhất 6 ký tự",
    "details": [
      {
        "field": "newPassword",
        "message": "Mật khẩu mới phải có ít nhất 6 ký tự"
      }
    ]
  }
}
```

**Cause**: Request body validation failed (missing fields, incorrect format)

**Common Validation Messages**:
- `"Mật khẩu cũ không được để trống"` - oldPassword is empty
- `"Mật khẩu cũ là bắt buộc"` - oldPassword is missing
- `"Mật khẩu mới phải có ít nhất 6 ký tự"` - newPassword is too short
- `"Mật khẩu mới không được để trống"` - newPassword is empty
- `"Mật khẩu mới là bắt buộc"` - newPassword is missing

---

### 404 Not Found - User Not Found

```json
{
  "error": {
    "code": "USER_NOT_FOUND",
    "message": "User not found or disabled"
  }
}
```

**Cause**: User account no longer exists or has been disabled

---

## Implementation Details

### Security Features

1. **User Identification**: User ID is extracted from the JWT token's `sub` claim, not from request body, preventing users from changing other users' passwords
2. **Password Verification**: Current password is verified using bcrypt before allowing change
3. **Password Hashing**: New password is hashed with bcrypt (10 rounds) before storage
4. **Same Password Check**: Prevents user from setting the same password again
5. **Roles**: `admin`, `staff`, and `viewer` can change their own password. The `teacher` role is **excluded** from this route's `authorize()` list — a teacher submitting this request receives `403 FORBIDDEN`. This is likely a backend bug.

### Database Impact

- Updates `passwordHash` field in the `users` collection
- No token invalidation - existing tokens remain valid after password change

### Backend Flow

1. Extract user ID from JWT token (`req.user.id` set by `authGuard` middleware)
2. Fetch user from database
3. Verify user exists and is active
4. Verify old password matches current hash
5. Verify new password is different from old
6. Hash new password with bcrypt
7. Save updated user document
8. Return success message

---

## Usage Examples

### cURL

```bash
curl -X PATCH http://localhost:3000/api/auth/change-password \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIs..." \
  -H "Content-Type: application/json" \
  -d '{
    "oldPassword": "oldSecurePass123",
    "newPassword": "newSecurePass456"
  }'
```

### JavaScript (fetch)

```javascript
const response = await fetch('http://localhost:3000/api/auth/change-password', {
  method: 'PATCH',
  headers: {
    'Authorization': `Bearer ${accessToken}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    oldPassword: 'oldSecurePass123',
    newPassword: 'newSecurePass456'
  })
});

const result = await response.json();
if (response.ok) {
  console.log(result.data.message); // "Password changed successfully"
}
```

### Frontend Integration (React Admin)

```typescript
import { request } from '@/api/client';

const changePassword = async (oldPassword: string, newPassword: string) => {
  return request<{ message: string }>('auth/change-password', {
    method: 'PATCH',
    body: { oldPassword, newPassword }
  });
};
```

---

## Testing Checklist

- [ ] Successfully change password with valid old password
- [ ] Reject change with incorrect old password (401)
- [ ] Reject change when new password equals old password (400)
- [ ] Reject password shorter than 6 characters (400)
- [ ] Reject request without authentication token (401)
- [ ] Reject request with expired token (401)
- [ ] Verify password is hashed in database (not plain text)
- [ ] Verify user can login with new password after change
- [ ] Verify existing tokens remain valid after password change

---

## Gotchas

- **`teacher` role cannot use this endpoint.** The route is gated `authorize(['admin', 'staff', 'viewer'])`. A teacher's request is rejected with `403 FORBIDDEN` before reaching the service. This is likely a bug in `auth.route.js`.

---

## Related Endpoints

- [POST /api/auth/login](./post-login.md) - Login with new password
- [POST /api/auth/refresh](./post-refresh.md) - Refresh access token
- [POST /api/auth/logout](./post-logout.md) - Logout

---

## Security Considerations

⚠️ **Token Persistence**: Existing access and refresh tokens remain valid after password change. For enhanced security, consider implementing token invalidation on password change.

⚠️ **Rate Limiting**: No rate limiting is currently implemented. Consider adding rate limiting to prevent brute force attacks on password changes.

💡 **Audit Trail**: Consider logging password change events for security auditing.

💡 **Email Notification**: Consider sending an email notification when password is changed.
