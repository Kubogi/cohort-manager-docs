# POST /api/auth/refresh

**Endpoint**: `POST /api/auth/refresh`
**Authentication**: ✅ Required (Bearer)

**Roles**: `admin`, `staff`, `viewer`

**Source**: [auth.route.js](../../../../backend/src/routes/auth.route.js), [auth.controller.js](../../../../backend/src/controllers/auth.controller.js), [auth.service.js](../../../../backend/src/services/auth.service.js)

**Last verified**: 2026-05-16

---

## Description

Exchanges a valid refresh token for a new access token. The route is mounted **behind `authGuard`**, so the caller must still send a *valid* (non-expired) access token in the `Authorization` header. This is unusual for a refresh endpoint — see [Behavior notes](#behavior-notes) below.

> **Important.** The previous version of this doc claimed that an *expired* access token could be sent in the `Authorization` header here. That is wrong — `authGuard` calls `jwt.verify` and throws `UNAUTHORIZED 401` on any expired token. If your access token has already expired, this endpoint cannot help you; the user must log in again. In the current shipped frontend (`src/api/client.ts`), 401 globally clears tokens and redirects to `/login`, so the refresh route is **not** part of the live flow today.

---

## Request

### Headers

```
Content-Type: application/json
Authorization: Bearer <valid access token>
```

The access token in the header is verified by `authGuard` (must be non-expired and the user must not be disabled). It is also used by `authorize(['admin', 'staff', 'viewer'])` to gate the route — note `teacher` is intentionally excluded.

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

The refresh token is **not** rotated — keep using the same one until it expires (default `7d`, configurable via `JWT_REFRESH_EXIRES_IN` — yes, that's the typo'd env-var name; see [`docs/guides/environment-variables.md`](../../../guides/environment-variables.md)).

---

## Error responses

### 400 — `VALIDATION_ERROR`

```json
{ "error": { "code": "VALIDATION_ERROR", "message": "\"refreshToken\" is required" } }
```

The body is missing `refreshToken`.

### 401 — `UNAUTHORIZED` (refresh token issue)

```json
{ "error": { "code": "UNAUTHORIZED", "message": "Missing refresh token" } }
```

Or any of:
- `Invalid refresh token` — `jwt.verify(refreshToken, JWT_REFRESH_SECRET)` threw (bad signature, malformed token, expired).
- `Invalid user` — the user referenced by `refreshToken.sub` no longer exists or has `status === 'disabled'`.

### 401 — `UNAUTHORIZED` (access token issue)

If the `Authorization` header is missing, expired, or signed with a different secret, `authGuard` rejects the request before `auth.service.refresh` runs. The error message will reflect the header problem (`Missing authorization token`, `Invalid or expired token`).

### 403 — `FORBIDDEN`

The user is a `teacher` (excluded from the route).

---

## Behavior notes

1. **Why authGuard is in front of refresh.** This deviates from the common "refresh endpoint is publicly callable with only the refresh token" pattern. The practical consequence is that you can't use this endpoint to *recover* an expired session — by the time your access token is expired, this route also rejects you. The intended threat model: refresh is callable only by an already-authenticated client extending its own lease, not by a holder of a leaked refresh token.
2. **No refresh-token rotation.** The same refresh token is reusable for its full 7 days. If it is leaked, anyone with both it *and* a fresh access token can extend access. To mitigate: change passwords on suspected compromise (does **not** automatically invalidate the refresh token — there's no token-blacklist), or rotate `JWT_REFRESH_SECRET` to invalidate every refresh token at once.
3. **Disabled users are blocked.** `User.status === 'disabled'` is checked before issuing the new access token; disabling a user is sufficient to lock them out at their next refresh (and immediately at their next access-token check via `authGuard`).
4. **Not currently used by the frontend.** `src/authProvider.ts` does not call `/auth/refresh`. The SPA logs out on any 401. Implementing sliding sessions on the client is the missing piece.

---

## Token lifecycle (with current frontend behavior)

```
1. POST /api/auth/login          → receive accessToken (1 h) + refreshToken (7 d)
2. Frontend stores both in localStorage
3. Bearer accessToken on every request
4. After 1 h, accessToken expires
5. Next request → 401 UNAUTHORIZED from authGuard
6. Frontend clears tokens + redirects to /login (refresh is NOT called)
7. User logs in again
```

When you wire up refresh on the client, step 6 becomes:

```
6. Frontend detects 401 + expired token
7. POST /api/auth/refresh with { refreshToken } and the now-expired accessToken
   ⚠ This currently fails because authGuard rejects the expired token.
   To make refresh actually usable, either:
   (a) remove authGuard + authorize from auth.route.js#refresh, OR
   (b) implement a pre-emptive refresh slightly before expiry, so the access
       token is still valid when refresh is called.
8. On success: store new accessToken, retry the original request.
9. On failure: fall back to redirect to /login.
```

Option (a) is the more conventional design; consider it before building a refresh-time-window scheduler.

---

## Example

```bash
curl -X POST http://localhost:3000/api/auth/refresh \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -d '{ "refreshToken": "'"$REFRESH_TOKEN"'" }'
```

---

## Related

- [POST /api/auth/login](./post-login.md) — issues both tokens
- [POST /api/auth/logout](./post-logout.md) — server-side no-op (clear client-side)
- [PATCH /api/auth/change-password](./patch-change-password.md) — does **not** rotate tokens
- [`docs/architecture/auth.md`](../../../architecture/auth.md) — full JWT + scoping picture
- [`docs/guides/environment-variables.md`](../../../guides/environment-variables.md) — `JWT_*` variable reference
