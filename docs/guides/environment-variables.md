# Environment Variables

Every variable the code reads, where it's read from, and the default if you leave it blank. The backend and frontend share **one** `.env` file at the project root — see [`getting-started.md`](getting-started.md#3-configure-env) for how that's wired.

The canonical sources are:

- Backend — [`backend/src/config/env.js`](../../backend/src/config/env.js). 10 process.env reads.
- Frontend — [`frontend/src/api/client.ts`](../../frontend/src/api/client.ts). 1 `import.meta.env` read.
- Vite reads `.env` from the parent directory via `envDir: '..'` in [`frontend/vite.config.ts`](../../frontend/vite.config.ts).

## Reference

| Variable | Required? | Default | Read by | Purpose |
|---|---|---|---|---|
| `PORT` | no | `3000` | backend ([env.js:8](../../backend/src/config/env.js)) | TCP port the Express server binds. PM2 sets this from `ecosystem.config.js` in production. |
| `MONGO_URI` | **yes** | `''` | backend ([env.js:9](../../backend/src/config/env.js), [db.js](../../backend/src/config/db.js)) | Primary MongoDB connection string. `connectPrimary()` rejects empty values; the server exits non-zero. |
| `MONGO_URI2` | **yes** | `''` | backend ([env.js:10](../../backend/src/config/env.js), [db/secondaryConnection.js](../../backend/src/db/secondaryConnection.js)) | Secondary MongoDB connection string for partner-school + certificate lookup data. **Throws synchronously at import time** if missing — the process crashes during module loading, before `connectPrimary` even runs. Effectively required. For dev you can point it at the same Mongo instance under a different database. |
| `CORS_ORIGIN` | no | `'*'` | backend ([env.js:11](../../backend/src/config/env.js)) | Read into `env.corsOrigin` but **not currently applied** — [`app.js`](../../backend/src/app.js) hard-codes `cors({ origin: true, credentials: true })`. Setting this has no effect today; the env var exists for future use. |
| `NODE_ENV` | no | `'development'` | backend ([env.js:12](../../backend/src/config/env.js)) | Read into `env.nodeEnv`. Used by libraries (mongoose, helmet) for default behavior; the app code does not branch on it. |
| `JWT_SECRET` | yes for prod | `'dev-secret'` | backend ([env.js:13](../../backend/src/config/env.js), [utils/auth.js](../../backend/src/utils/auth.js)) | Signs access tokens (HS256). **Set this in production** — leaving it as `dev-secret` means anyone with the source can forge tokens. Rotating this instantly invalidates every issued access token. |
| `JWT_EXPIRES_IN` | no | `'1h'` | backend ([env.js:14](../../backend/src/config/env.js)) | Access-token lifetime. Accepts any `jsonwebtoken` duration string (`'30m'`, `'2h'`, `'1d'`). |
| `JWT_REFRESH_SECRET` | yes for prod | `'dev-refresh-secret'` | backend ([env.js:15](../../backend/src/config/env.js)) | Signs refresh tokens. Same rotation/security caveat as `JWT_SECRET`. Use a distinct value from `JWT_SECRET`. |
| `JWT_REFRESH_EXIRES_IN` | no | `'7d'` | backend ([env.js:16](../../backend/src/config/env.js)) | Refresh-token lifetime. ⚠ **The env var name has a typo (`EXIRES` not `EXPIRES`).** This is the *real* name expected by the code. Use the typo'd name in `.env` and in `ecosystem.config.js`. Setting `JWT_REFRESH_EXPIRES_IN` (correct spelling) does nothing — the code reads `JWT_REFRESH_EXIRES_IN`. |
| `LOCAL` | no | unset | backend ([env.js:17](../../backend/src/config/env.js)) | When `LOCAL=1`, `env.isLocal` is `true`. Currently read but no code path branches on it — reserved for future feature flags. |
| `VITE_API_URL` | yes for non-default | `'http://localhost:3000/api'` | frontend ([api/client.ts:8](../../frontend/src/api/client.ts)) | Base URL the SPA hits. The client auto-prefixes `http://` if missing, so `localhost:3000/api` works. Must be **the URL the browser can reach** — `localhost:…` works only when the SPA and the user share the same machine. For a VPS deploy, set this to `https://your-domain/api`. |

## Variables that do *not* exist (despite appearing in older docs)

- `JWT_REFRESH_EXPIRES_IN` (correct English spelling) — not read. See `JWT_REFRESH_EXIRES_IN` above.
- `DATABASE_URL` — not read. Use `MONGO_URI`.
- `API_PORT` — not read. Use `PORT`.

## Vite-specific rules

Vite only exposes variables prefixed with `VITE_` to the bundle. Adding a non-prefixed variable to `.env` makes it available to the backend but **invisible to the frontend**. If you need a shared value in both, expose it twice (with and without the `VITE_` prefix) — there is no automatic mirroring.

## Where `.env` lives, where it does not

| Location | Purpose | In git? |
|---|---|---|
| `<root>/.env` | The active local configuration. **Read by both backend and frontend.** | **No** (`.gitignore`'d) |
| `<root>/example.env` | Template with empty values for every required key. Commit this; do not commit the real `.env`. | Yes |
| `backend/.env` | Not read by anything. Don't put one here. | — |
| `frontend/.env` | Not read by anything. Don't put one here. | — |

If a stray `.env` exists in either subdirectory, it has no effect — the backend looks at `path.resolve('../.env')` from inside `backend/`, and the frontend's Vite config explicitly sets `envDir: '..'`.

## PM2 in production

PM2 does not read `.env` automatically. Pick one of:

1. **Reference `.env` from `ecosystem.config.js`** — `require('dotenv').config({ path: '../.env' })` at the top of the ecosystem file, then `env: { ...process.env }` for the app.
2. **Pass `env: { ... }` inline** in `ecosystem.config.js` (duplicating the variables there). Useful if you want different values for `pm2 start --env production`.
3. **Use `pm2 start --env-file ../.env`** if your PM2 version supports it.

Option 1 is the convention currently documented in [`deployment.md`](deployment.md). Whichever you pick, make sure `JWT_REFRESH_EXIRES_IN` (with the typo) is included verbatim, or refresh tokens silently fall back to the 7-day default.

## Rotation playbook

| Variable | Effect of rotation | When to rotate |
|---|---|---|
| `JWT_SECRET` | Every issued access token becomes invalid immediately; all users are forced to re-login. | On suspected leak; on every staff offboarding for high-trust deployments. |
| `JWT_REFRESH_SECRET` | Every issued refresh token becomes invalid. Effect on UX: same as above today (the frontend isn't using refresh tokens), but matters if you wire up refresh later. | Same as `JWT_SECRET`. |
| `MONGO_URI` / `MONGO_URI2` | Backend restarts on the new cluster. No client-side impact. | On cluster migration; on credential rotation. |
| `VITE_API_URL` | Frontend bundle must be rebuilt — Vite bakes this into the build. A change here is **not** picked up by `pm2 reload`; you have to `npm run build` and re-deploy `dist/`. | On API host change. |

Keep a separate secret store (1Password, Vault, AWS Secrets Manager) so the production `.env` is reproducible — losing `JWT_SECRET` is recoverable (force re-login) but losing `MONGO_URI` is not.
