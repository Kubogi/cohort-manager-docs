# API Endpoint Reference

**Base URL**: `/api`
**Last updated**: 2026-05-16

This is the catalog of every REST endpoint the backend exposes. Per-endpoint files (request/response schemas, examples, error codes) live in [`endpoints/`](./endpoints/) — one Markdown file per HTTP verb + path.

For the cross-cutting picture (request lifecycle, error envelope, auth flow), see [`docs/architecture/backend.md`](../architecture/backend.md) and [`docs/architecture/auth.md`](../architecture/auth.md).

## Conventions

- **Auth.** All routes require an `Authorization: Bearer <accessToken>` header except `POST /api/auth/login` and `GET /api/hello`. See [`auth.md`](../architecture/auth.md).
- **Roles.** Four roles exist: `admin`, `staff`, `viewer`, `teacher`. Per-route gating is listed in the per-endpoint files.
- **Per-unit scoping.** `staff` reads are filtered by `User.allowedUnits`. `teacher` reads are filtered by `User.teacherScope`. `admin` and `viewer` are unscoped — viewer is read-only via route gating but sees every row their role can hit (`applyUnitScope` in `utils/permissions.js` no-ops for non-staff). See [`auth.md`](../architecture/auth.md#5-per-unit-scoping--allowedunits-staff).
- **Response shape.**
  - List endpoints return `{ data: Array, meta: { total, page, limit } }`.
  - Single-record endpoints return `{ data: Object }`.
  - Most deletes return `204 No Content` (empty body).
  - File-download and Excel-export endpoints return a **binary stream** with `Content-Disposition: attachment` — *not* the JSON envelope.
- **Error envelope.** `{ "error": { "code": "SCREAMING_SNAKE", "message": "Human-readable" } }`. The most common codes (`VALIDATION_ERROR`, `UNAUTHORIZED`, `FORBIDDEN`, `NOT_FOUND`, …) are listed in [`backend.md`](../architecture/backend.md#5-error-envelope).
- **Pagination.** `?page=1&limit=25` on list endpoints. Defaults vary per resource. `meta.total` is the unpaginated count.
- **Filtering.** List endpoints accept domain-specific query params. There is no generic `?filter=` parameter — only the named ones documented per endpoint.

## Authentication (`/api/auth`)

| Method | Path | Roles | Detail |
|---|---|---|---|
| POST | `/login` | public | [post-login.md](./endpoints/auth/post-login.md) |
| POST | `/register` | `admin` | [post-register.md](./endpoints/auth/post-register.md) |
| POST | `/refresh` | `admin`, `staff`, `viewer` | [post-refresh.md](./endpoints/auth/post-refresh.md) |
| POST | `/logout` | `admin`, `staff`, `viewer` | [post-logout.md](./endpoints/auth/post-logout.md) |
| PATCH | `/change-password` | `admin`, `staff`, `viewer` | [patch-change-password.md](./endpoints/auth/patch-change-password.md) |

## Users (`/api/users`)

Admin-only. Role gating lives inside the route file.

| Method | Path | Detail |
|---|---|---|
| GET | `/` | [get-list.md](./endpoints/users/get-list.md) |
| GET | `/:id` | [get-one.md](./endpoints/users/get-one.md) |
| PATCH | `/:id` | [patch-update.md](./endpoints/users/patch-update.md) |
| DELETE | `/:id` | [delete.md](./endpoints/users/delete.md) |

## Students (`/api/sinh-vien`)

`admin`/`staff`/`viewer`/`teacher` for reads; `admin` only for writes (except teachers can edit grades via `/api/diem`).

| Method | Path | Detail |
|---|---|---|
| GET | `/` | [get-list.md](./endpoints/sinh-vien/get-list.md) |
| GET | `/:id` | [get-one.md](./endpoints/sinh-vien/get-one.md) |
| POST | `/` | [post.md](./endpoints/sinh-vien/post.md) |
| PATCH | `/:id` | [patch.md](./endpoints/sinh-vien/patch.md) |
| DELETE | `/:id` | [delete.md](./endpoints/sinh-vien/delete.md) |
| POST | `/import` | [post-import.md](./endpoints/sinh-vien/post-import.md) — Excel import; multipart |
| GET | `/export/all` | [get-export.md](./endpoints/sinh-vien/get-export.md) — **stub** |

## Decisions (`/api/quyet-dinh`)

| Method | Path | Detail |
|---|---|---|
| GET | `/` | [get-list.md](./endpoints/quyet-dinh/get-list.md) |
| GET | `/:id` | [get-one.md](./endpoints/quyet-dinh/get-one.md) — admin/staff/viewer/teacher; teacher results scoped to `teacherScope` |
| POST | `/` | [post.md](./endpoints/quyet-dinh/post.md) — admin only |
| PATCH | `/:id` | [patch.md](./endpoints/quyet-dinh/patch.md) — admin only |
| DELETE | `/:id` | [delete.md](./endpoints/quyet-dinh/delete.md) — admin only |

Status workflow (`Chưa xử lý` ⇄ `Đã xử lý` with student-status side-effects) lives in [`workflows/decision-processing.md`](../workflows/decision-processing.md).

## Health records (`/api/ho-so-suc-khoe`)

| Method | Path | Detail |
|---|---|---|
| GET | `/` | [get-list.md](./endpoints/ho-so-suc-khoe/get-list.md) |
| GET | `/:id` | [get-one.md](./endpoints/ho-so-suc-khoe/get-one.md) |
| POST | `/` | [post.md](./endpoints/ho-so-suc-khoe/post.md) |
| PATCH | `/:id` | [patch.md](./endpoints/ho-so-suc-khoe/patch.md) |
| DELETE | `/:id` | [delete.md](./endpoints/ho-so-suc-khoe/delete.md) |
| GET | `/stats` | [get-stats.md](./endpoints/ho-so-suc-khoe/get-stats.md) |

## Grades (`/api/diem`)

Grades are stored as an embedded array on `SinhVien`; these endpoints are a *logical view*.

| Method | Path | Detail |
|---|---|---|
| GET | `/` | [get-list.md](./endpoints/diem/get-list.md) |
| GET | `/:id` | [get-one.md](./endpoints/diem/get-one.md) |
| POST | `/` | [post.md](./endpoints/diem/post.md) — admin/staff/teacher |
| PATCH | `/:id` | [patch.md](./endpoints/diem/patch.md) — admin/staff/teacher |
| DELETE | `/:id` | [delete.md](./endpoints/diem/delete.md) — admin/staff/teacher |
| GET | `/transcripts` | [get-transcripts.md](./endpoints/diem/get-transcripts.md) |
| GET | `/summary` | [get-summary.md](./endpoints/diem/get-summary.md) |
| GET | `/xep-loai-by-unit` | Rank-distribution roll-up per unit; query `?khoa=&daiDoi=&he=`. Returns `{ data: Array<{ unit, counts: { Giỏi, Khá, … } }> }`. |
| GET | `/export-so-tong` | Streams the bound "Sổ tổng" Excel binary for a given `(he, khoa, daiDoi)`. Uses `exceljs` against the `Sổ tổng.xlsx` template under `backend/forms/Hệ {ĐH,CĐ}/`. |
| POST | `/import` | [post-import.md](./endpoints/diem/post-import.md) — Excel grade-book import; multipart with body fields `he`, `mon`, `khoa`, `donViLienKet` |

## Courses (`/api/khoa-hoc`)

| Method | Path | Detail |
|---|---|---|
| GET | `/` | [get-list.md](./endpoints/khoa/get-list.md) |
| GET | `/:id` | [get-one.md](./endpoints/khoa/get-one.md) |
| POST | `/` | [post.md](./endpoints/khoa/post.md) |
| PATCH | `/:id` | [patch.md](./endpoints/khoa/patch.md) |
| DELETE | `/:id` | [delete.md](./endpoints/khoa/delete.md) |

## Management staff (`/api/can-bo-quan-ly`)

`teacher` role is **not** allowed on this resource.

| Method | Path | Detail |
|---|---|---|
| GET | `/` | [get-list.md](./endpoints/can-bo-quan-ly/get-list.md) |
| GET | `/:id` | [get-one.md](./endpoints/can-bo-quan-ly/get-one.md) |
| POST | `/` | [post.md](./endpoints/can-bo-quan-ly/post.md) |
| PATCH | `/:id` | [patch.md](./endpoints/can-bo-quan-ly/patch.md) |
| DELETE | `/:id` | [delete.md](./endpoints/can-bo-quan-ly/delete.md) |

## Units (`/api/don-vi`)

Three sub-resources for the organizational structure.

### Faculties — `/api/don-vi/khoa`

| Method | Path | Detail |
|---|---|---|
| GET | `/khoa` | [khoa-get-list.md](./endpoints/don-vi/khoa-get-list.md) |
| POST | `/khoa` | [khoa-post.md](./endpoints/don-vi/khoa-post.md) |
| PATCH | `/khoa/:id` | [khoa-patch.md](./endpoints/don-vi/khoa-patch.md) |
| DELETE | `/khoa/:id` | [khoa-delete.md](./endpoints/don-vi/khoa-delete.md) |

### Battalions — `/api/don-vi/dai-doi`

| Method | Path | Detail |
|---|---|---|
| GET | `/dai-doi` | [dai-doi-get-list.md](./endpoints/don-vi/dai-doi-get-list.md) |
| GET | `/dai-doi/:id` | [dai-doi-get-one.md](./endpoints/don-vi/dai-doi-get-one.md) |
| POST | `/dai-doi` | [dai-doi-post.md](./endpoints/don-vi/dai-doi-post.md) |
| PATCH | `/dai-doi/:id` | [dai-doi-patch.md](./endpoints/don-vi/dai-doi-patch.md) |
| DELETE | `/dai-doi/:id` | [dai-doi-delete.md](./endpoints/don-vi/dai-doi-delete.md) |

### Partner schools — `/api/don-vi/don-vi-lien-ket`

Read-only from this API. Lives in the secondary Mongo cluster.

| Method | Path | Detail |
|---|---|---|
| GET | `/don-vi-lien-ket` | [don-vi-lien-ket-get-list.md](./endpoints/don-vi/don-vi-lien-ket-get-list.md) |

## Learning materials (`/api/hoc-lieu`)

Folder/file tree across three drives (`deCuong`, `giaoTrinh`, `taiLieuThamKhao`). Quotas: 3 GB per drive, 100 MB per file. See [`docs/architecture/file-storage.md`](../architecture/file-storage.md).

| Method | Path | Detail |
|---|---|---|
| GET | `/` | List entries inside a drive/folder. Query: `drive`, optional `parent` |
| GET | `/usage` | Per-drive bytes used and percentage. Query: `drive` |
| POST | `/folder` | Create folder. Body: `{ name, drive, parent? }` |
| POST | `/upload` | Upload a file. multipart: `file` + `drive` + `parent?` |
| PATCH | `/:id/rename` | Rename a file or folder. Body: `{ name }` |
| DELETE | `/:id` | Delete (folder recursively) |
| GET | `/:id/download` | Stream the file binary |

## Excel form export (`/api/bieu-mau`)

| Method | Path | Detail |
|---|---|---|
| GET | `/export` | Pick a template by `(he, danhSach, mon, …)`, return populated `.xlsx`. See [`docs/architecture/excel-pipeline.md`](../architecture/excel-pipeline.md) for the template map |

Per-endpoint file under `endpoints/bieu-mau/` is planned.

## System settings (`/api/settings`)

Role gating is inside the route file (admin-only for writes; reads are broader). The currently shipped sub-routes:

| Method | Path | Detail |
|---|---|---|
| GET | `/training-schedule-link` | Read the configured "Lịch trình đào tạo" iframe URL |
| PATCH | `/training-schedule-link` | Admin updates the iframe URL |

Per-endpoint files under `endpoints/settings/` are planned.

## Quality survey (`/api/khao-sat-chat-luong`)

Annual survey configuration: per-year links + per-instructor feedback links. Role gating is inside the route file — reads are `admin`/`staff`/`viewer`/`teacher`; writes are `admin`-only; the `/process` upload is `admin`/`staff`.

| Method | Path | Purpose |
|---|---|---|
| GET | `/nam` | List configured years (`KhaoSatChatLuongNam`) |
| POST | `/nam` | Create a new survey year — admin |
| PATCH | `/nam/:id` | Update a year's GV-self-evaluation + center-feedback links — admin |
| GET | `/giang-vien?nam=…` | List instructors configured for a given year |
| POST | `/giang-vien` | Add an instructor (per-year) with a feedback URL — admin |
| PATCH | `/giang-vien/:id` | Update an instructor's name or link — admin |
| DELETE | `/giang-vien/:id` | Remove an instructor — admin |
| POST | `/process` | Multipart upload (`.xls`/`.xlsx`, 20 MB cap) of a Google-Forms survey export; backend parses it and writes per-instructor aggregate results. Admin/staff. See [`endpoints/khao-sat-chat-luong/README.md`](./endpoints/khao-sat-chat-luong/README.md). |

Per-endpoint files under `endpoints/khao-sat-chat-luong/` are summarized in the domain README.

## Military duty roster (`/api/truc-quan-su`)

`admin`/`staff`/`viewer` only — `teacher` is excluded. **Writes (POST/PATCH/DELETE) are admin/staff only**; viewers can only read.

| Method | Path | Roles | Purpose |
|---|---|---|---|
| GET | `/` | admin, staff, viewer | List roster rows (paginated by date) |
| GET | `/:id` | admin, staff, viewer | One row |
| POST | `/` | admin, staff | Create a row for a specific `ngay` (unique date) |
| PATCH | `/:id` | admin, staff | Edit a row |
| DELETE | `/:id` | admin, staff | Delete a row |

See [`endpoints/truc-quan-su/README.md`](./endpoints/truc-quan-su/README.md).

## Public lookup (`/api/tra-cuu`)

Reads from the secondary cluster's `CertificateLookup` collection. Used by the "Tra cứu" page to look up a student's grades or certificates by name + birthdate, or by maSV + sending school. **Both endpoints are `POST`** (body-based, not query-string) — see [`endpoints/tra-cuu/README.md`](./endpoints/tra-cuu/README.md) for the shared body shape.

| Method | Path | Purpose |
|---|---|---|
| POST | `/grades` | Look up grade rows |
| POST | `/certificates` | Look up certificate rows |

## Attachments (`/api/attachments`)

One attachment per `(ownerType, ownerId)` — see [`docs/architecture/database.md`](../architecture/database.md). Role gating is inside the route file.

| Method | Path | Roles | Purpose |
|---|---|---|---|
| GET | `/?ownerType=&ownerId=` | admin, staff, viewer, teacher | Look up the attachment metadata for a single owner record (used by the QĐ / Hồ sơ sức khỏe row components). |
| POST | `/` | admin, staff | Multipart upload (`file` field) — multer caps at `MAX_FILE_SIZE` (100 MB) and filters on `ALLOWED_MIMETYPES`. Creates one `Attachment` row keyed by `(ownerType, ownerId)`. |
| GET | `/:id/download` | admin, staff, viewer, teacher | Stream the file binary |
| DELETE | `/:id` | admin, staff | Delete (DB row + on-disk file) |

See [`endpoints/attachments/README.md`](./endpoints/attachments/README.md).

## Utility

| Method | Path | Purpose |
|---|---|---|
| GET | `/api/hello` | Public smoke-test endpoint. Returns the literal string `Ciallo ~ (∠・ω< )⌒★`. Useful for confirming the API is reachable through Nginx without needing a token. |
