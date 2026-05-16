# DELETE /api/users/:id

Delete a user (hard delete).

## Authorization

**Required Role:** Admin only

## URL Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `id` | string | Yes | MongoDB ObjectId of the user |

## Response

### Success (200 OK)

```json
{
  "data": {
    "message": "Xóa người dùng thành công"
  }
}
```

### Error Responses

#### 400 Bad Request - Cannot Delete Self
```json
{
  "error": {
    "code": "CANNOT_DELETE_SELF",
    "message": "Không thể xóa tài khoản của chính mình"
  }
}
```

#### 400 Bad Request - Cannot Delete Last Admin
```json
{
  "error": {
    "code": "CANNOT_DELETE_LAST_ADMIN",
    "message": "Không thể xóa quản trị viên cuối cùng"
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
curl -X DELETE "http://localhost:5000/api/users/60f7b3b3b3b3b3b3b3b3b3b3" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

### JavaScript (fetch)
```javascript
const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
  method: 'DELETE',
  headers: {
    'Authorization': `Bearer ${accessToken}`
  }
});
const { data } = await response.json();
```

### React Admin
```typescript
import { userAPI } from '@/api/client';

await userAPI.delete(userId);
```

## Security Considerations

⚠️ **Self-Deletion Prevention**: Users cannot delete their own account. The API checks if the user ID in the JWT token matches the ID being deleted.

⚠️ **Last Admin Protection**: The system must always have at least one admin user. If attempting to delete the last admin, the request will be rejected with a 400 error.

## Notes

- This is a **hard delete** — the user document is permanently removed from the database.
- **Issued JWT tokens remain valid until their natural expiry** (access tokens up to 1 h, refresh tokens up to 7 d). There is no token blacklist, so a deleted user's tokens can still authenticate requests within the expiry window. To block access immediately, disable the user first (`status: 'disabled'`) and delete later.
- Cannot be undone — consider using `status: 'disabled'` as a soft-delete for production use.
- The system prevents deleting the last admin account (400 CANNOT_DELETE_LAST_ADMIN).
