# Backend Architecture

The backend is a single Express 5 process talking to MongoDB via Mongoose. It serves a REST API mounted under `/api`, secured by JWT, and returns either `{ data, meta }` JSON (most reads/writes), `204 No Content` (most deletes), or a binary stream (downloads + Excel exports).

This document describes the layered structure, the request-handling chain, and the conventions a contributor must follow when adding a new route. For specific schemas see [`database.md`](database.md); for the auth flow see [`auth.md`](auth.md).

## 1. Process bootstrap

Two entry files:

```
backend/src/server.js          ── starts the HTTP listener after Mongo connects
   └─ connectPrimary()         ── from config/db.js (required)
   └─ app.listen(env.port)

backend/src/app.js             ── builds the Express app instance
   ├─ cors({origin:true, credentials:true})
   ├─ helmet()
   ├─ express.json()
   ├─ express.urlencoded({extended:true})
   ├─ morgan('dev')
   ├─ app.use('/api', routes)
   ├─ app.use(notFound)
   └─ app.use(errorHandler)
```

- **Mongo connection** opens *before* the listener binds (server.js:7). If it fails, the process exits non-zero — PM2 then restarts.
- **CORS** is wide-open (`origin: true`). Suitable for split frontend/backend deployments; lock down with `CORS_ORIGIN` in `.env` if you want stricter behavior (env var is read but currently only used as a default fallback).
- **No rate limiting** is configured. Helmet's defaults are accepted as-is.

## 2. Folder layout

```
backend/src/
├─ server.js                ── entrypoint (Mongo connect + listen)
├─ app.js                   ── Express app builder
├─ config/
│  ├─ env.js                ── reads ../.env at the project root
│  ├─ db.js                 ── connectPrimary() + connectSecondary()
│  ├─ storage.js            ── uploads paths, drive names, size limits, MIME allowlist
│  └─ bieuMau.config.js     ── Excel template (he, danhSach) → file map
├─ db/
│  └─ secondaryConnection.js  ── shared createConnection() for secondary cluster
├─ routes/
│  ├─ index.js              ── master router; mounts /auth then authGuard for the rest
│  └─ <domain>.route.js     ── per-domain routers (sinhVien, quyetDinh, …)
├─ middlewares/
│  ├─ auth.middleware.js    ── authGuard, authorize([roles])
│  ├─ validate.middleware.js  ── Joi schema runner
│  ├─ error.middleware.js   ── { error: { code, message } } envelope
│  └─ notFound.middleware.js  ── 404 fallthrough
├─ controllers/             ── thin request adapters; call into services
├─ services/                ── business logic; talk to models
├─ models/                  ── Mongoose schemas (15 of them)
├─ validators/              ── Joi schemas per domain
├─ utils/
│  ├─ auth.js               ── hash/verify password, sign/verify JWT
│  ├─ httpError.js          ── HttpError(status, code, message)
│  ├─ asyncHandler.js       ── async fn → middleware wrapper
│  └─ permissions.js        ── applyTeacherScope + allowedUnits scope builders
├─ tests/                   ── Jest integration tests (mongodb-memory-server)
└─ scripts/                 ── admin/seed scripts (npm-scriptable)
```

## 3. Route mounting and authorization

`backend/src/routes/index.js` is the single source of truth for which paths exist and which roles can hit them:

```js
router.use('/auth', authRouter);                         // public + protected sub-routes
router.get('/hello', publicHandler);                     // public smoke test
router.use(authGuard);                                   // ↓ everything below requires Bearer token
router.use('/sinh-vien',          authorize(['admin','staff','viewer','teacher']), studentsRouter);
router.use('/quyet-dinh',         authorize(['admin','staff','viewer','teacher']), decisionsRouter);
router.use('/ho-so-suc-khoe',     authorize(['admin','staff','viewer','teacher']), healthRouter);
router.use('/khoa-hoc',           authorize(['admin','staff','viewer','teacher']), khoaRouter);
router.use('/can-bo-quan-ly',     authorize(['admin','staff','viewer']),           staffRouter);
router.use('/don-vi',             authorize(['admin','staff','viewer','teacher']), unitsRouter);
router.use('/diem',               authorize(['admin','staff','viewer','teacher']), gradesRouter);
router.use('/users',                                                                  userRouter);   // admin-only inside route
router.use('/hoc-lieu',           authorize(['admin','staff','viewer','teacher']), hocLieuRouter);
router.use('/bieu-mau',           authorize(['admin','staff','viewer','teacher']), bieuMauRouter);
router.use('/settings',                                                                systemSettingRouter);
router.use('/khao-sat-chat-luong',                                                     khaoSatChatLuongRouter);
router.use('/truc-quan-su',       authorize(['admin','staff','viewer']),           trucQuanSuRouter);
router.use('/tra-cuu',            authorize(['admin','staff','viewer','teacher']), traCuuRouter);
router.use('/attachments',                                                             attachmentRouter);
```

Notes on this list:

- **17 mount points** total, including the `/hello` smoke route and `/auth` (which has both public and protected sub-routes).
- The middleware order matters: `authGuard` is applied via `router.use(...)` *after* `/auth` is mounted, so the `/auth/login` (public) is reachable; everything below the `router.use(authGuard)` line requires a valid token.
- `/can-bo-quan-ly` and `/truc-quan-su` do **not** include the `teacher` role.
- `/users`, `/settings`, `/khao-sat-chat-luong`, and `/attachments` are mounted **without** an outer `authorize()`. Their role gating lives **inside** the route file (`router.use(authorize([...]))` per sub-path). Don't assume from this file alone that they're public.

When adding a new domain, add an import + a `router.use(...)` line here; don't introduce a parallel mounting point.

## 4. The five layers per request

Each domain has 5 collaborating files under `backend/src/`:

```
routes/<domain>.route.js     │  HTTP verbs + paths, middleware order, validator wiring
controllers/<domain>.ctrl.js │  req → params/body, call service, format response
services/<domain>.svc.js     │  business rules, scope filtering, calls to models
models/<Domain>.js           │  Mongoose schema + indexes + lifecycle hooks
validators/<domain>.val.js   │  Joi schemas exported per route (create/update/list query)
```

**Layering rules** (enforce in review):

1. Routes contain no business logic — only `router.<verb>('/...', validate(schema), authorize([roles]), asyncHandler(controller.action))`.
2. Controllers don't touch Mongo — they extract `req.params`, `req.body`, `req.query`, `req.user`, hand off to a service, and shape the response into `{ data, meta }` (or call `res.status(204).end()`).
3. Services don't touch `req`/`res` — they take plain arguments + `actor` (the `req.user`) and return plain values or throw `HttpError`.
4. Models contain schema, indexes, and lightweight hooks (e.g. computing `tbMon` on `diem`). No service logic.
5. Validators export Joi schemas only — no logic, no DB.

## 5. Error envelope

Every error response goes through `errorHandler` (`middlewares/error.middleware.js`):

```json
{
  "error": {
    "code": "ERROR_CODE_IN_SCREAMING_SNAKE",
    "message": "Human-readable string"
  }
}
```

Throw with `HttpError(status, code, message)` from `utils/httpError.js`. Common codes used in the codebase:

| Code | Status | Where it comes from |
|---|---|---|
| `VALIDATION_ERROR` | 400 | `validate` middleware when a Joi schema fails |
| `INVALID_CREDENTIALS` | 401 | `auth.service.login` on bad username/password |
| `UNAUTHORIZED` | 401 | `authGuard` (missing/bad/expired token) |
| `FORBIDDEN` | 403 | `authorize` when role mismatches |
| `NOT_FOUND` | 404 | per-service when an entity can't be found |
| `USER_NOT_FOUND` | 404 | `auth.service.changePassword`, user routes |
| `USER_EXISTS` | 409 | `auth.service.register` on duplicate username |
| `DUPLICATE_*` | 409 | services that hit a unique-index conflict |
| `INVALID_FILE_TYPE` | 400 | `hoc-lieu` upload |
| `DRIVE_FULL` | 413 | `hoc-lieu` upload exceeding the 3 GB drive cap |
| `INTERNAL_ERROR` | 500 | error fallthrough |

`asyncHandler` (in `utils/`) wraps async handlers so thrown errors propagate to `errorHandler` without uncaught-promise warnings. Always wrap async controllers with it.

## 6. Validation

Joi runs *before* the controller via the `validate` middleware. The schema is per-route:

```js
// validators/sinhVien.validator.js
export const sinhVienCreateSchema = Joi.object({ maSV: Joi.string().required(), … });

// routes/sinhVien.route.js
router.post('/',
  authorize(['admin']),
  validate(sinhVienCreateSchema),
  asyncHandler(sinhVienController.create)
);
```

Behavior (from `validate.middleware.js`):
- `abortEarly: false` — all errors are collected, then concatenated with `'; '` into one message.
- `stripUnknown: true` — fields not in the schema are dropped from `req.body`.
- The validated, stripped body replaces `req.body` so downstream code can trust it.

Two routes intentionally **do not** validate body content:
- `POST /api/diem/import` — validator is `null`; the controller + service enforce `he`/`mon`/`khoa`/`donViLienKet` form fields and per-row contents directly.
- `POST /api/sinh-vien/import` and `POST /api/hoc-lieu/upload` — `multer` parses the multipart payload before Joi can run; per-field validation happens inside the controller/service.

Query parameters are mostly **not** validated by Joi. Each list endpoint reads `req.query` directly in its service. This is a known gap.

## 7. Response shape conventions

| Shape | Used for |
|---|---|
| `{ data: <object>, meta?: undefined }` | Single-record reads, creates, updates |
| `{ data: <array>, meta: { total, page, limit } }` | List endpoints (`GET /` collection routes) |
| HTTP 204 (no body) | Most delete endpoints |
| Binary stream | `GET /api/bieu-mau/export`, `GET /api/hoc-lieu/:id/download` |

The frontend dataProvider relies on this contract. If you add a new write endpoint, return `{ data: ... }` even on simple operations (e.g. `auth/change-password` returns `{ data: { message: '...' } }`).

## 8. Database connections

Two distinct Mongoose connections (see [`database.md`](database.md) for full detail):

- **Primary** — opened by `connectPrimary()` in `config/db.js` against `MONGO_URI`. Uses the default `mongoose.connect()` global. Models that use it: everything except the two below.
- **Secondary** — `db/secondaryConnection.js` calls `mongoose.createConnection(MONGO_URI2)`. Throws at import time if `MONGO_URI2` is missing. Models that use it: `DonViLienKet`, `CertificateLookup`.

If your service needs to join data from both connections, you cannot do it in Mongo — the connections live in different clusters. The convention is to fetch from one, then query the other with the resulting IDs in JS.

## 9. File uploads

`multer` is used in six places:

| Route | Limit | Destination | Allowed types |
|---|---|---|---|
| `POST /api/sinh-vien/import` | 20 MB | `uploads/.tmp/` then deleted | `.xls`, `.xlsx` |
| `POST /api/diem/import` | 20 MB | `uploads/.tmp/` then deleted | `.xls`, `.xlsx` |
| `POST /api/hoc-lieu/upload` | 100 MB | `uploads/<drive>/<folder-path>/` (persistent) | See `ALLOWED_EXTENSIONS` + `ALLOWED_MIMETYPES` in `config/storage.js` |
| `POST /api/attachments` | 100 MB (`MAX_FILE_SIZE`) | `uploads/attachments/<OwnerType>/<file>` (persistent) | per attachment policy |
| `POST /api/khao-sat-chat-luong/process` | 20 MB | `uploads/.tmp/` then deleted | `.xls`, `.xlsx` |
| `POST /api/can-bo-quan-ly/import` | 20 MB | `uploads/.tmp/` then deleted | `.xls`, `.xlsx` |

Disk-full and oversize errors should be checked in the controller (multer's own error events bubble to `errorHandler`). See [`file-storage.md`](file-storage.md).

## 10. Adding a new endpoint — checklist

When the task is "add `PATCH /api/foo/:id/bar`":

1. **Validator** — add `barUpdateSchema = Joi.object({...})` in `validators/foo.validator.js`.
2. **Service** — add `barUpdate({ id, payload, actor })` in `services/foo.service.js`. Apply `allowedUnits` / `teacherScope` filtering using helpers from `utils/permissions.js`. Throw `HttpError` for any non-success.
3. **Controller** — add `barUpdate = asyncHandler(async (req, res) => { const data = await service.barUpdate({...}); res.json({ data }); })`.
4. **Route** — in `routes/foo.route.js`: `router.patch('/:id/bar', authorize(['admin','staff']), validate(barUpdateSchema), barUpdate)`.
5. **Mount** — confirm `foo` is already mounted in `routes/index.js`; if not, add the import + `router.use(...)`.
6. **Test** — add an integration test in `backend/src/tests/foo.test.js` using `supertest` against the in-memory Mongo. Cover happy path + at least one failure mode.
7. **Docs** — add a new endpoint doc under `docs/api/endpoints/foo/patch-bar.md` following the template of `docs/api/endpoints/auth/post-login.md`. Add a row to `docs/api/README.md`.

## 11. Operational notes

- **Logs** — `morgan('dev')` to stdout. In production, PM2 captures stdout/stderr to `~/.pm2/logs/`. There is no JSON or structured logging — `grep` is the analysis tool.
- **Shutdown** — there is no graceful-shutdown handler. `SIGTERM` from PM2 kills the process; in-flight Mongo writes can theoretically be interrupted. Acceptable for current usage.
- **Migrations** — none in code. Schema changes are applied by editing `models/*.js` and writing one-off scripts under `backend/src/scripts/` (e.g. `fix-tbMon-precision.js`).
- **Secrets** — read from a single `.env` at the *project root* (one directory above `backend/`). The path is hard-coded in `config/env.js` as `path.resolve('../.env')`. PM2 must be started from the `backend/` directory for this to resolve correctly.

## 12. See also

- [`overview.md`](overview.md) — top-of-stack diagram and request flow
- [`auth.md`](auth.md) — JWT + role matrix + scope evaluation
- [`database.md`](database.md) — schemas, ERD, indexes, two-connection setup
- [`docs/api/README.md`](../api/README.md) — endpoint reference
- [`docs/guides/scripts.md`](../guides/scripts.md) — npm scripts and standalone scripts
