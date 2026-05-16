# SystemSetting Schema

**Source**: [backend/src/models/SystemSetting.js](../../../backend/src/models/SystemSetting.js)

**Collection**: `systemsettings` (Mongoose default; pluralization of model name. **Not** `system_settings`)

**Last verified**: 2026-05-16

Key–value store for app-wide runtime configuration that an admin can change without redeploying. Currently used for the **training-schedule iframe URL** displayed on the "Lịch trình đào tạo" page. Future settings should follow the same pattern (add a new `key`, write to it via a dedicated route).

## Fields

| Field | Type | Required | Unique | Default | Description |
|---|---|---|---|---|---|
| `key` | String | Yes | Yes | — | Identifier for the setting (trimmed). |
| `value` | String | No | — | `''` | Value (trimmed). All values are strings — store JSON-encoded payloads if you need structure. |
| `updatedBy` | ObjectId → User | No | — | — | Last user who modified the value. |
| `createdAt`, `updatedAt` | Date | Auto | — | — | Mongoose timestamps; `versionKey: false` is set so `__v` is omitted. |

## Indexes

| Fields | Type | Why |
|---|---|---|
| `{ key: 1 }` | **unique** | Each setting key exists exactly once. Implicit from `unique: true` plus an explicit `schema.index`. |

## Known keys

| Key | Value semantics | Set via |
|---|---|---|
| `training-schedule-link` | URL string the SPA renders inside an iframe on the "Lịch trình đào tạo" page. | `PATCH /api/settings/training-schedule-link` (admin) |

Adding a new key:

1. Add a route under `/api/settings/<key>` in [`backend/src/routes/systemSetting.route.js`](../../../backend/src/routes/systemSetting.route.js) with role gating.
2. Add a controller + service method that does an upsert on `{ key }`.
3. Add a frontend helper in `frontend/src/api/client.ts#settingsAPI`.

## Related

- [`/api/settings/*`](../../api/README.md#system-settings-apisettings)
- [Glossary: `quanLyLink`](../../glossary.md#operations-and-admin) — the admin-only UI that edits these
