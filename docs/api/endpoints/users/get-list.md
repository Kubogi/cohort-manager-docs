# GET /api/users

List all users with optional filtering.

## Authorization

**Required Role:** Admin only

## Query Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `page` | number | No | Page number (default: 1) |
| `limit` | number | No | Results per page (default: 20) |
| `q` | string | No | Search by username (case-insensitive) |
| `role` | string | No | Filter by role: `admin`, `staff`, `viewer`, or `teacher` |

## Response

### Success (200 OK)

```json
{
  "data": [
    {
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
  ],
  "meta": {
    "total": 15,
    "page": 1,
    "limit": 20
  }
}
```

### Error Responses

#### 401 Unauthorized
No valid authentication token provided.

#### 403 Forbidden
User does not have admin role.

## Examples

### cURL
```bash
curl -X GET "http://localhost:5000/api/users?page=1&limit=10&role=admin" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

### JavaScript (fetch)
```javascript
const response = await fetch('http://localhost:5000/api/users?q=admin', {
  headers: {
    'Authorization': `Bearer ${accessToken}`
  }
});
const { data, meta } = await response.json();
```

### React Admin
```typescript
import { userAPI } from '@/api/client';

const { data, meta } = await userAPI.list({ page: 1, limit: 10, role: 'admin' });
```

## Notes

- `passwordHash` is never included in the response
- Results are sorted by creation date (newest first)
- Search is case-insensitive and matches partial usernames
