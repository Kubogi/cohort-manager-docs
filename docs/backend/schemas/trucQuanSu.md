# TrucQuanSu Schema

**Source**: [backend/src/models/trucQuanSu.js](../../../backend/src/models/trucQuanSu.js)

**Collection**: `truc_quan_su` (primary cluster)

**Last verified**: 2026-05-16

Daily military-duty roster. One row per **calendar date** — the unique index on `ngay` enforces this. Each row records who is on duty in seven named roles plus a free-text remarks field.

## Fields

| Field | Type | Required | Default | Description |
|---|---|---|---|---|
| `ngay` | Date | Yes | — | The duty date. Unique. |
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
| `{ ngay: 1 }` | **unique** | One roster per date. Implicit from `unique: true` plus an explicit `schema.index`. |

## Behavior notes

- **Date semantics.** `ngay` is a Mongo `Date`. Queries should compare with `Date` values, not raw strings. The frontend posts an ISO date string; Mongoose coerces.
- **`trucQuanYNgay` is an array of two strings.** The Mongoose default is `['', '']`. If you mutate this in the UI, send back an array of length 2; otherwise rows look inconsistent in the table view.
- **Visibility.** Mounted under `/api/truc-quan-su` with `authorize(['admin', 'staff', 'viewer'])` — `teacher` cannot access this resource.

## Related

- [`/api/truc-quan-su`](../../api/README.md#military-duty-roster-apitruc-quan-su) — endpoints
- [`/docs/glossary.md`](../../glossary.md#operations-and-admin) — "Trực quân sự" term reference
