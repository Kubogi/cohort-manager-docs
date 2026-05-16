# Scripts Reference

Every script in the repo, what it does, and when to run it. The canonical npm-scriptable ones live in `backend/package.json`; the rest are standalone files invoked with `node`.

## Backend `package.json` scripts

Run from inside `backend/`.

| Script | Underlying command | What it does | When to run |
|---|---|---|---|
| `npm test` | `node --experimental-vm-modules ./node_modules/jest/bin/jest.js --runInBand` | Runs the backend Jest suite (`backend/src/tests/`). Uses `mongodb-memory-server`, so no real Mongo needed. `--runInBand` keeps tests sequential to avoid port collisions. | Before every PR; in CI when CI exists. |
| `npm run validate:docs` | `node src/tests/utils/schemaValidator.js` | Validates docs schema definitions against the actual Mongoose models. | After any change to a model or schema doc; in a doc-quality check. |
| `npm run create:admin` | `node src/scripts/create-admin.js` | Creates or updates an `admin` user. Prompts for username/password, or accepts `--username`, `--password`. | First time setting up a deployment, or to reset the admin password. |
| `npm run register-and-test:admin2` | `node -e "import('./src/scripts/register-admin2-and-test-diem.js')"` | Registers a second admin user via the public API and exercises the `/api/diem` endpoints. Used for grade-flow regression testing. | Manual sanity check after grade-related code changes. |
| `npm run seed:random-test-data` | `node src/scripts/seed-random-names-and-grades.js` | Populates the primary cluster with synthetic students + grades for UI testing. | On a fresh dev environment when you need data to look at. |

## Frontend `package.json` scripts

Run from inside `frontend/`.

| Script | Underlying command | What it does | When to run |
|---|---|---|---|
| `npm run dev` | `vite` | Starts Vite's dev server on `http://localhost:5173` with HMR. | The default during local development. |
| `npm run build` | `tsc -b && vite build` | Type-checks the whole project then bundles to `frontend/dist/`. Bakes `VITE_API_URL` from `.env` at build time. | Before deploying. CI build verification. |
| `npm run preview` | `vite preview` | Serves the built `dist/` over HTTP for smoke-testing the production bundle locally. | After `npm run build`, to confirm the static bundle works. |
| `npm run lint` | `eslint .` | Lints the whole repo with the flat ESLint config. | Before pushing. CI gate. |
| `npm test` | `vitest` | Runs the Vitest unit suite. **No unit tests are committed today** — running this exits with "no tests found" until someone adds the first one. | Once unit tests start landing. |
| `npm run test:ui` | `vitest --ui` | Opens Vitest's browser UI for live test browsing. | When iterating on a new test file. |
| `npm run test:coverage` | `vitest --coverage` | Runs the suite with V8 coverage. | When inspecting coverage gaps. |
| `npm run cypress:open` | `cypress open` | Cypress GUI runner. The only existing spec is `cypress/e2e/login.cy.ts`. | When developing/exploring an E2E flow. |
| `npm run cypress:run` | `cypress run` | Cypress headless run; suitable for CI. | CI E2E gate when CI exists. |

Note: `tsc --noEmit` is **not** in `package.json` but is the project's standard pre-commit type-check. Run `npx tsc --noEmit` from inside `frontend/` before each commit.

## Standalone backend scripts

These don't have package.json shortcuts. Invoke with `node` from `backend/` (or `cd` to the relevant directory first).

### `backend/src/scripts/create-admin.js`

Creates or upserts an admin user.

```bash
# Interactive
cd backend
node src/scripts/create-admin.js

# Non-interactive
node src/scripts/create-admin.js --username admin --password "Str0ng!"
```

Connects to `MONGO_URI`. Idempotent — re-running with the same username updates the password and role. Exits non-zero on connection failure.

### `backend/src/scripts/seed-random-names-and-grades.js`

Inserts ~20 students with randomly chosen Vietnamese names, distributed across the existing `Khoa` and `DaiDoi` rows. Also seeds `diem[]` for each student.

```bash
cd backend
node src/scripts/seed-random-names-and-grades.js
```

**Idempotency: none.** Re-running creates more students. Use only on dev databases.

### `backend/src/scripts/seed-tongket-grades.js`

Populates end-of-cohort summary grades on already-existing `SinhVien` rows. Useful when developing the "Tổng kết cuối khóa" tab of "Nhập điểm".

```bash
cd backend
node src/scripts/seed-tongket-grades.js
```

Requires students to already exist. Updates `diem[]` in-place.

### `backend/src/scripts/fix-tbMon-precision.js`

One-off migration. Re-saves every `SinhVien` document so the `diem[].pre('validate')` hook recomputes `tbMon` with the current integer-arithmetic formula. Run this if you find legacy data with `tbMon` values that drift from the formula (e.g. `7.499999…`).

```bash
cd backend
node src/scripts/fix-tbMon-precision.js
```

Safe to re-run — recomputed `tbMon` is deterministic from the raw grades.

### `backend/src/scripts/register-admin2-and-test-diem.js`

Hits the running backend's API (POSTs to `/auth/login`, then exercises `/diem` reads). Used as a manual end-to-end test for the grade flow.

```bash
# Backend must be running on http://localhost:3000
cd backend
node -e "import('./src/scripts/register-admin2-and-test-diem.js')"
```

The unusual `node -e import(...)` invocation in `package.json` is to work around ESM/CJS interop on Windows.

### `backend/scripts/seedCertLookup.js`

Populates the **secondary** cluster's `students` collection ([CertificateLookup](../backend/schemas/CertificateLookup.md)) from a hard-coded sample of certificate records. Used to seed test data for the public "Tra cứu" page.

```bash
cd backend
node scripts/seedCertLookup.js
```

Connects to `MONGO_URI2`. Upserts on `truong_id + ma_sinh_vien` — safe to re-run.

### `inspect_excel.js` (root)

Ad-hoc script for inspecting an Excel file's structure (sheet names, header rows). Used when reverse-engineering a new operator template. Edit the file path inline before running.

```bash
node inspect_excel.js
```

Not part of the normal workflow; kept around as a debugging aid.

## Operational scripts in the deployment guide

These are not committed scripts but shell snippets referenced by [`deployment.md`](deployment.md):

- **Cleanup orphan multer temp files** — `find backend/uploads/.tmp -type f -mmin +60 -delete` (run from cron).
- **Mongo backup** — `mongodump --uri="$MONGO_URI" --out=/backups/$(date +%F)`.
- **Uploads backup** — `rsync -a backend/uploads/ /backups/uploads/$(date +%F)/`.

## Conventions for new scripts

If you add a new script:

1. Place it in `backend/src/scripts/<name>.js` if it touches Mongo via the app's models. Place it in `backend/scripts/<name>.js` if it's standalone and bypasses the app (e.g. shells out, talks to a different DB).
2. Make it idempotent if you can. If you can't, name it `seed-…` and document the non-idempotence explicitly.
3. Add a `package.json` entry if anyone will run it more than once.
4. Connect to Mongo at the top, exit non-zero on failure. Don't trust the caller to have Mongo up.
5. Add an entry here.
