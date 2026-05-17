# CI Deploy (GitHub Actions)

Automated deploy on every push to `main`, defined in [`.github/workflows/deploy.yml`](../../.github/workflows/deploy.yml).

This guide covers the **automated** path. For the one-time VPS bootstrap (Nginx, PM2, MongoDB, Certbot, firewall), see [`deployment.md`](deployment.md) — the CI deploy assumes that work is already done.

## What it does

On every push to `main` (or a manual trigger from the Actions tab), GitHub Actions opens an SSH session to the VPS and runs:

```bash
set -e
cd /var/www/student-management
git pull origin main
cd backend  && npm ci --omit=dev
cd ../frontend && npm ci && npm run build
pm2 reload site-backend2
systemctl restart nginx
```

| Step | Purpose |
|---|---|
| `set -e` | Stop the script on the first failure. Without this, a failed build still ran `pm2 reload` against the new commit. |
| `git pull origin main` | Fast-forward `main` on the VPS. The repo is cloned to `/var/www/student-management` during the initial deployment. |
| `npm ci --omit=dev` (backend) | Production install. Skips dev-only deps like Jest. Required when `backend/package-lock.json` changes. |
| `npm ci` (frontend) | Full install. Vite needs its dev deps to build. |
| `npm run build` | Generates `frontend/dist/`. Nginx serves this directory as the static root. |
| `pm2 reload site-backend2` | Drains in-flight HTTP requests on the backend, then swaps the process. Brief interruption but no dropped requests. |
| `systemctl restart nginx` | Picks up any changes to `frontend/dist/` and re-opens log files. |

Trigger: any push to `main`, **or** a manual run from **Actions → Deploy → Run workflow**.

## Required GitHub secrets

Configure under **Repository Settings → Secrets and variables → Actions**:

| Secret | Value | Notes |
|---|---|---|
| `VPS_HOST` | Public IP or hostname of the VPS | e.g. `203.0.113.42` or `cohort.example.com` |
| `VPS_USER` | SSH user on the VPS | In this deployment: **`root`**. PM2 and the repo are owned by root; the workflow assumes that. |
| `VPS_SSH_KEY` | Private key (full PEM, including header/footer) | Paste the contents of `id_ed25519` or `id_rsa`, not the path. |
| `VPS_SSH_PASSPHRASE` | Passphrase that protects `VPS_SSH_KEY` | Required because the deploy key is passphrase-protected. If you ever generate an unprotected key, leave this secret empty (don't delete it) — the action treats missing as no-passphrase. |

To generate a deploy-only key (passphrase-protected):

```bash
# On any machine. Enter a strong passphrase when prompted.
ssh-keygen -t ed25519 -C "github-actions-deploy" -f deploy_key

# Add the public half to the VPS (root login)
ssh-copy-id -i deploy_key.pub root@<VPS_HOST>

# Paste the private half (`deploy_key`) into VPS_SSH_KEY
# Paste the passphrase you entered above into VPS_SSH_PASSPHRASE
cat deploy_key
```

Do **not** reuse a personal SSH key — if the GitHub secret leaks, you can rotate only the deploy key without touching your laptop. The passphrase is a defence-in-depth measure: even if the GitHub Actions secrets log were somehow exposed (e.g. a malicious workflow injection), the leaked private key on its own would be unusable.

## VPS prerequisites

The workflow expects the following on the VPS before its first run:

1. Repo cloned at `/var/www/student-management`. Root owns it, so writability is not a concern.
2. A working `.env` at the repo root (Vite reads it; the workflow does not provision it).
3. PM2 already running a process named **`site-backend2`** that loads the backend. The workflow only *reloads* it; it cannot start it from cold. Since SSH is as root, `pm2` here is root's PM2 daemon — `pm2 list` from a root shell must show `site-backend2`.
4. Node 22+ and npm available on `PATH` for the non-login SSH shell. If `npm` isn't found, set it explicitly via root's `~/.bashrc`, or invoke the absolute path in the workflow.

Because the SSH user is root, the workflow does not need `sudo` for `systemctl restart nginx` — it runs directly.

If any of these are missing, the first push to `main` will fail loudly — read the failed job log in the Actions tab to see exactly which step blew up.

## Operating it

### Manually trigger a deploy

Actions → **Deploy** → **Run workflow** → select `main` → **Run workflow**. Useful when:

- You want to redeploy without a code change (e.g. after editing the VPS-side `.env`).
- A previous deploy failed mid-script and the VPS is in an inconsistent state — rerun once the underlying problem is fixed.

### Skip the deploy

The workflow runs on **every** push to `main`. To skip:

- Land the change on a branch and merge later — pushes to other branches are ignored.
- Or include `[skip ci]` in the commit message subject — GitHub honours this without changes to the workflow file.

### Watch a deploy in progress

The Actions tab streams the SSH script output in near-real-time. Useful lines to grep mentally:

- `npm ci` warnings — usually safe to ignore.
- `error: Your local changes to the following files would be overwritten by merge` — the VPS working tree was edited manually. Resolve by SSHing in and `git stash` / `git reset --hard origin/main` before retrying.
- `[PM2] reload` confirms the backend swap actually happened.

### Roll back

The workflow has no built-in rollback. To revert:

```bash
ssh root@<VPS_HOST>
cd /var/www/student-management
git log --oneline -10
git reset --hard <previous-good-sha>
cd backend && npm ci --omit=dev
cd ../frontend && npm ci && npm run build
pm2 reload site-backend2
systemctl restart nginx
```

Then either revert the commit on GitHub (which will re-trigger the workflow and roll forward to the revert) or temporarily disable the workflow (Actions tab → Deploy → ⋯ → Disable) while you stabilise.

## What the workflow does **not** do

These are deliberately out of scope; handle them separately:

| Concern | Where it lives |
|---|---|
| Running tests before deploy | Not yet — TODO: add a `test` job that the `deploy` job `needs:`. See `tests` script in [`backend/package.json`](../../backend/package.json) and `npm test` in [`frontend/package.json`](../../frontend/package.json). |
| DB migrations | None exist in this codebase — Mongoose is schemaless. If you add migrations, run them between `git pull` and `pm2 reload`. |
| `.env` provisioning | Manual on the VPS. Workflow secrets are GitHub-side only. |
| Backups | See [`backups.md`](backups.md). Should be a separate cron, not coupled to deploys. |
| Asset invalidation (CDN) | Not applicable — Nginx serves directly from `frontend/dist/`. |
| Smoke test after deploy | Not yet — TODO: append `curl --fail https://<host>/api/hello` to the script so a broken deploy fails the workflow. |

## Failure modes

| Symptom | Likely cause | Fix |
|---|---|---|
| Workflow fails at `git pull` with merge conflict | Manual edit on the VPS clashes with `main`. | SSH in, `git status`, resolve or hard-reset to `origin/main`. |
| Workflow fails at `npm ci` | `package-lock.json` out of sync with `package.json`, or registry timeout. | Run `npm install` locally, commit the updated lock, push again. |
| Workflow succeeds but the site shows the old build | Nginx is caching aggressively, or the browser is. | `systemctl reload nginx` (already done by the workflow); hard-refresh the browser. If persistent, inspect `frontend/dist/index.html` on the VPS and confirm the new bundle hash. |
| `pm2 reload site-backend2` says **process not found** | PM2 process is down or named differently. | `pm2 list` on the VPS. If down, `pm2 start ecosystem.config.js`; if renamed, update the workflow or rename the PM2 process. |
| `Permission denied (publickey)` on first run | `VPS_SSH_KEY` doesn't match an `authorized_keys` entry for root. | Re-run `ssh-copy-id` with the matching public key; verify with a manual `ssh -i deploy_key root@<VPS_HOST>`. |
| `Permission denied (publickey)` with the key set | Passphrase secret missing or wrong. | Confirm `VPS_SSH_PASSPHRASE` matches the passphrase used when `ssh-keygen` was run. Rotate the key if the passphrase is lost. |
| `systemctl: command not found` | The SSH `PATH` doesn't include `/usr/bin`. | Rare with root; if it happens, use the absolute path `/bin/systemctl` in the workflow. |

## See also

- [`deployment.md`](deployment.md) — one-time VPS setup that this workflow assumes is in place.
- [`environment-variables.md`](environment-variables.md) — what must be present in `.env` on the VPS.
- [`troubleshooting.md`](troubleshooting.md) — general runtime issues.
