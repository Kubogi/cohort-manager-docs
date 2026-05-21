# Architecture Overview

This document is the highest-level system map. It tells you *where things live* and *how a request flows* without going deep into any one subsystem. For depth, see the sibling docs in this folder.

## 1. The big picture

```
                ┌─────────────────────────────────────────────────┐
                │                  Browser (SPA)                   │
                │  React 19 · TypeScript · Vite · react-admin 5    │
                │  ─ frontend/src/App.tsx                          │
                │  ─ dataProvider.tsx  ──► HTTP w/ Bearer token    │
                │  ─ authProvider.ts   ──► JWT in localStorage     │
                └────────────┬────────────────────────────────────┘
                             │ HTTPS / fetch                    
                             ▼                                  
                ┌─────────────────────────────────────────────────┐
                │           Nginx reverse proxy (prod)            │
                │   /        → static frontend (dist/)            │
                │   /api/*   → proxy_pass http://127.0.0.1:3000   │
                └────────────┬────────────────────────────────────┘
                             │                                  
                             ▼                                  
                ┌─────────────────────────────────────────────────┐
                │          Express API   (PM2-managed)            │
                │  backend/src/server.js → app.js → routes/       │
                │                                                 │
                │  helmet → cors → json → routes/index.js         │
                │    ├─ /api/auth/*           (public + protected)│
                │    ├─ authGuard → authorize(roles)              │
                │    └─ controller → service → model              │
                └────┬─────────────────────┬──────────────────────┘
                     │                     │                      
            primary  │                     │  secondary           
            Mongoose │                     │  Mongoose            
                     ▼                     ▼                      
        ┌──────────────────────┐  ┌─────────────────────────────┐ 
        │  MongoDB (primary)   │  │ MongoDB (secondary cluster) │ 
        │   MONGO_URI          │  │   MONGO_URI2                │ 
        │   13 collections     │  │   2 collections:            │ 
        │   sinh_vien, khoas,  │  │   ─ DonViLienKet (schools)  │ 
        │   daidois, diem*,    │  │   ─ CertificateLookup       │ 
        │   quyetdinhs, …      │  │   (read-mostly reference)   │ 
        └──────────────────────┘  └─────────────────────────────┘ 
                     ▲                                            
                     │ multer  ──► local FS                       
                     │                                            
                ┌─────────────────────────────────────────────────┐
                │   backend/uploads/                              │
                │   ├─ .tmp/             (multer staging)         │
                │   ├─ attachments/      (per-record files)       │
                │   └─ {deCuong,giaoTrinh,taiLieuThamKhao}/       │
                │                        (học liệu drives, 3 GB ea)│
                │                                                 │
                │   backend/forms/       (read-only xlsx templates│
                │   └─ Hệ {ĐH,CĐ}/       fed to /api/bieu-mau)    │
                └─────────────────────────────────────────────────┘
```

*\*The `diem` (grade) collection is **not** a separate collection: grades are an embedded array on each `SinhVien` document.*

## 2. Process model

In production, exactly one Node process serves the API (PM2-managed, restartable via `pm2 reload`). The frontend is a static bundle served directly by Nginx. There is no separate worker, no queue, no cron — all work is synchronous on the request path.

```
PM2 ─┬─ student-mgmt-api    (one fork instance, env from .env)
     │
Nginx (system service)
     │
mongod (system service, self-hosted) — primary cluster
mongod (remote, Atlas or self-hosted) — secondary cluster
```

See [`docs/guides/deployment.md`](../guides/deployment.md) for the full VPS playbook.

## 3. Request lifecycle (typical authenticated read)

A request from the SPA to fetch one page of students looks like this end-to-end:

```
1. User loads /coSoDuLieuSinhVien
2. react-admin asks dataProvider.getList('coSoDuLieuSinhVien', {pagination, sort, filter})
3. dataProvider.resourcePathMap maps 'coSoDuLieuSinhVien' → 'sinh-vien'
4. api/client.ts request() builds GET ${VITE_API_URL}/sinh-vien?page=1&limit=25
   - attaches Authorization: Bearer <accessToken from localStorage>
5. Nginx forwards to localhost:3000
6. Express:
   - app.js:        helmet → cors → json → urlencoded → morgan → routes/index.js
   - routes/index:  authGuard (verify JWT) → authorize(['admin','staff','viewer','teacher'])
   - sinhVien.route.js: GET '/' → sinhVien.controller.list
   - sinhVien.controller: validate query → sinhVien.service.list
   - sinhVien.service: build Mongoose query (applying unit-scope filters
                       from req.user.allowedUnits + teacherScope)
                     → SinhVien.find(...).populate(...)
   - returns { data: [...], meta: { total, page, limit } }
7. dataProvider normalizes _id → id and returns to react-admin
8. react-admin renders the table
```

If the JWT is invalid or missing, step 6 throws `HttpError(401, 'UNAUTHORIZED', ...)`, the error middleware emits `{ error: { code: 'UNAUTHORIZED', message: ... } }`, and the frontend's `api/client.ts` global 401 handler clears tokens and redirects to `/login`.

## 4. Tech stack at a glance

| Layer | Technology | Why it's there |
|---|---|---|
| Browser SPA | React 19 + TypeScript + Vite | Standard modern stack; Vite gives fast HMR. |
| Admin shell | react-admin 5.12 | Provides the resource/dataProvider/authProvider scaffolding; saves writing CRUD UIs from scratch. |
| Styling | Inline styles via design tokens (`src/styles/tokens/`) | No external CSS framework. See [`frontend/STYLING_GUIDE.md`](../../frontend/STYLING_GUIDE.md). |
| HTTP | `fetch` wrapped in `src/api/client.ts` | Bearer-token injection + global 401 logout + JSON/FormData handling. |
| API server | Node 22 + Express 5 | Minimal middleware: helmet, cors, json/urlencoded, morgan. |
| Validation | Joi 17 (`backend/src/validators/`) via `validate` middleware | Per-route schemas; strips unknown fields. |
| ORM | Mongoose 9 | Two `mongoose.connect`s — primary (default) and secondary (createConnection). |
| Auth | `jsonwebtoken` (HS256) + `bcryptjs` | Stateless JWT in localStorage; separate access (1 h) and refresh (7 d) tokens. See [`auth.md`](auth.md). |
| File upload | `multer` to `backend/uploads/` | Synchronous local-disk writes; size + extension + MIME validation in `backend/src/config/storage.js`. |
| Excel | **SheetJS (`xlsx`)** for imports; **ExcelJS** for exports | SheetJS handles all imports: student import (`sinhVien.service.js`), grade import (`diem.service.js`), survey upload (`khaoSatChatLuongProcess.service.js`). ExcelJS handles all exports: form templates (`bieuMau.service.js`), survey result download (`khaoSatChatLuongProcess.service.js`). See [`excel-pipeline.md`](excel-pipeline.md). |
| Logging | `morgan('dev')` to stdout | PM2 captures and rotates. No structured logging. |
| Backend tests | Jest + supertest + mongodb-memory-server | In-process Mongo; integration-test bias. |
| Frontend tests | Cypress (E2E) + Vitest + MSW (planned) | Only E2E login tests exist today; Vitest harness present but empty. |

## 5. Where data lives

| Concern | Where | Backed up how |
|---|---|---|
| Operational records (students, units, decisions, grades, health, staff, materials metadata) | **Primary MongoDB** (`MONGO_URI`) | `mongodump` per [`docs/guides/backups.md`](../guides/backups.md) |
| Partner schools + certificate lookup | **Secondary MongoDB** (`MONGO_URI2`) | Separate cluster; treated as mostly-read reference data |
| Uploaded files (attachments, học liệu) | `backend/uploads/` on the API host's filesystem | Filesystem snapshot / rsync (planned) |
| Excel templates (read-only) | `backend/forms/` (committed to git) | Git history |
| Auth tokens | Client-side `localStorage` only | N/A — stateless, re-issued on login |
| User passwords | `users.passwordHash` (bcrypt cost 10) in primary DB | Mongo backup |

## 6. Roles and access scoping at a glance

Four roles exist. See [`auth.md`](auth.md) for the full matrix.

| Role | Reads | Writes | Special |
|---|---|---|---|
| `admin` | Everything | Everything (incl. user management, link config) | Only role that can create users |
| `staff` | Records inside `allowedUnits` | Records inside `allowedUnits` | Most day-to-day work |
| `viewer` | Records inside `allowedUnits` | None | Read-only |
| `teacher` | Records inside `teacherScope` (`{khoa, daiDoi}` pairs) | Limited writes (grades, attendance) | Reduced menu (no Thông tin chung / Quản lý sinh viên admin) |

The JWT access token carries `role`, `allowedUnits`, and (for teachers) `teacherScope` so the frontend can gate UI without an extra API call.

## 7. What is *not* in this system

If you came expecting any of the following, you won't find them — and adding them is non-trivial:

- **No background workers** — every request handler runs to completion on the request thread. Excel exports of large grade books block the request.
- **No message queue.**
- **No cron jobs** managed inside the app. PM2 can schedule restarts, but no scheduled business logic exists.
- **No structured logging** (no Winston/Pino). `morgan('dev')` only.
- **No application-level caching** (no Redis). All reads hit Mongo.
- **No multi-tenant separation** — one Mongo cluster, one set of collections, all data co-mingled. Scoping is per-user via `allowedUnits`/`teacherScope`, not per-tenant.
- **No CI** at present (an empty `.github/workflows` would need to be added).

## 8. Where to go next

- **API reference** — [`docs/api/README.md`](../api/README.md)
- **Backend internals** — [`backend.md`](backend.md) (Express layer map, error envelope, validation, services)
- **Frontend internals** — [`frontend.md`](frontend.md) (react-admin shell, dataProvider, authProvider)
- **Database details** — [`database.md`](database.md) (two-connection setup, schemas, indexes, ERD)
- **Auth flow** — [`auth.md`](auth.md) (JWT issuance/refresh, role matrix, scope evaluation)
- **File storage** — [`file-storage.md`](file-storage.md)
- **Excel pipeline** — [`excel-pipeline.md`](excel-pipeline.md)
- **Local setup** — [`docs/guides/getting-started.md`](../guides/getting-started.md)
- **Production deployment** — [`docs/guides/deployment.md`](../guides/deployment.md)
