# GET /api/users/:id

Get a single user by ID.

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
    "_id": "60f7b3b3b3b3b3b3b3b3b3b3",
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
    },
    "status": "active",
    "teacherScope": [],
    "lastLogin": "2026-02-22T10:30:00.000Z",
    "createdAt": "2026-01-01T00:00:00.000Z",
    "updatedAt": "2026-02-22T10:30:00.000Z"
  }
}
```

### Error Responses

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
curl -X GET "http://localhost:5000/api/users/60f7b3b3b3b3b3b3b3b3b3b3" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

### JavaScript (fetch)
```javascript
const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
  headers: {
    'Authorization': `Bearer ${accessToken}`
  }
});
const { data } = await response.json();
```

### React Admin
```typescript
import { userAPI } from '@/api/client';

const user = await userAPI.getOne('60f7b3b3b3b3b3b3b3b3b3b3');
```

## Notes

- `passwordHash` is never included in the response
- Returns full user object including `allowedUnits` configuration
