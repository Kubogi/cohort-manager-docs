# /api/settings

Key–value system configuration. Today the only configured key is `training-schedule-link` (the URL shown on the "Lịch trình đào tạo" page). Future settings should follow the same shape (one route per key, admin gating inside the route file).

Role gating is **inside** the route file, not at the mount. For `training-schedule-link`: reads are `admin`/`staff`/`viewer` (teachers are excluded — this collides with the "Lịch trình đào tạo" page being visible to all roles; if a teacher hits that page they will receive `403` from this endpoint). Writes are `admin`-only.

**Source files**: [systemSetting.route.js](../../../../backend/src/routes/systemSetting.route.js), [systemSetting.controller.js](../../../../backend/src/controllers/systemSetting.controller.js), [systemSetting.service.js](../../../../backend/src/services/systemSetting.service.js), [SystemSetting schema](../../../backend/schemas/SystemSetting.md).

## Endpoints

### `GET /api/settings/training-schedule-link`

Returns the configured iframe URL.

**Response**: `{ data: { link: "https://..." } }`. `link` is `""` if not yet configured.

**Roles**: `admin`, `staff`, `viewer`. **Teachers are not allowed** — calling this endpoint with a teacher token returns `403 FORBIDDEN`.

### `PATCH /api/settings/training-schedule-link`

Update the URL. Admin-only.

**Body**: `{ link: string }`. The frontend regex enforces `http(s)://`; the backend takes the value as-is.

**Response**: `{ data: { link: "..." } }` with the new value.

**Errors**: `400 VALIDATION_ERROR`, `401 UNAUTHORIZED`, `403 FORBIDDEN`.

## Cross-cutting notes

- **Upserts by `key`.** First write creates the row; subsequent writes update it. No DELETE — a "blank" setting is `link: ""`.
- **No history.** Settings overwrite in place. If you need an audit trail, capture it externally.
- **Frontend cache.** `settingsAPI.getTrainingScheduleLink` in [`api/client.ts`](../../../../frontend/src/api/client.ts) is called on every page mount of "Lịch trình đào tạo" — no client-side caching, no stale state.

## Adding a new setting

1. Pick a `key` (kebab-case advisable, e.g. `notice-banner-text`).
2. Add `GET /api/settings/<key>` and `PATCH /api/settings/<key>` handlers that upsert on `{ key }` (see existing implementations).
3. Add role gating inside the route — admin for writes, broader for reads.
4. Add a typed helper in `frontend/src/api/client.ts#settingsAPI`.
5. Document here.

## Related

- [SystemSetting schema](../../../backend/schemas/SystemSetting.md)
- [`docs/frontend/pages/quan-ly-tai-khoan/quan-ly-link.md`](../../../frontend/pages/quan-ly-tai-khoan/quan-ly-link.md) — the admin UI
- [`docs/frontend/pages/thong-tin-chung/lich-trinh-dao-tao.md`](../../../frontend/pages/thong-tin-chung/lich-trinh-dao-tao.md) — the consumer
