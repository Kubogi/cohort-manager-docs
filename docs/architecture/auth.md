# Authentication & Authorization

Auth in this app is stateless JWT, role-based, and additionally scoped per-user to specific organizational units. Most of what makes it correct lives in three places: the issuance pipeline in `auth.service.js`, the `authGuard` + `authorize` middlewares mounted by `routes/index.js`, and the scope helpers in `utils/permissions.js` that each service calls to filter queries.

This document covers the full picture: token issuance, the role matrix, the two scoping mechanisms (`allowedUnits` for **staff only**, `teacherScope` for teachers), and the failure modes you'll see when something is mis-configured. The viewer role currently has **no per-unit scoping** — see §5.

## 1. Token issuance

### Login flow

```
Client                              Backend
  │                                   │
  │  POST /api/auth/login             │
  │  { username, password }           │
  │ ────────────────────────────────► │  auth.controller.login
  │                                   │   ↓
  │                                   │  auth.service.login
  │                                   │   ├─ User.findOne({ username:lc })
  │                                   │   ├─ if disabled → 401 INVALID_CREDENTIALS
  │                                   │   ├─ bcrypt.compare(plain, hash)
  │                                   │   ├─ user.lastLogin = now
  │                                   │   ├─ signAccessToken(user) ─ 1 h
  │                                   │   └─ signRefreshToken(user) ─ 7 d
  │                                   │   ↓
  │  200 { data: {                    │
  │    accessToken,                   │
  │    refreshToken,                  │
  │    user: { id, username, role,    │
  │            allowedUnits,          │
  │            teacherScope? }        │  ← teacherScope only if role==='teacher'
  │  }}                               │
  │ ◄──────────────────────────────── │
```

### Access-token payload

Signed with `JWT_SECRET`, default expiry `1h` (overridable via `JWT_EXPIRES_IN`):

```json
{
  "sub": "<user._id as string>",
  "role": "admin | staff | viewer | teacher",
  "allowedUnits": {
    "daiDoi": ["<id>", "..."],
    "khoa": ["<id>", "..."],
    "truong": ["<id>", "..."],
    "allowAll": { "daiDoi": false, "khoa": false, "truong": false }
  },
  "teacherScope": [ /* only present for teachers */
    { "khoa": "<id>|null", "daiDoi": "<id>|null", "allKhoa": false, "allDaiDoi": false }
  ],
  "iat": …, "exp": …
}
```

The frontend decodes the payload client-side (atob + JSON.parse in `authProvider.ts:66`) to drive UI gating without an extra round-trip. This means a stolen token *carries* the user's scope — but it cannot grant access broader than what the user already had.

### Refresh-token payload

Signed with `JWT_REFRESH_SECRET`, default expiry `7d` (overridable via `JWT_REFRESH_EXIRES_IN` — **yes, that's the env var name, with a typo**; see [`docs/guides/environment-variables.md`](../guides/environment-variables.md)):

```json
{ "sub": "<user._id>", "iat": …, "exp": … }
```

The refresh token carries no role/scope; it is exchanged for a fresh access token via `POST /api/auth/refresh` (`{ refreshToken }` in the body). The current frontend does **not** call this endpoint — on a 401 the client clears tokens and redirects to login. Refresh is available for future use but not on the live login flow.

### Password storage

`auth.service.register` and `auth.service.changePassword` both go through `utils/auth.js#hashPassword` which is `bcrypt.hash(plain, 10)`. Cost 10 ≈ 100 ms on modern hardware — adequate for current login volume. There is no pepper, no per-user salt management — bcrypt's per-hash salt is sufficient.

## 2. The authGuard middleware

`middlewares/auth.middleware.js#authGuard`:

```js
const token = header.startsWith('Bearer ') ? header.slice(7) : null;
if (!token) throw new HttpError(401, 'UNAUTHORIZED', 'Missing authorization token');
const decoded = jwt.verify(token, env.jwtSecret);     // throws on bad/expired
const user = await User.findById(decoded.sub).lean(); // reaches Mongo on every request
if (!user || user.status === 'disabled') throw new HttpError(401, 'UNAUTHORIZED', 'Invalid user');
req.user = { ...user, id: user._id.toString() };
```

Two important consequences:

1. **Expired tokens fail closed.** `jwt.verify` throws on `exp` in the past. The handler catches it and emits `UNAUTHORIZED 401`. `authGuard` is mounted on the `/auth/refresh` route — it verifies the access token before the service runs. If the access token is expired, the request is rejected at the middleware layer and never reaches `auth.service.refresh`. The service reads only the body's `refreshToken`, but the route still requires a valid access token to get there.
2. **Every protected request touches Mongo.** `User.findById` runs on every call. This bounds API throughput at the User collection's read latency. If the bottleneck matters, cache the User doc in memory keyed by `_id` with a short TTL.

## 3. The authorize middleware

```js
export const authorize = (roles = []) => (req, _res, next) => {
  if (!req.user) return next(new HttpError(401, 'UNAUTHORIZED', 'Not authenticated'));
  if (roles.length && !roles.includes(req.user.role)) {
    return next(new HttpError(403, 'FORBIDDEN', 'Insufficient permissions'));
  }
  next();
};
```

`authorize` is a *role gate*. It cannot enforce per-record permissions — those live in the service layer using `applyUnitScope` and `applyTeacherScope`. The pattern is always:

```js
router.use(authGuard);                      // who you are
router.use(authorize(['admin','staff']));   // what role can hit this route
// inside the service:
const filter = applyTeacherScope(
  applyUnitScope({ /* user filter */ }, req.user, { khoa: 'khoa', daiDoi: 'daiDoi' }),
  req.user,
  { khoa: 'khoa', daiDoi: 'daiDoi' }
);
const rows = await SinhVien.find(filter)…
```

`authorize` alone is **not** enough. A staff user with `allowedUnits.khoa = ['K1']` passing `authorize(['staff'])` could otherwise read every student in every faculty.

## 4. Role matrix

| Role | What it can read | What it can write | Notable gates |
|---|---|---|---|
| `admin` | All collections, all records | All collections, all records | Only role that can `POST /api/auth/register`, `PATCH /api/auth/users/:id`, manage `/api/settings` links, `quanLyLink` page in UI |
| `staff` | Records inside `allowedUnits` for `khoa`/`daiDoi`/`truong` axes | Same set | Can edit grades + decisions + health records. Cannot manage users. |
| `viewer` | **All records their route gate allows** — `applyUnitScope` no-ops for non-staff, so `viewer` effectively sees what an unscoped `staff` user with `allowAll` would. | Nothing | Read-only by route gating. Verify this is intentional before deploying widely; see §5. |
| `teacher` | Records inside `teacherScope` `(khoa, daiDoi)` pairs | Grades (`/api/diem` POST/PATCH/DELETE on rows inside scope) | Reduced menu; sees `lichHuanLuyen`, `soTayGiangVien`, `khoHocLieuSo`, `nhapDiem`, `traCuu`, `thongKeBaoCao`, the survey pages, and `caiDatTaiKhoan`. |

### Which routes allow which roles (effective, after `routes/index.js`)

| Route prefix | `admin` | `staff` | `viewer` | `teacher` |
|---|:-:|:-:|:-:|:-:|
| `POST /api/auth/login` | public | public | public | public |
| `POST /api/auth/register` | ✓ | — | — | — |
| `POST /api/auth/refresh` | ✓ | ✓ | ✓ | — ⚠ |
| `POST /api/auth/logout` | ✓ | ✓ | ✓ | — ⚠ |
| `PATCH /api/auth/change-password` | ✓ | ✓ | ✓ | — ⚠ |
| `/api/sinh-vien/*` | ✓ | ✓ | ✓ | ✓ |
| `/api/quyet-dinh/*` | ✓ | ✓ | ✓ | ✓ |
| `/api/ho-so-suc-khoe/*` | ✓ | ✓ | ✓ | ✓ |
| `/api/khoa-hoc/*` | ✓ | ✓ | ✓ | ✓ |
| `/api/can-bo-quan-ly/*` | ✓ | ✓ | ✓ | — |
| `/api/don-vi/*` | ✓ | ✓ | ✓ | ✓ |
| `/api/diem/*` | ✓ | ✓ | ✓ | ✓ |
| `/api/users/*` | ✓ | — | — | — |
| `/api/hoc-lieu/*` | ✓ | ✓ | ✓ | ✓ |
| `/api/bieu-mau/*` | ✓ | ✓ | ✓ | ✓ |
| `/api/settings/*` | depends on sub-route | depends | depends | depends |
| `/api/khao-sat-chat-luong/*` | depends | depends | depends | depends |
| `/api/truc-quan-su/*` | ✓ | ✓ | ✓ | — |
| `/api/tra-cuu/*` | ✓ | ✓ | ✓ | ✓ |
| `/api/attachments/*` | depends | depends | depends | depends |
| `GET /api/hello` | public | public | public | public |

> Cells marked "depends" are mounted in `routes/index.js` *without* an outer `authorize()` — they apply role gating inside the route file (`router.use(authorize(...))` per sub-path). Don't assume an unprotected mount means the route is open; read the route file.
>
> ⚠ **`teacher` excluded from `/auth/refresh`, `/auth/logout`, `/auth/change-password`.** These three routes are gated `['admin', 'staff', 'viewer']` in [`auth.route.js`](../../backend/src/routes/auth.route.js). A teacher account therefore cannot refresh its access token (so it is forced back to the login page after the 1 h access-token expiry), cannot call the logout endpoint, and cannot change its own password. This is almost certainly a bug — add `'teacher'` to those three authorize lists when fixing.

## 5. Per-unit scoping — `allowedUnits` (staff)

`User.allowedUnits` is the scope for `staff` users only (not `viewer`, not `teacher`, never `admin`). The schema:

```ts
allowedUnits: {
  daiDoi: ['<id>', ...],      // battalions this user can see
  khoa: ['<id>', ...],        // faculties
  truong: ['<id>', ...],      // partner schools
  allowAll: {
    daiDoi: boolean,          // if true: ignore daiDoi list, allow all
    khoa: boolean,
    truong: boolean
  }
}
```

`applyUnitScope(filter, user, mapping)` from `utils/permissions.js` rewrites a Mongoose filter to intersect the user's request with the user's allowed set:

```js
// Example: SinhVien list, staff with allowedUnits.khoa = ['K1','K2'], allowAll.daiDoi = true
const filter = applyUnitScope({}, req.user, { khoa: 'khoa', daiDoi: 'daiDoi' });
// → { khoa: { $in: ['K1','K2'] } }
// (daiDoi axis is skipped because allowAll.daiDoi = true)

// Same user, asks for ?khoa=K3
const filter = applyUnitScope({ khoa: 'K3' }, req.user, { khoa: 'khoa', daiDoi: 'daiDoi' });
// K3 is not in their allowed set → returns { khoa: 'K3', _id: { $in: [] } } (zero rows)
```

> **Important:** only pass axes in the mapping that the user has been granted access to. If you include an axis (e.g. `daiDoi`) that the user has no `allowedUnits.daiDoi` list AND `allowAll.daiDoi = false`, the helper treats it as a disjoint constraint and returns `{ _id: { $in: [] } }` — zero rows. Either set `allowAll.<axis> = true` for axes you want to permit freely, or omit them from the mapping entirely.

The "disjoint sentinel" `_id: { $in: [] }` is how the helper expresses "no rows for you" without throwing. Services use it directly in queries; an empty list is returned to the client, no 403. (A 403 would leak the existence of records the user can't see.)

For roles other than `staff`, `applyUnitScope` returns the input filter unchanged. **Admin is global; viewer and teacher use other mechanisms.** That said, the viewer role currently has no scope mechanism — viewers see whatever a staff user with `allowAll: true` would see. Verify this matches policy before deploying widely.

## 6. Per-unit scoping — `teacherScope` (teacher)

`User.teacherScope` is the scope for `teacher` users only. It is an *array of access entries*, each entry being a `(khoa, daiDoi)` pair, possibly with wildcards:

```ts
teacherScope: Array<{
  khoa: ObjectId | null,
  daiDoi: ObjectId | null,
  allKhoa: boolean,        // wildcard on khoa axis
  allDaiDoi: boolean       // wildcard on daiDoi axis
}>
```

`applyTeacherScope(filter, user, mapping)` ORs the entries together and ANDs the result into the existing filter:

```js
// teacherScope:
//   { khoa: 'K1', daiDoi: 'D2' },                   // can see K1 ∩ D2
//   { khoa: 'K1', allDaiDoi: true }                 // and all of K1
//
// Query: SinhVien.find(applyTeacherScope({}, user, {khoa:'khoa', daiDoi:'daiDoi'}))
// → { $and: [ { $or: [
//     { khoa: 'K1', daiDoi: 'D2' },
//     { khoa: 'K1' }
//   ] } ] }
```

Rules:
- **Non-teacher users:** filter returned unchanged.
- **Empty teacherScope array:** sentinel `{ _id: { $in: [] } }` — zero rows.
- **Entry with `allKhoa && allDaiDoi`:** that entry contributes no constraint (it's a wildcard) and is silently dropped from `orBranches`. If ALL entries in `teacherScope` are all-wildcard, `orBranches` is empty and the filter is returned unchanged — the teacher effectively has full read access. If specific (non-wildcard) entries are also present, only those specific entries constrain the query; the all-wildcard entry is dropped without affecting the others.
- **User-supplied filter intersects with the scope** (`$and`) — the teacher cannot widen by passing query params.

For *write* operations on a specific entity, services call `isPairInTeacherScope(khoa, daiDoi, user)` from the same file. It returns `true` for non-teachers (write authorization happens through `authorize(...)` for them). For teachers, it returns `false` if their scope doesn't include the (khoa, daiDoi) pair. Use this whenever a teacher's write operation targets a specific entity (e.g. editing a grade row).

> **Cross-DB validation:** `services/user.service.js#validateTeacherScopeAgainstDB` checks every `(khoa, daiDoi)` in a new teacher's scope against the actual Khoa and DaiDoi collections, so misconfiguration is caught at the time of user creation, not at first request.

## 7. Token lifetime and the missing refresh flow

The access token lifetime is **1 hour** by default; the refresh token is **7 days**. The frontend currently *does not* call `POST /api/auth/refresh` — on a 401 (which happens 1 hour after login, or sooner if the user's account is disabled), `api/client.ts` clears localStorage and redirects to `/login`.

This means users are silently logged out every hour and have to re-enter credentials. If you want sliding sessions:

1. Add an interceptor in `api/client.ts` that detects 401 with an expired access token (parse the `exp` claim) and calls `/api/auth/refresh` with the stored refresh token.
2. On success, store the new access token and retry the original request.
3. On failure, fall through to the current "redirect to login" path.

The backend is ready; only the client is missing the wiring.

## 8. Failure modes (what error envelopes you'll see)

| HTTP | code | Meaning | Likely cause |
|---|---|---|---|
| 400 | `VALIDATION_ERROR` | Joi rejected the request body | Missing username, password too short, malformed `teacherScope` IDs |
| 401 | `INVALID_CREDENTIALS` | Wrong username/password on `/auth/login` | — |
| 401 | `INVALID_OLD_PASSWORD` | Wrong `oldPassword` on `/auth/change-password` | — |
| 400 | `SAME_PASSWORD` | New password identical to current on `/auth/change-password` | — |
| 401 | `UNAUTHORIZED` | Missing/expired/bad token, or `user.status === 'disabled'` | Token expired (1 h default), user disabled, secret rotated |
| 403 | `FORBIDDEN` | Token valid, but role not in `authorize(...)` list | Trying to hit an admin route as staff |
| 404 | `USER_NOT_FOUND` | `change-password` on a deleted/disabled user | Account state changed mid-session |
| 409 | `USER_EXISTS` | `register` with an already-taken username | — |

If the client receives 401 from any non-`/auth/*` route, it MUST clear tokens. The frontend does this globally; ad-hoc fetch calls outside `api/client.ts` should follow the same pattern.

## 9. Operational checks

- **JWT secrets must be stable across PM2 restarts.** If `JWT_SECRET` rotates, every existing token instantly becomes invalid (clients see 401 and have to re-login). This is acceptable as a forced-logout mechanism; just be intentional about it.
- **Disable accounts via `User.status = 'disabled'`** rather than deleting. `authGuard` rejects disabled users at the next request without needing to invalidate tokens.
- **Account creation is admin-only.** Use `npm run create:admin` for the first admin; thereafter use the user-management UI.
- **Auditing.** There is no audit log. `User.lastLogin` is the only access trace. If you need richer audit data, add it to the service layer.

## 10. See also

- [`backend.md`](backend.md) — where auth slots into the request lifecycle
- [`docs/api/endpoints/auth/`](../api/endpoints/auth/) — per-endpoint contracts for `/api/auth/*`
- [`docs/workflows/login-and-session.md`](../workflows/login-and-session.md) — end-to-end login walkthrough
- [`docs/workflows/user-management.md`](../workflows/user-management.md) — admin creating a teacher and configuring `teacherScope`
- `backend/src/utils/permissions.js` — the source of truth for `applyUnitScope`, `applyTeacherScope`, `isPairInTeacherScope`
- `backend/src/middlewares/auth.middleware.js` — `authGuard` and `authorize`
