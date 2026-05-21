# TrucQuanSu Schema

**Source**: [backend/src/models/trucQuanSu.js](../../../backend/src/models/trucQuanSu.js)
**Collection**: `truc_quan_su` (primary cluster)
**Last verified**: 2026-05-19

Daily military-duty roster, **scoped per Khóa**. One row per `(khoa, ngay)` pair — the compound unique index enforces this. Each row records who is on duty in seven named roles plus a free-text remarks field.

## Fields

| Field | Type | Required | Default | Description |
|---|---|---|---|---|
| `khoa` | ObjectId (ref `Khoa`) | **Yes** | — | The Khóa that owns this schedule entry. Each Khóa has its own duty calendar. |
| `ngay` | Date | Yes | — | The duty date. Unique within a Khóa (two khóa can share the same date). |
| `trucLanhDao` | String | No | `''` | "Leadership on duty" — typically a senior officer's name. |
| `trucChiHuy` | String | No | `''` | "Command on duty". |
| `truongKhungNhaD1D2` | String | No | `''` | "Frame leader of D1/D2 dormitories". |
| `trucBan` | String | No | `''` | "Duty officer". |
| `trucQuanYNgay` | String[] | No | `['', '']` | "Medic on day shift" — two slots (the array length is fixed at 2 by the default; do not push). |
| `trucQuanYDem` | String | No | `''` | "Medic on night shift". |
| `trucDienNuoc` | String | No | `''` | "Electricity & water on duty". |
| `donViKiemSoatQuanSu` | String | No | `''` | "Military police unit on duty". |
| `ghiChu` | String | No | `''` | Free-text notes. |
| `createdAt`, `updatedAt` | Date | Auto | — | Mongoose timestamps. |

## Indexes

| Fields | Type | Why |
|---|---|---|
| `{ khoa: 1, ngay: 1 }` | **unique** | One roster per `(khoa, ngay)`. Two khóa can hold an entry for the same date. |

## Behavior notes

- **Per-Khóa storage.** Each Khóa has its own duty calendar. The frontend tab requires a Khóa to be selected before any data is shown; `list()` returns an empty page when called without `khoa`.
- **Date semantics.** `ngay` is a Mongo `Date`. Queries should compare with `Date` values, not raw strings. The frontend posts an ISO date string; Mongoose coerces.
- **`trucQuanYNgay` is an array of two strings.** The Mongoose default is `['', '']`. If you mutate this in the UI, send back an array of length 2; otherwise rows look inconsistent in the table view.
- **Visibility.** Mounted under `/api/truc-quan-su` with `authorize(['admin', 'staff', 'viewer'])` — `teacher` cannot access this resource.

## Migration history

- **2026-05-19.** Schema migrated from a global single-calendar model (`unique` on `ngay`) to per-Khóa storage. The migration was destructive: existing rows were wiped via [`backend/scripts/drop-truc-quan-su.js`](../../../backend/scripts/drop-truc-quan-su.js) (chosen over backfill since recreating a calendar from the Excel import is trivial).
- **2026-05-22.** Deployments that survived the 2026-05-19 row wipe were still carrying the *index* from the old schema — a `{ngay: 1, unique: true}` left over even after the Mongoose model dropped it. Mongoose's `autoIndex` only adds missing indexes, never drops stale ones, so the legacy index silently blocked cross-Khóa imports: the second khoa's rows all collided with the first khoa's dates and reported as duplicates. Fix shipped via [`backend/src/scripts/drop-trucQuanSu-legacy-ngay-index.js`](../../../backend/src/scripts/drop-trucQuanSu-legacy-ngay-index.js); run once per environment (idempotent).

## Related

- [`/api/truc-quan-su`](../../api/README.md#military-duty-roster-apitruc-quan-su) — endpoints
- [`/docs/glossary.md`](../../glossary.md#operations-and-admin) — "Trực quân sự" term reference
