# POST /api/auth/refresh

**Endpoint**: `POST /api/auth/refresh`
**Authentication**: ✅ Refresh token in body (no `Authorization` header required)
**Roles**: any role with a valid refresh token (including `teacher`)
**Source**: [auth.route.js](../../../../backend/src/routes/auth.route.js), [auth.controller.js](../../../../backend/src/controllers/auth.controller.js), [auth.service.js](../../../../backend/src/services/auth.service.js)
**Last verified**: 2026-05-22

---

## Description

Exchanges a valid refresh token for a new access token. **No Authorization header is required** — the whole point of refresh is to extend a session whose access token has expired. Identity is taken from the `sub` claim of the refresh token; the user is reloaded from MongoDB and a fresh access token is issued.

---

## Request

### Headers

```
Content-Type: application/json
```

### Body

```json
{
  "refreshToken": "string (required)"
}
```

Validated by `refreshSchema`:

| Field | Type | Rule |
|---|---|---|
| `refreshToken` | string | required |

---

## Response

### 200 OK

```json
{
  "data": {
    "accessToken": "eyJhbGciOiJIUzI1NiIs..."
  }
}
```

| Field | Type | Description |
|---|---|---|
| `data.accessToken` | string | A fresh JWT signed with `JWT_SECRET`, expiring per `JWT_EXPIRES_IN` (default `1h`). |

The refresh token is **not** rotated — keep using the same one until it expires (default `7d`, configurable via `JWT_REFRESH_EXPIRES_IN`).

---

## Error responses

### 400 — `VALIDATION_ERROR`

```json
{ "error": { "code": "VALIDATION_ERROR", "message": "\"refreshToken\" is required" } }
```

The body is missing `refreshToken`.

### 401 — `UNAUTHORIZED`

```json
{ "error": { "code": "UNAUTHORIZED", "message": "Invalid or expired refresh token" } }
```

Possible causes:
- `Missing refresh token` — body field empty.
- `Invalid or expired refresh token` — `jwt.verify(refreshToken, JWT_REFRESH_SECRET)` threw (bad signature, malformed token, expired).
- `Invalid user` — the user referenced by `refreshToken.sub` no longer exists or has `status === 'disabled'`.

---

## Behavior notes

1. **No access-token requirement.** Previously this route was gated by `authGuard, authorize(['admin','staff','viewer'])` — meaning a valid (non-expired) access token was required to refresh, and the `teacher` role was excluded. Both restrictions defeated the purpose. Fixed 2026-05-22: refresh now relies solely on the refresh token in the body.
2. **No refresh-token rotation.** The same refresh token is reusable for its full 7 days. If it is leaked, the holder can extend access for up to that long. To mitigate: disable the user account (blocks the next refresh + the next access-token check via `authGuard`), or rotate `JWT_REFRESH_SECRET` to invalidate every refresh token at once.
3. **Frontend integration.** [`frontend/src/api/client.ts`](../../../../frontend/src/api/client.ts) automatically calls `/auth/refresh` on any 401 (except for `/auth/refresh` itself) before bouncing the user to `/login`. A module-level mutex guarantees concurrent 401s share a single in-flight refresh call rather than each kicking one off.

---

## Token lifecycle (with the current frontend retry interceptor)

```
1. POST /api/auth/login                       → access (1 h) + refresh (7 d)
2. Frontend stores both in localStorage
3. Each request sends Bearer accessToken
4. After 1 h, accessToken expires
5. Next request → 401
6. Frontend silently:
   - POST /api/auth/refresh with { refreshToken }
   - On 200 → save new accessToken, retry the original request
   - On 401 → clear tokens, redirect to /login
7. User sees no interruption while the refresh token is still valid (7 d).
```

---

## Example

```bash
curl -X POST http://localhost:3000/api/auth/refresh \
  -H "Content-Type: application/json" \
  -d '{ "refreshToken": "'"$REFRESH_TOKEN"'" }'
```

---

## Related

- [POST /api/auth/login](./post-login.md) — issues both tokens
- [POST /api/auth/logout](./post-logout.md) — server-side no-op (clear client-side)
- [PATCH /api/auth/change-password](./patch-change-password.md) — does **not** rotate tokens
- [`docs/architecture/auth.md`](../../../architecture/auth.md) — full JWT + scoping picture
- [`docs/guides/environment-variables.md`](../../../guides/environment-variables.md) — `JWT_*` variable reference
