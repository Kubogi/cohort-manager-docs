# Deployment

Target: a single self-hosted Ubuntu 22.04 LTS VPS running:

- **Nginx** as the public-facing reverse proxy and static asset server
- **PM2** as the process manager for the Node backend
- **MongoDB** self-hosted on the same VPS (or a separate cluster — see [Variants](#variants))
- **Certbot** for HTTPS

This guide assumes you have root or sudo access on a fresh Ubuntu host with a public IP and a DNS A record pointing to it.

For local setup, see [`getting-started.md`](getting-started.md).
For environment variables, see [`environment-variables.md`](environment-variables.md).

## 1. Install system packages

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl ufw git

# Node 22 LTS (NodeSource binary distribution)
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt install -y nodejs
node --version    # → v22.x

# PM2 (global)
sudo npm install -g pm2

# MongoDB 7 — see https://www.mongodb.com/docs/manual/installation/
# Skip if you're using MongoDB Atlas or another managed cluster.
sudo apt install -y gnupg
curl -fsSL https://www.mongodb.org/static/pgp/server-7.0.asc \
  | sudo gpg --dearmor -o /usr/share/keyrings/mongodb-server-7.0.gpg
echo "deb [signed-by=/usr/share/keyrings/mongodb-server-7.0.gpg] https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/7.0 multiverse" \
  | sudo tee /etc/apt/sources.list.d/mongodb-org-7.0.list
sudo apt update
sudo apt install -y mongodb-org
sudo systemctl enable --now mongod

# Nginx
sudo apt install -y nginx
sudo systemctl enable --now nginx

# Certbot
sudo apt install -y certbot python3-certbot-nginx
```

## 2. Configure MongoDB

Edit `/etc/mongod.conf` (see the annotated [`mongod.conf.example`](../../mongod.conf.example) at the repo root for the template). Key changes from the Ubuntu defaults:

```yaml
net:
  bindIp: 127.0.0.1          # localhost only; Nginx and PM2 reach Mongo over loopback
  port: 27017

security:
  authorization: enabled     # require username/password — see step 2b

storage:
  dbPath: /var/lib/mongodb

systemLog:
  destination: file
  path: /var/log/mongodb/mongod.log
  logAppend: true
```

Then create application credentials inside `mongosh`:

```bash
sudo systemctl restart mongod
mongosh
```

```js
use admin
db.createUser({
  user: "[DB_ROOT_USER]",
  pwd: "<strong random>",
  roles: [{ role: "root", db: "admin" }]
})

use [DB_NAME]
db.createUser({
  user: "[DB_APP_USER]",
  pwd: "<strong random>",
  roles: [{ role: "readWrite", db: "[DB_NAME]" }]
})
```

The `MONGO_URI` for `.env` becomes `mongodb://[DB_APP_USER]:<strong-password>@127.0.0.1:27017/[DB_NAME]?authSource=[DB_NAME]`.

`MONGO_URI2` follows the same pattern with a separate database name (e.g. `[DB_NAME_2]`) — or points at the actual [Certificate Management](https://github.com/Kubogi/certificate-management-docs) cluster if you run that separately.

> ⚠ Don't use the same credentials as another environment. Don't put production passwords in a non-production `.env`.

## 3. Create the deploy user

```bash
sudo adduser --disabled-password --gecos "" deploy
sudo usermod -aG sudo deploy
```

Switch to `deploy` for the rest of the steps:

```bash
sudo -iu deploy
```

## 4. Clone and install

```bash
mkdir -p ~/apps
cd ~/apps
git clone <your-repo-url> student-management
cd student-management

# Root .env (shared by backend + frontend)
cp example.env .env
nano .env    # fill MONGO_URI, MONGO_URI2, JWT_SECRET, etc.
chmod 600 .env

# Backend deps
cd backend
npm ci --omit=dev      # production-only deps; faster, deterministic
cd ..

# Frontend deps + build
cd frontend
npm ci
npm run build          # → frontend/dist/
cd ..
```

`VITE_API_URL` in `.env` must be set to the public URL the browser will use, e.g. `https://your-domain/api`. Vite bakes this into the bundle at build time — changing `.env` after the build does **not** update the dist files; rebuild.

## 5. Bootstrap the admin account

```bash
cd backend
npm run create:admin -- --username admin --password "<strong random>"
cd ..
```

## 6. Configure PM2

The repo ships an example PM2 config at [`ecosystem.config.js.example`](../../ecosystem.config.js.example). Copy it and adjust paths/usernames:

```bash
cp ecosystem.config.js.example ecosystem.config.js
```

Start, persist across reboots, and inspect:

```bash
pm2 start ecosystem.config.js
pm2 save
pm2 startup systemd       # follow the printed sudo command
pm2 status
pm2 logs student-mgmt-api --lines 100
```

PM2 will log to `~/.pm2/logs/student-mgmt-api-out.log` and `…-error.log` by default. Set up log rotation:

```bash
pm2 install pm2-logrotate
pm2 set pm2-logrotate:max_size 50M
pm2 set pm2-logrotate:retain 14
```

> **Reload vs restart.** `pm2 reload student-mgmt-api` drains existing requests before swapping; `pm2 restart` is hard-stop. Always prefer reload for production. With `instances: 1` (the default in our config) reload still has a brief interruption — to truly avoid it, raise `instances` and add a SIGINT handler in `server.js`. The current `server.js` has no graceful-shutdown handler, so reload is "best effort, fast".

## 7. Configure Nginx

Copy the example server block from [`nginx.conf.example`](../../nginx.conf.example) into `/etc/nginx/sites-available/student-management`:

```bash
sudo cp /home/deploy/apps/student-management/nginx.conf.example \
        /etc/nginx/sites-available/student-management
sudo nano /etc/nginx/sites-available/student-management   # edit server_name, root, etc.
sudo ln -s /etc/nginx/sites-available/student-management /etc/nginx/sites-enabled/
sudo nginx -t                  # validate config
sudo systemctl reload nginx
```

The key pieces of that file:

- `root /home/deploy/apps/student-management/frontend/dist;` — Nginx serves the SPA build directly.
- `try_files $uri $uri/ /index.html;` — for client-side routing fallback (react-router hash routing actually does not need this, but it's safe and supports future history-mode routing).
- `location /api/ { proxy_pass http://127.0.0.1:3000; … }` — backend API.
- `client_max_body_size 110M;` — required for học-liệu uploads. The default 1 MB rejects uploads that the backend would otherwise accept.
- HTTP→HTTPS redirect handled by Certbot (step 8).

## 8. TLS with Certbot

```bash
sudo certbot --nginx -d your-domain
```

Certbot edits the Nginx config in place, restarts Nginx, and installs a renewal cron. To verify renewal:

```bash
sudo certbot renew --dry-run
```

A successful dry run means the cron will keep the cert fresh. Nginx is reloaded automatically by Certbot's renewal hook.

## 9. Firewall

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow OpenSSH
sudo ufw allow 'Nginx Full'
sudo ufw enable
sudo ufw status
```

Port 3000 (Node) and 27017 (Mongo) must **not** be exposed publicly — they sit behind Nginx and on localhost, respectively.

## 10. Sanity-check the deployment

```bash
curl https://your-domain/api/hello
# → "Ciallo ~ (∠・ω< )⌒★"
```

Then open `https://your-domain/` in a browser, log in as the admin you created in step 5.

## 11. Routine operations

### Restart the backend

```bash
pm2 reload student-mgmt-api
```

### Tail logs

```bash
pm2 logs student-mgmt-api --lines 100
sudo journalctl -u nginx -f
sudo tail -f /var/log/mongodb/mongod.log
```

### Deploy a new version

```bash
cd ~/apps/student-management
git pull
cd backend && npm ci --omit=dev && cd ..
cd frontend && npm ci && npm run build && cd ..
pm2 reload student-mgmt-api
```

(`frontend/dist` is generated at build time, so the Nginx static root automatically picks up the new bundle as soon as `npm run build` writes it.)

### Roll back

```bash
cd ~/apps/student-management
git log --oneline -10            # find the previous good commit
git checkout <sha>
cd backend && npm ci --omit=dev && cd ..
cd frontend && npm ci && npm run build && cd ..
pm2 reload student-mgmt-api
```

For a fully reversible deploy strategy, run two checkouts (`current/` and `previous/`) with a symlink in `nginx.conf` pointing at one of them.

### Backup

See [`backups.md`](backups.md). Short version:

```bash
mongodump --uri="$MONGO_URI"  --out=/backups/$(date +%F)
mongodump --uri="$MONGO_URI2" --out=/backups/$(date +%F)-secondary
rsync -a /home/deploy/apps/student-management/backend/uploads/ /backups/uploads/$(date +%F)/
```

## Variants

### Managed MongoDB (Atlas)

Skip step 2; replace `MONGO_URI` and `MONGO_URI2` with the Atlas connection strings. Add the VPS public IP to the Atlas access list. The other steps are unchanged.

### Reverse proxy without TLS (internal network)

Skip step 8 and `ufw allow 'Nginx Full'` becomes `ufw allow 'Nginx HTTP'`. **Do not** run a public-internet deployment without TLS — JWT tokens sent over plain HTTP are trivially captured.

### Separate API and static hosts

If the frontend is served from a CDN and the API from a different host, set `VITE_API_URL` to the API host's URL during `npm run build`, and configure `cors` on the backend accordingly. The default `cors({ origin: true })` already accepts any origin; the cost is that an attacker-controlled site can XHR your API if they can also obtain a token.

## Troubleshooting

| Symptom | Fix |
|---|---|
| `pm2 status` shows `errored` | `pm2 logs` — usually missing env vars (`MONGO_URI2` in particular kills the process on import). |
| Nginx 502 on `/api/*` | Backend down. `pm2 status` to confirm, `pm2 logs` to find the error. |
| Nginx 413 on hoc-lieu uploads | `client_max_body_size` is too small. Bump it in the server block and reload Nginx. |
| Browser shows the SPA but every API call 401s after refresh | `VITE_API_URL` doesn't match the actual proxy. Check the Network tab; fix `.env`; **rebuild** the frontend (`npm run build`). |
| `Certbot renew` fails | DNS no longer points at the VPS, or port 80 is blocked. Restore both before the cert expires. |
| Disk usage climbing fast | `du -sh ~/apps/student-management/backend/uploads/*` — the `học liệu` drives may be near the 3 GB cap. The cap is per-drive and per-app; the host filesystem has no automatic limit. |
