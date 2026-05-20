# Migration: DaiDoi aligned arrays → CanBoQuanLy.phanCong[]

**Script**: [`backend/scripts/migrate-cbql-phancong.js`](../../backend/scripts/migrate-cbql-phancong.js)
**Status**: One-shot migration. Idempotent.
**When to run**: Once per environment, after deploying the schema changes (`feat(cbql): phanCong array …` and `feat(daiDoi): drop aligned arrays …`).

---

## Why

Until May 2026, CBQL ↔ DaiDoi assignments lived as six aligned arrays on `DaiDoi` (`canBo[]`, `soQD[]`, `ngayQD[]`, `hieuLuc[]`, `tkpk[]`, `ghiChu[]`). The schema enforced "all six arrays equal length" via a `pre('validate')` hook. This made the "CBQL attached to a khoa without a daiDoi" use case impossible to express.

The new model lifts those values onto `CanBoQuanLy.phanCong[]` — one entry per khoa, holding `{ khoa, daiDoi?, soQD, ngayRaQD, hieuLuc, tkpk, ghiChu }`. `daiDoi: null` represents the orphan / khoa-only case.

This migration walks every `DaiDoi` doc, expands each aligned-array slot into a `CanBoQuanLy.phanCong` entry, then unsets the six legacy fields off the DaiDoi.

---

## What the script does

1. Reads every `dai_doi` document via raw collection access (the Mongoose schema no longer declares the legacy fields, so we bypass it).
2. For each non-empty `canBo[i]` slot:
   - Looks up the referenced `CanBoQuanLy`.
   - Skips if a `phanCong` entry for `(cbqlId, khoa)` already exists (idempotency).
   - Otherwise, appends a `phanCong` entry with the values from `soQD[i]`, `ngayQD[i]`, `hieuLuc[i]`, `tkpk[i]`, `ghiChu[i]`.
3. `$unset`s `canBo`, `soQD`, `ngayQD`, `hieuLuc`, `tkpk`, `ghiChu` from every `dai_doi` doc.
4. Returns `{ scannedDaiDois, entriesAdded, skipped, daiDoiCleanedUp }` for logging.

---

## How to run

```powershell
cd backend
node --env-file=../.env scripts/migrate-cbql-phancong.js
```

The script reads `MONGO_URI` from the env. Expected output:

```
Migration complete: { scannedDaiDois: 27, entriesAdded: 42, skipped: 0, daiDoiCleanedUp: 27 }
```

Re-running is safe — `entriesAdded` will be `0` because each `(cbqlId, khoa)` pair only gets one phanCong entry.

---

## Skipped slots

- A `canBo[i]` referencing a CBQL that no longer exists → counted in `skipped`, no entry added.
- A `canBo[i]` whose CBQL already has a `phanCong` entry for `(khoa=dd.khoa)` → counted in `skipped`. This protects re-runs after partial failure.

---

## Rollback

There is no automatic rollback. If the migration introduces bad data, the simplest recovery is:

1. Restore the DaiDoi docs from a pre-migration backup.
2. Set every CBQL's `phanCong` to `[]` via `db.can_bo_quan_ly.updateMany({}, { $set: { phanCong: [] } })`.

If you only need to reset a single CBQL, PATCH it via the API with `phanCong: []`.

---

## Verification

After running, sanity-check a few CBQL:

```javascript
// In mongosh
db.can_bo_quan_ly.findOne({ 'phanCong.0': { $exists: true } })
// Should show a phanCong[] array with at least one entry whose khoa + daiDoi match a DaiDoi doc.

db.dai_doi.findOne({ canBo: { $exists: true } })
// Should return null — every legacy aligned-array field has been unset.
```

In the UI, open `/thong-tin-chung/khoa-hoc`, pick a khóa, and confirm the CBQL table populates with the same rows as before the migration.
