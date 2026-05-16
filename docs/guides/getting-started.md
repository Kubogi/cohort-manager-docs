# Getting Started

This guide takes you from a fresh clone to a running local instance with an admin account you can log in as. Follow it linearly; total elapsed time on a healthy machine is ~10 minutes (most of which is `npm install`).

For production deployment to a VPS, see [`deployment.md`](deployment.md).

## 1. Prerequisites

| Tool | Version | Why |
|---|---|---|
| Node.js | ≥ 22 LTS | Backend uses ES modules + Express 5; frontend uses Vite 7. Older Node fails on `import` of `.js`. |
| npm | ≥ 10 | Bundled with Node 22. |
| MongoDB | A reachable cluster (primary + secondary). MongoDB Atlas Free Tier is enough for development. | Two URIs are required — see step 3. |
| Git | any recent | Cloning + history. |

Optional but recommended:

| Tool | Why |
|---|---|
| `mongosh` | Inspect the database when something looks wrong. |
| A REST client (curl, Insomnia, Postman) | Hit the API directly. |
| Visual Studio Code | The repo has a `.vscode/` and some TypeScript niceties. |

## 2. Clone the repo

```bash
git clone <repo-url> student-management
cd student-management
```

The top of the tree looks like:

```
backend/    Express + Mongoose API
frontend/   React + react-admin SPA
docs/       You're reading these
.env        ← you create this in step 3
example.env ← template
```

## 3. Configure `.env`

The backend and frontend share a single `.env` file at the **project root** (not inside `backend/` or `frontend/`). Backend reads it via `dotenv` (`backend/src/config/env.js` resolves `path.resolve('../.env')` relative to the backend's cwd). Frontend reads it via Vite (`vite.config.ts` sets `envDir: '..'`).

```bash
cp example.env .env
```

Then edit `.env`. The minimum set for local development:

```
MONGO_URI=mongodb://localhost:27017/[DB_NAME]
MONGO_URI2=mongodb://localhost:27017/[DB_NAME_2]
VITE_API_URL=http://localhost:3000/api
JWT_SECRET=<any random ≥ 32 chars>
JWT_REFRESH_SECRET=<any other random ≥ 32 chars>
LOCAL=1
```

For the full table of every variable the code reads, see [`environment-variables.md`](environment-variables.md). Key points:

- `MONGO_URI2` is **required**, not optional. The secondary connection (`backend/src/db/secondaryConnection.js`) throws synchronously at import time if this is missing — the server won't even start. For dev you can point it at the same Mongo instance under a different database name, as above.
- `VITE_API_URL` can omit the `http://` prefix; `frontend/src/api/client.ts` auto-prefixes. `localhost:3000/api` works.
- `LOCAL=1` is read into `env.isLocal` but currently has no effect on routing. Leave it set; it's there for future feature flags.

> The repo ships an `example.env` with placeholders. **Do not** commit your real `.env` — `.gitignore` excludes it. If your `.env` contains real credentials and you suspect it might have been committed, rotate those credentials before continuing.

## 4. Install dependencies

Backend and frontend each have their own `package.json`. The root `package.json` is currently `{}` and has no workspace scripts.

```bash
# Backend
cd backend
npm install
cd ..

# Frontend
cd frontend
npm install
cd ..
```

Expect ~150 MB of node_modules in each subdirectory. Network-limited environments should pre-populate `~/.npm` cache.

## 5. Seed an admin account

Without an admin user, nothing else works — `/api/auth/register` requires admin auth, so you can't bootstrap from the UI. Use the seed script:

```bash
cd backend
npm run create:admin
```

This runs [`backend/src/scripts/create-admin.js`](../../backend/src/scripts/create-admin.js), which prompts (or accepts `--username` and `--password` flags) and upserts an `admin` role user into the primary cluster's `users` collection.

```bash
# Non-interactive variant
npm run create:admin -- --username admin --password "Str0ngPass!"
```

If this exits with `MongoServerSelectionError`, your `MONGO_URI` is wrong or unreachable. Confirm with:

```bash
mongosh "$MONGO_URI" --eval "db.runCommand({ ping: 1 })"
```

## 6. Start the backend

```bash
cd backend
node src/server.js
```

You should see:

```
Server is running on port 3000
```

If you see `Failed to start server:` with a Mongo error, see [troubleshooting.md](troubleshooting.md) or check the `MONGO_URI` / `MONGO_URI2` values.

The server listens on `env.port` (default `3000`, overridable via `PORT`). It does **not** auto-restart on file changes — for that, use `node --watch src/server.js` (Node 22 supports `--watch` natively) or `nodemon`.

Verify the API is reachable:

```bash
curl http://localhost:3000/api/hello
# → "Ciallo ~ (∠・ω< )⌒★"
```

## 7. Start the frontend (new terminal)

```bash
cd frontend
npm run dev
```

You should see Vite's banner:

```
VITE v7.x.x  ready in NNN ms

  ➜  Local:   http://localhost:5173/
  ➜  Network: …
  ➜  press h + enter to show help
```

Open `http://localhost:5173/` in a browser. You should see the login screen.

## 8. Log in

Use the admin credentials from step 5. After successful login the dashboard renders the Vietnamese menu down the left side.

What you should be able to do as admin:

- Open **Quản lý tài khoản → Quản lý danh sách người dùng** to create more users (staff, viewer, teacher) without re-running the seed script.
- Open **Quản lý tài khoản → Quản lý link** (admin-only) to configure the training-schedule iframe URL.
- Open **Thông tin chung → Khóa học** to set up your first cohort.

If the menu is empty or the page seems frozen, open the browser devtools network tab. Common issues:

- All `/api/*` requests failing CORS — your `VITE_API_URL` doesn't match the backend's host. Fix `.env` and restart Vite.
- All `/api/*` requests returning 401 — the access token isn't being attached. Clear localStorage in devtools and re-login.

## 9. (Optional) Seed test data

The repo ships several seed scripts. From `backend/`:

```bash
npm run seed:random-test-data           # creates students with random names + grades
node src/scripts/seed-tongket-grades.js  # populates end-of-cohort summary grades
node scripts/seedCertLookup.js           # mirrors a few students into the secondary cluster for certificate lookup testing
```

See [`scripts.md`](scripts.md) for the full inventory.

## 10. Run the test suites (optional)

```bash
# Backend — Jest + supertest + mongodb-memory-server (no external Mongo needed)
cd backend
npm test

# Frontend — Cypress E2E
cd frontend
npm run cypress:open    # interactive runner
# OR
npm run cypress:run     # headless
```

The frontend's Vitest setup exists (`npm run test`) but no unit tests have been added yet — this command runs and exits cleanly with zero tests.

## 11. Where to next

- **Read the architecture overview** — [`docs/architecture/overview.md`](../architecture/overview.md). Understand the request lifecycle before making backend changes.
- **Read the styling guide** — [`frontend/STYLING_GUIDE.md`](../../frontend/STYLING_GUIDE.md). Required reading before adding any UI.
- **Read the auth model** — [`docs/architecture/auth.md`](../architecture/auth.md). The role + scope system is non-obvious.

## 12. Common first-time problems

| Symptom | Likely cause | Fix |
|---|---|---|
| `Cannot find module ...` on backend startup | Forgot `npm install` in `backend/` | `cd backend && npm install` |
| `Error: MONGO_URI2 is not defined in environment variables` | `.env` missing the secondary URI | Add `MONGO_URI2=…` to `.env` |
| `MongoServerSelectionError` | Mongo not running, or wrong URI | Start mongod / fix URI / check Atlas IP allowlist |
| Frontend hangs on the login screen | `VITE_API_URL` points to a wrong/unreachable host | Fix `.env`, restart `npm run dev` |
| 401 immediately after login | JWT secret mismatch (you regenerated `.env` after the previous token was issued) | Clear browser localStorage, log in again |
| "Phiên đăng nhập đã hết hạn" | Access token expired after 1 h | Log in again. There's no auto-refresh on the client today. |
| `npx tsc --noEmit` errors after pulling | Dependency drift | `cd frontend && rm -rf node_modules && npm install` |
