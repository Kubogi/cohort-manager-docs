# Backups & Restore

This system's stateful data is in three places:

1. **Primary MongoDB cluster** (`MONGO_URI`) — students, decisions, grades, health records, users, units, system settings, surveys.
2. **Secondary MongoDB cluster** (`MONGO_URI2`) — partner schools (`schools`) and certificate-lookup index (`students`).
3. **The host filesystem** at `backend/uploads/` — học liệu drives + attachments. Excel imports also stage temporary files here.

A backup that doesn't cover all three is incomplete: Mongo-only restores will leave attachment downloads as 404s; uploads-only restores will leave Mongo references pointing at the right files but no metadata.

There is **no in-repo backup automation**. This guide is the runbook the operator follows; wire whatever scheduler you prefer (cron, systemd timer, your VPS provider's snapshot service).

## 1. What to back up

| Data | Source | Frequency | Retention (suggested) |
|---|---|---|---|
| Primary Mongo | `MONGO_URI` | Nightly | 14 daily + 8 weekly |
| Secondary Mongo | `MONGO_URI2` | Weekly (read-mostly) | 4 weekly |
| `backend/uploads/` | API host filesystem | Daily | 14 daily |
| `.env` | VPS only | On every change | Off-host (password manager) |

Don't back up `node_modules`, `frontend/dist/`, `backend/uploads/.tmp/` (multer staging, auto-cleaned), or anything you keep outside the application. None contain unrecoverable state.

## 2. Daily Mongo backup

```bash
#!/usr/bin/env bash
set -euo pipefail

DATE=$(date +%F)
BACKUP_ROOT=/var/backups/student-mgmt
mkdir -p "$BACKUP_ROOT/$DATE"

# Primary cluster (operational data)
mongodump \
  --uri="$MONGO_URI" \
  --gzip \
  --out="$BACKUP_ROOT/$DATE/primary"

# Secondary cluster (partner schools + certificates) — weekly is enough,
# but running daily costs nothing and keeps the layout uniform.
mongodump \
  --uri="$MONGO_URI2" \
  --gzip \
  --out="$BACKUP_ROOT/$DATE/secondary"
```

Save as `/usr/local/sbin/backup-mongo.sh`, `chmod +x`, then cron:

```cron
# /etc/cron.d/student-mgmt-backup
0 2 * * *  root  /usr/local/sbin/backup-mongo.sh >> /var/log/student-mgmt-backup.log 2>&1
```

The `--gzip` flag halves on-disk size and means restores need `--gzip` too.

## 3. Daily uploads backup

```bash
#!/usr/bin/env bash
set -euo pipefail

DATE=$(date +%F)
UPLOADS=/home/deploy/apps/student-management/backend/uploads
BACKUP_ROOT=/var/backups/student-mgmt-uploads

# Hardlink-based daily snapshots (cheap for unchanged files).
rsync -a --delete \
  --link-dest="$BACKUP_ROOT/latest" \
  --exclude='.tmp/' \
  "$UPLOADS"/ \
  "$BACKUP_ROOT/$DATE/"

# Swap the "latest" symlink atomically.
ln -sfn "$BACKUP_ROOT/$DATE" "$BACKUP_ROOT/latest"
```

The `--link-dest` flag makes unchanged files hardlinks against yesterday's snapshot, so a 5 GB uploads tree with one file added per day takes ~5 GB total, not 5 GB × days.

Exclude `.tmp/` — those are multer staging files; backing them up wastes space and could capture half-uploaded data.

## 4. Off-host retention

Keep at least one backup off the VPS. Otherwise a host disk loss takes the backups with it. Options:

- **Cloud object storage** — push the latest snapshot to S3/Wasabi/Backblaze B2 via `rclone` after each backup run.
- **A second VPS** — rsync from the backup host to a different region.
- **Provider snapshot** — most VPS providers offer per-image snapshots. These are convenient but not a substitute for application-level backups; restore granularity is "whole VM, point-in-time".

Encrypt off-host backups (`age`, `gpg`, S3 server-side encryption with a customer key). The dump includes student PII.

## 5. Restore — primary Mongo

```bash
# Restore into a fresh database. Replace target_db with the actual name.
mongorestore \
  --uri="mongodb://restoreUser:<pwd>@127.0.0.1:27017/admin" \
  --gzip \
  --nsFrom='[DB_NAME].*' --nsTo='target_db.*' \
  /var/backups/student-mgmt/2026-05-15/primary
```

Use `--nsFrom`/`--nsTo` to restore into a parallel DB if you want to inspect before swapping. To restore over the live DB, drop or rename it first:

```bash
mongosh "$MONGO_URI" --eval "db.dropDatabase()"
```

Then run `mongorestore` against the live URI.

## 6. Restore — uploads

```bash
DATE=2026-05-15
sudo systemctl stop pm2-deploy   # or `pm2 stop student-mgmt-api`
rsync -a --delete \
  /var/backups/student-mgmt-uploads/$DATE/ \
  /home/deploy/apps/student-management/backend/uploads/
sudo systemctl start pm2-deploy
```

Stopping the API process during the restore avoids races (multer writing the same file you're restoring). For a strict restore (file metadata exactly matches Mongo), restore Mongo first, then uploads.

## 7. Verify the restore

After any restore:

```bash
# Mongo: confirm collection counts roughly match what you expect
mongosh "$MONGO_URI" --eval "db.sinh_vien.countDocuments()"
mongosh "$MONGO_URI2" --eval "db.schools.countDocuments()"

# Uploads: spot-check that an attachment URL still resolves
curl -I -H "Authorization: Bearer $ADMIN_TOKEN" \
  https://your-domain/api/attachments/<id>/download
```

The integration test for "file referenced by Mongo exists on disk" is the user opening a record with an attachment. Build a small spot-check page or a dedicated `check-attachments.js` script if you do many restores.

## 8. Drill it once per quarter

A backup nobody has tried restoring is not a backup. Schedule a quarterly drill:

1. Spin up a throwaway VPS / dev environment.
2. Point `MONGO_URI` / `MONGO_URI2` at scratch databases.
3. Run the restore steps from a recent off-host backup.
4. Bring up the app, log in as admin, open a few pages.

Most failures are configuration drift (`.env` paths, Mongo auth changes, file permissions) — catching them on a drill, not on a real outage, is the entire point.
