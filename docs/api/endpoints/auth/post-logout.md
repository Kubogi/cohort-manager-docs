# POST /api/auth/logout

**Endpoint**: `POST /api/auth/logout`  
**Authentication**: ✅ Required  

**Roles**: admin, staff, viewer  

**Source**: [auth.route.js](../../../../backend/src/routes/auth.route.js), [auth.controller.js](../../../../backend/src/controllers/auth.controller.js)  

**Last Verified**: 2026-05-16

---

## Description

Logs out the current user. 

**Note**: Currently a no-op endpoint - tokens are NOT invalidated server-side. Client should discard tokens locally.

---

## Request

### Headers

```
Authorization: Bearer <access_token>
```

### Body

```json
{}
```

**Note**: Request body should be empty object or omitted.

---

## Response

### Success Response (204 No Content)

**Empty response body**

**Status**: 204 No Content

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

**When**: No Authorization header or invalid/expired token

---

## Implementation Details

**Current Implementation**: The logout endpoint currently does NOT:
- Invalidate access tokens
- Invalidate refresh tokens
- Add tokens to a blacklist
- Store logout state on server

The endpoint simply returns 204 No Content.

**Client Responsibility**: The client MUST:
1. Discard access token from memory/storage
2. Discard refresh token from memory/storage
3. Clear any cached user data
4. Redirect to login page

---

## Token Handling

Since tokens are NOT invalidated server-side:

**Valid Until Expiration**:
- Access tokens remain valid until their `exp` claim
- Refresh tokens remain valid until their `exp` claim
- Tokens CAN still be used for API requests after logout

**Security Implications**:
- If attacker has token, they can use it until expiration
- Users should be informed that "logout" doesn't invalidate session server-side
- For high-security scenarios, implement token blacklisting

---

## Client-Side Logout Flow

```javascript
// 1. Call logout endpoint
const response = await fetch('/api/auth/logout', {
  method: 'POST',
  headers: {
    'Authorization': `Bearer ${accessToken}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({})
});

// 2. Clear tokens locally
localStorage.removeItem('accessToken');
localStorage.removeItem('refreshToken');
// or for cookies:
document.cookie = 'accessToken=; expires=Thu, 01 Jan 1970 00:00:00 UTC; path=/;';
document.cookie = 'refreshToken=; expires=Thu, 01 Jan 1970 00:00:00 UTC; path=/;';

// 3. Clear user data
localStorage.removeItem('user');

// 4. Redirect to login
window.location.href = '/login';
```

---

## Notes

1. **No-Op Endpoint**: Server does not invalidate tokens
2. **Client Responsibility**: Client must discard tokens
3. **Tokens Remain Valid**: Until natural expiration
4. **Status Code**: Returns 204 (No Content), not 200
5. **Empty Body**: No response body returned

---

## Future Enhancement

For enhanced security, consider implementing:

```javascript
// Token blacklist in Redis
export const logout = async ({ accessToken, refreshToken }) => {
  const decoded = verifyAccessToken(accessToken);
  const ttl = decoded.exp - Math.floor(Date.now() / 1000);
  
  // Add to blacklist with TTL matching token expiration
  await redis.setex(`blacklist:${accessToken}`, ttl, '1');
  await redis.setex(`blacklist:${refreshToken}`, ttl, '1');
  
  return { message: 'Logged out' };
};

// Then check blacklist in authGuard middleware
```

---

## Example Request

```bash
curl -X POST https://api.example.com/api/auth/logout \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIs..." \
  -H "Content-Type: application/json" \
  -d '{}'
```

**Response**: 204 No Content (empty body)

---

## Related Endpoints

- [POST /api/auth/login](./post-login.md) - Login user
- [POST /api/auth/refresh](./post-refresh.md) - Refresh access token
- [POST /api/auth/register](./post-register.md) - Register new user
