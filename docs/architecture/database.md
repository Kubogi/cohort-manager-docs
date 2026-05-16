# Database Architecture

The backend talks to **two distinct MongoDB clusters** through Mongoose. The primary cluster owns the operational data the app reads and writes daily. The secondary cluster is treated as a mostly-read reference store for partner schools and historical certificate lookups.

This document covers: the two-connection setup, the 15 models and their relationships, indexes worth knowing, embedded-vs-referenced choices, and the gotchas that have bitten contributors before.

> **Per-collection schema docs.** `docs/backend/schemas/` contains one file per model with field-by-field definitions. This document is the cross-cutting view. When you need to know what a single field does, go to the schema doc; when you need to know how a query fans across collections, stay here.

## 1. Two connections, two roles

```
                  ┌──────────────────────────────────────────────┐
                  │  primary Mongo cluster ── MONGO_URI          │
                  │  ─ Opened via connectPrimary() at boot       │
                  │  ─ Uses mongoose's default global connection │
                  │  ─ 13 collections                            │
                  └────────────────────────┬─────────────────────┘
                                           │
       mongoose.model('Name', schema)      │
       ─────────────────────────────────►  │  ◄── 13 models bind to default
                                           │
                  ┌────────────────────────┴─────────────────────┐
                  │  secondary Mongo cluster ── MONGO_URI2       │
                  │  ─ Opened lazily by db/secondaryConnection.js│
                  │  ─ Uses mongoose.createConnection()           │
                  │  ─ 2 collections (schools, students)         │
                  └──────────────────────────────────────────────┘

       secondaryConnection.model('Name', schema)
       ─────────────────────────────────►       ◄── 2 models bind here
```

**Bootstrap behavior:**

- `connectPrimary()` in `backend/src/config/db.js` is awaited in `server.js` before the HTTP listener binds. If it fails, the process exits and PM2 restarts.
- `secondaryConnection.js` is **eagerly imported** the moment any model that needs it loads. It **throws synchronously** if `MONGO_URI2` is missing, which propagates as a startup crash. There is no graceful skip — `MONGO_URI2` is effectively required.

**Why two connections?** The secondary database is a separate project (the [Certificate Management](https://github.com/Kubogi/certificate-management-docs) system) that owns the catalog of partner schools and the historical certificate index. Keeping it in its own cluster lets the lookup service iterate independently and lets multiple consumers (this app + others) share that data without coupling deploys.

You **cannot** join across the two clusters in MongoDB. Code that needs data from both fetches from one, then queries the other in JavaScript with the resulting IDs.

## 2. Collections at a glance

### Primary cluster

| Mongoose model | Collection | Owner role | Why it exists |
|---|---|---|---|
| `User` | `users` | Auth | Application accounts; bcrypt hashes, role, per-unit scope |
| `SinhVien` | `sinh_vien` | Core | Student records; **embeds grades** in `diem[]` |
| `Khoa` | `khoa` | Org | Faculties / departments |
| `DaiDoi` | `dai_doi` | Org | Battalions; aligned arrays for staff assignments per period |
| `QuyetDinh` | `quyet_dinh` | Workflow | Administrative decisions affecting a student |
| `HoSoSucKhoe` | `ho_so_suc_khoe` | Workflow | Health/hospital records per student |
| `CanBoQuanLy` | `can_bo_quan_ly` | Org | Management staff / instructors |
| `HocLieu` | `hoc_lieu` | Files | Learning-materials repository entries (file + folder nodes) |
| `Attachment` | `attachments` | Files | Polymorphic file refs for QuyetDinh & HoSoSucKhoe |
| `SystemSetting` | `systemsettings` *(default collection name)* | Admin | Key–value config (training-schedule link, etc.) |
| `KhaoSatChatLuongNam` | `khao_sat_chat_luong_nam` | Surveys | Per-year survey link configuration |
| `GiangVienKhaoSat` | `giangVien` *(yes, camelCase)* | Surveys | Per-year list of instructors with their per-instructor feedback link |
| `TrucQuanSu` | `truc_quan_su` | Org | Daily military-duty roster (one row per date) |

### Secondary cluster

| Mongoose model | Collection | Mode | Notes |
|---|---|---|---|
| `DonViLienKet` | `schools` | Read-mostly | Partner schools. Fields aliased: `ten` ↔ `ten_truong`, `heDaoTao` ↔ `he_dao_tao` |
| `CertificateLookup` | `students` | Read-only | Historical certificate index. Snake-case field names (`ho_va_ten`, `ngay_sinh`, …) — populated by the Certificate Management project. |

> **Naming gotcha.** Two of the secondary models target collection names that look generic (`schools`, `students`). Don't confuse `students` (secondary, snake-case, read-only) with `sinh_vien` (primary, camelCase, mutable).

## 3. ERD (primary cluster only)

```
                ┌────────────────┐
                │ DonViLienKet   │  (lives in secondary cluster)
                │   schools      │
                └───────┬────────┘
                        │ truong (ref by id only — no FK in Mongo)
                        ▼
   ┌──────────┐   khoa  ┌──────────────┐   khoa   ┌────────┐
   │ Khoa     │◄────────│  SinhVien    │─────────►│ DaiDoi │◄───┐
   │          │         │              │          │        │    │ daiDoi
   │          ├────────►│  diem[] ◄── embedded     │        │    │
   │ daiDoi[] │ daiDoi  │              │          │ canBo[]│    │
   └──────────┘         │              │          └────────┘    │
                        │              │                        │
                        │              │  ┌──────────────┐      │
                        ├──────────────┼─►│  QuyetDinh   │      │
                        │              │  │  khoa,daiDoi ├──────┘
                        │              │  └──────────────┘
                        │              │
                        │              │  ┌────────────────┐
                        ├──────────────┼─►│  HoSoSucKhoe   │
                        │              │  │  khoa,daiDoi   │
                        │              │  │  truong        │
                        │              │  └────────────────┘
                        │              │
                        │              │  ┌────────────────┐
                        │              │  │  Attachment    │
                        │              │  │ ownerType:     │
                        │              │  │ QuyetDinh|     │
                        │              │  │ HoSoSucKhoe    │
                        │              │  └────────────────┘
                        │              │
   ┌────────────────┐   │              │
   │  CanBoQuanLy   │◄──┴── canBo[]    │
   │                │                  │
   │  taiKhoan ─────┼─────────────────►┴───► User
   └────────────────┘                       (one-way: staff → user)

   ┌────────────────┐
   │  HocLieu       │  parent ──► HocLieu (self-ref, folder tree)
   │  uploadedBy ───┼──► User
   └────────────────┘

   Independent: TrucQuanSu, SystemSetting,
                KhaoSatChatLuongNam, GiangVienKhaoSat
```

References (`Schema.Types.ObjectId, ref: 'X'`) are **logical only** — MongoDB does not enforce referential integrity. Cleanup of orphaned references is application-level (most services do not currently do it).

## 4. Indexes

| Collection | Index | Type | Why |
|---|---|---|---|
| `users` | `{ username: 1 }` | unique | Implicit from `unique: true` on the field |
| `sinh_vien` | `{ maSV: 1, hoTen: 1, ngaySinh: 1 }` | unique, **sparse** | Detect duplicate students; sparse so legacy records with missing maSV don't conflict |
| `khoa` | `{ ten: 1 }` | unique, sparse | One faculty per name |
| `dai_doi` | `{ ten: 1, khoa: 1, donViLienKet: 1 }` | unique | Battalion is unique inside (khoa, sending school) |
| `quyet_dinh` | `{ sinhVien: 1, soQD: 1 }` | unique, sparse | One decision per (student, decision-number) |
| `can_bo_quan_ly` | `{ hoTen: 1 }` | non-unique | Sort/filter by name |
| `can_bo_quan_ly` | `{ taiKhoan: 1 }` | unique, **partial** | Linked user account is 1:1. `partialFilterExpression: { taiKhoan: { $type: 'objectId' } }` keeps un-linked rows out of the index entirely (avoids null-collision) |
| `hoc_lieu` | `{ drive: 1, parent: 1, name: 1 }` | unique | One file/folder per name inside (drive, parent) |
| `hoc_lieu` | `{ drive: 1, parent: 1 }` | non-unique | Fast listing inside a folder |
| `attachments` | `{ ownerType: 1, ownerId: 1 }` | unique | One attachment per owning record |
| `systemsettings` | `{ key: 1 }` | unique | Implicit from `unique: true` on the field |
| `khao_sat_chat_luong_nam` | `{ nam: 1 }` | unique | One config per year |
| `giangVien` (survey collection) | `{ nam: 1, hoTen: 1 }` | unique | One instructor per (year, name) |
| `giangVien` (survey collection) | `{ hoTen: 1 }` | non-unique | Sort/filter |
| `truc_quan_su` | `{ ngay: 1 }` | unique | One roster row per date |

**No indexes** on `ho_so_suc_khoe`, `schools`, or `students` (secondary). The health-records collection is small enough today; flag for follow-up if it grows.

When adding a new model, add `schema.index(...)` calls inline (not lazily — Mongoose's auto-index runs at app start, so missing indexes are silent perf regressions).

## 5. Embedded vs referenced — the `diem` choice

`SinhVien.diem` is an **embedded array of subdocuments**, not a separate `diem` collection. The decision predates this doc; the reasons it has held up:

- Reads are always *for one student* (the score-entry UI, the certificate lookup, the report exporter). Embedding lets a single `findOne` return the full record.
- Writes update a single student's grades in one document write — atomic, no transaction needed.
- The total document size stays well under Mongo's 16 MB limit (max ~10 subjects × ~10 fields per student).
- The `pre('validate')` hook on the subdocument auto-computes `tbMon` using integer-precision arithmetic (see `models/sinhVien.js:18-28`), so the average is always consistent with the raw scores.

The API endpoint `/api/diem` is a **logical view** over `SinhVien.diem`: list/create/update/delete operations on grades are implemented by reaching into the parent SinhVien doc and mutating the embedded array. This is also why the docs call it a "grade" model even though no `diem` collection exists.

> **If you ever want to query "all students with `mon = Quân sự` and `tbMon < 5`"**, you must do it via `SinhVien.find({ 'diem.mon': 'Quân sự', 'diem.tbMon': { $lt: 5 } })` and then post-filter in JS, because the match is against the whole array, not the matching element. There is no Diem collection to query directly.

## 6. Field-name conventions

- **Vietnamese camelCase** in the schema definition (`hoTen`, `ngaySinh`, `daiDoi`). This matches the API request/response shape.
- **snake_case Vietnamese** in `collection:` names (`sinh_vien`, `dai_doi`, `quyet_dinh`) so that the raw shell view (`db.sinh_vien.find()`) is readable.
- **English** for boolean/scalar fields that don't have a clear Vietnamese term (`status`, `role`, `mimeType`, `size`).
- **Secondary cluster fields** are snake_case Vietnamese on disk (`ten_truong`, `he_dao_tao`, `ho_va_ten`) because they're owned by a different project. The Mongoose `alias` feature maps `ten ↔ ten_truong` so app code can use camelCase.

## 7. Lifecycle hooks worth knowing

| Model | Hook | What it does |
|---|---|---|
| `SinhVien.diem[]` | `pre('validate')` | Computes `tbMon` from `(thuongXuyen, mieng, giuaHP, hetHP)` per the weight rules for `mon`. Uses integer math (×10) to avoid IEEE-754 drift. |
| `DaiDoi` | `pre('validate')` | Throws a *plain `Error`* (not `HttpError`) if `canBo`, `soQD`, `ngayQD`, `hieuLuc` arrays have different non-zero lengths. The error becomes a 500 via the error envelope. If your client gets a 500 on a `DaiDoi` write, this is almost always the cause. |

No other models have hooks today. Adding one is fine, but throw `HttpError(400, 'VALIDATION_ERROR', …)` so the error envelope produces a 4xx, not 5xx.

## 8. Common gotchas

1. **`Khoa.ten` is a sparse unique index.** Two `Khoa` docs with no `ten` won't collide. Two with the same `ten` will. Set `ten` on every new doc.
2. **`SinhVien` has no `_id` lookup index beyond the default Mongo `_id`.** Searches by `maSV` alone do a collection scan. The unique `{maSV, hoTen, ngaySinh}` index is only used when all three fields are in the query.
3. **`HoSoSucKhoe` has no indexes.** All queries scan. Acceptable today; revisit when row counts climb.
4. **`Attachment` is one-attachment-per-owner** by the unique `{ownerType, ownerId}` index. If you want multiple attachments per QuyetDinh, this schema and index need to change.
5. **`GiangVienKhaoSat` lives in the `giangVien` collection**, not `giang_vien_khao_sat`. This is purely historical; renaming requires a Mongo-side rename and is intentionally deferred.
6. **`DonViLienKet.ten` is an alias for `ten_truong`** on disk. Queries like `DonViLienKet.findOne({ ten: '...' })` work; raw shell queries against the `schools` collection need `ten_truong`.
7. **The secondary connection is fail-fast at import time.** A missing `MONGO_URI2` does not gracefully degrade — the app won't start. Future flexibility (lazy connection, feature flag for the certificate-lookup features) would need a refactor.

## 9. Backups

There is no in-repo backup automation. Recommended (see [`../guides/backups.md`](../guides/backups.md) — to be written):

- `mongodump` from each cluster's URI on a schedule (cron on the VPS or a dedicated backup host).
- Snapshot `backend/uploads/` separately — those binaries are not in Mongo.
- Test the restore path at least once per release cycle.

## 10. See also

- [`docs/backend/schemas/`](../backend/schemas/) — field-by-field per-model docs (some still missing for the 7 newer models)
- [`backend.md`](backend.md) — request lifecycle, where models sit in the layers
- [`auth.md`](auth.md) — how `User.allowedUnits` and `User.teacherScope` shape every query
- [`file-storage.md`](file-storage.md) — what's stored on disk vs in Mongo
