# Troubleshooting

Common failure modes during setup and operation, with the diagnosis path. If you can't find your problem here, check the per-area docs in [`../architecture/`](../architecture/) and [`../api/`](../api/), or `pm2 logs student-mgmt-api` if you're on a production box.

## Backend won't start

### `Error: MONGO_URI2 is not defined in environment variables`

**Symptom:** The Node process crashes immediately on startup.

**Why:** [`backend/src/db/secondaryConnection.js`](../../backend/src/db/secondaryConnection.js) throws synchronously at import time when `MONGO_URI2` is missing. There's no graceful degradation.

**Fix:** Add `MONGO_URI2=…` to `.env` (project root). For dev you can point it at the same Mongo instance, different database name:

```
MONGO_URI=mongodb://localhost:27017/[DB_NAME]
MONGO_URI2=mongodb://localhost:27017/[DB_NAME_2]
```

### `MongoServerSelectionError: getaddrinfo ENOTFOUND ...`

**Symptom:** `Failed to start server` with `MongoServerSelectionError`.

**Why:** Mongo URI hostname doesn't resolve, or the port is closed, or Atlas hasn't whitelisted your IP.

**Fix:**
- Confirm `mongosh "$MONGO_URI" --eval "db.runCommand({ ping: 1 })"` works from the same host.
- For Atlas: add the deploying machine's IP under Network Access.
- For self-hosted Mongo on the same VPS: confirm `bindIp: 127.0.0.1` in `/etc/mongod.conf` and the URI uses `127.0.0.1`, not `localhost` (some configurations have the IPv6 form of `localhost` going through IPv6 while Mongo binds IPv4).

### Backend boots but `/api/hello` returns 502 through Nginx

**Why:** PM2 says the process is up, but the port doesn't match what Nginx proxies to.

**Fix:**
- `pm2 logs` to confirm `Server is running on port <port>`.
- Confirm `nginx.conf`'s `proxy_pass http://127.0.0.1:3000;` uses the same port as `env.port` (default 3000).
- `sudo nginx -t` validates Nginx config.

### `npm test` (backend) fails with "Cannot use import statement outside a module"

**Why:** Jest needs `--experimental-vm-modules` to handle ESM. The `test` script in `package.json` already passes it; if you're invoking Jest manually, you may have dropped the flag.

**Fix:** Run `npm test` (uses the script), not `jest` directly.

## Frontend won't connect to backend

### Login form spins forever / "Network Error"

**Why:** `VITE_API_URL` points to a host the browser can't reach.

**Fix:**
- Open browser devtools → Network tab. Find the failing `/auth/login` request. Look at its URL.
- Confirm that URL matches what's in `.env` after the Vite build. Vite bakes `VITE_API_URL` into the bundle at build time, so changes require a `npm run build` (or just restart `npm run dev` in dev mode).
- For local dev: `VITE_API_URL=http://localhost:3000/api`.
- For prod: `VITE_API_URL=https://your-domain/api` (Nginx will proxy `/api/*` to PM2).

### Every API call returns 401 immediately, even right after login

**Why:** `JWT_SECRET` rotated between issuing the token and verifying it. The token was signed by an old secret; verification fails.

**Fix:** Clear `localStorage` (devtools → Application → Local Storage → clear `accessToken`, `refreshToken`, `authUser`). Log in again. The new token will be signed with the current secret.

### "Phiên đăng nhập đã hết hạn" every hour

**Why:** Access token expired after 1 hour (the default `JWT_EXPIRES_IN`). The frontend does **not** call `/api/auth/refresh` — it just clears tokens and redirects. This is expected behavior today; sliding sessions are a planned but unimplemented feature.

**Fix:** Either log in again, or increase `JWT_EXPIRES_IN` (e.g. `8h`) and restart the backend. To get true sliding sessions, implement the refresh-token flow in the frontend; see [`../architecture/auth.md`](../architecture/auth.md#7-token-lifetime-and-the-missing-refresh-flow).

### CORS errors in the browser console

**Why:** The backend's `cors({ origin: true })` should accept any origin. If you're still seeing CORS errors, something is reaching the network *before* the backend (e.g. Nginx, a corporate proxy, a service worker).

**Fix:**
- For dev: confirm `cors({ origin: true, credentials: true })` is still in `app.js`.
- For prod: check the Nginx server block isn't stripping CORS headers from the proxied response.
- Browser service workers can cache old responses with `Access-Control-Allow-Origin: <wrong>`. Clear them via devtools → Application → Service Workers → Unregister.

## API behavior surprises

### Excel import says "0 inserted, 0 duplicates, 0 errors"

**Why:** Either no sheet contained the auto-detected header row (the parser looks for a row whose A/B/C cells are `STT`, `CCCD`, `Mã SV`, case-insensitive), every sheet's name was `SỐ LƯỢNG` (skipped), or the rows below the header were entirely empty.

**Fix:** Open the workbook in Excel. Confirm at least one sheet has a header row with `STT / CCCD / Mã SV / …` and data immediately below. Confirm sheet names are battalion names (`D1`, `D2`, …), not the literal `SỐ LƯỢNG`.

### Excel import created a battalion I didn't expect

**Why:** A typo in the sheet name. The importer auto-creates any `daiDoi` it can't find under the chosen `khoa`.

**Fix:** In the Khóa học page, open the typo'd daiDoi, move its students to the correct one, then delete the typo'd row. Or rename the typo'd `daiDoi.ten` in mongosh and accept the typo's path.

### `quyết định` deleted but student status didn't revert

**Why:** The frontend's decision-processing flow does the revert in a separate request *before* deleting. If the revert request failed (network blip, validation error), the delete should not have proceeded; if it did, the system is inconsistent.

**Fix:** Re-set the student's `trangThai` to `Đang học` manually via the Cơ sở dữ liệu sinh viên page (admin). See [decision-processing.md](../workflows/decision-processing.md).

### Excel export returns a 1 KB empty workbook

**Why:** The filters resolved to zero students. The exporter doesn't error out on empty selections; it returns the template populated with zero rows.

**Fix:** Loosen filters or pick a `khoa` that actually has students.

### `tbMon` looks slightly wrong on old data (e.g. `7.499999`)

**Why:** Legacy data was written before the integer-arithmetic `pre('validate')` hook was added.

**Fix:** Run `node src/scripts/fix-tbMon-precision.js` from inside `backend/` against the live Mongo. The script re-saves every `SinhVien` so the hook recomputes `tbMon`. Safe to re-run.

## File uploads

### 413 on hoc-lieu upload

**Why:** Either the file exceeds the 100 MB per-file limit (`MAX_FILE_SIZE` in `config/storage.js`), the drive is over the 3 GB quota, or Nginx's `client_max_body_size` is too small.

**Fix:**
- Confirm file size; split if it's a giant archive.
- `du -sh backend/uploads/{giaoAn,giaoTrinh,taiLieuThamKhao}/*` to check current drive sizes.
- In production: `client_max_body_size 110M;` in the Nginx server block.

### Download returns 404 but the file exists in the UI

**Why:** Mongo has the `HocLieu` or `Attachment` row, but the on-disk file is missing. Common after a partial restore (Mongo restored, uploads not).

**Fix:** Restore the corresponding `backend/uploads/` snapshot. Or, if the file is genuinely gone, delete the Mongo row to avoid future confusion.

### Temp files piling up in `backend/uploads/.tmp/`

**Why:** Excel imports normally clean up in a `finally` block. If the process was killed (PM2 reload, crash), the orphan file remains.

**Fix:** Set up the cron from [backups.md](backups.md#cleanup-of-tmp):

```cron
0 * * * * deploy find /home/deploy/apps/student-management/backend/uploads/.tmp -type f -mmin +60 -delete
```

## PM2 / production

### `pm2 status` shows `errored` repeatedly

**Why:** The process is crashing on startup. PM2 retries up to `max_restarts` (10) before giving up.

**Fix:**
- `pm2 logs student-mgmt-api --lines 100` — find the actual exception.
- 90% of the time it's a missing or wrong env var (`MONGO_URI2`, `JWT_SECRET`).
- Fix `.env`, then `pm2 reload student-mgmt-api`.

### Logs not being captured / filling disk

**Why:** PM2 writes to `~/.pm2/logs/<name>-{out,error}.log`. Without rotation, they grow unbounded.

**Fix:** Install `pm2-logrotate`:

```bash
pm2 install pm2-logrotate
pm2 set pm2-logrotate:max_size 50M
pm2 set pm2-logrotate:retain 14
```

### Restart on reboot doesn't bring PM2 back

**Why:** `pm2 startup` was never run, or `pm2 save` wasn't run after the last config change.

**Fix:**
```bash
pm2 startup systemd
# follow the printed sudo command
pm2 save
```

## Auth / permissions

### Admin can't see a record a viewer can

**Why:** Almost never. Admin is unscoped — both `applyUnitScope` and `applyTeacherScope` return the filter unchanged for `role: 'admin'`. If this happens, you probably have a bug in a service that's adding filters outside those helpers. Check the service file directly.

### Teacher sees a student outside their `teacherScope`

**Why:** Bug, not configuration. The teacher scope is enforced server-side by `applyTeacherScope`; client-side decoding of the JWT is only for UI gating. If you see a leak, file an issue with the JWT payload and the offending student's IDs.

### Staff with `allowedUnits.allowAll.khoa = true` only sees their listed khoas

**Why:** Probably an old token without the `allowAll` field. The JWT carried the user's `allowedUnits` at issue time; if the user's units were updated server-side after login, the token is stale.

**Fix:** Log out and back in. The new token will carry the current `allowedUnits`.

## See also

- [`getting-started.md`](getting-started.md) — what should work on day one
- [`deployment.md`](deployment.md) — what should work in production
- [`environment-variables.md`](environment-variables.md) — every variable, where it's read, default
- [`../architecture/auth.md`](../architecture/auth.md) — full auth model
- `pm2 logs student-mgmt-api` — first stop for any production runtime issue
