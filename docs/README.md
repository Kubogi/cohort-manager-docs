# Documentation

Welcome. This is the index for everything you need to understand, run, deploy, or extend the student-management system.

Vietnamese domain terms appear throughout — see [glossary.md](glossary.md) for translations.

## I want to…

### …get the app running locally
Start at [guides/getting-started.md](guides/getting-started.md). You'll need Node 22 and a MongoDB connection string. Total setup time on a clean machine is ~15 minutes.

### …deploy to production
[guides/deployment.md](guides/deployment.md) walks through a fresh Ubuntu 22.04 VPS: Nginx, PM2, MongoDB, Certbot. Pairs with `nginx.conf.example`, `ecosystem.config.js.example`, and `mongod.conf.example` at the repo root.

### …understand the system
- [architecture/overview.md](architecture/overview.md) — the 10-minute system tour
- [architecture/backend.md](architecture/backend.md) — Express layers, routes, middleware
- [architecture/frontend.md](architecture/frontend.md) — react-admin shell, dataProvider, authProvider
- [architecture/database.md](architecture/database.md) — Mongo primary + secondary, ERD, indexes
- [architecture/auth.md](architecture/auth.md) — JWT flow, roles, `allowedUnits`, `teacherScope`
- [architecture/file-storage.md](architecture/file-storage.md) — `uploads/` layout, drive quotas
- [architecture/excel-pipeline.md](architecture/excel-pipeline.md) — Excel import (sinh-vien) and export (bieu-mau)

### …add or change an API endpoint
Read [api/README.md](api/README.md) for the endpoint catalog and [backend/schemas/README.md](backend/schemas/README.md) for the data model. Each individual endpoint and schema has its own file.

### …understand or use a specific page
[frontend/README.md](frontend/README.md) is the page-by-page index, organized to mirror the menu in the app. Each page doc covers: when to use it, the layout, every tab, common tasks, and gotchas.

### …understand a business workflow
[workflows/README.md](workflows/README.md) covers the major end-to-end processes (login, student lifecycle, decision processing, grade entry, Excel import, etc.) across UI + API + DB.

### …configure environment variables
[guides/environment-variables.md](guides/environment-variables.md) lists every var the backend and frontend read, with defaults and the file:line where each is consumed.

### …run or write tests
[guides/testing.md](guides/testing.md) covers strategy. For commands, see [`backend/docs/TESTING.md`](../backend/docs/TESTING.md) and [`frontend/docs/TESTING.md`](../frontend/docs/TESTING.md).

### …back up or restore production data
[guides/backups.md](guides/backups.md) — `mongodump` / `mongorestore` for both Mongo connections plus the `backend/uploads/` tree.

### …decode design choices
[decisions/](decisions/) holds ADRs for non-obvious calls (dual-Mongo, Vietnamese resource names, embedded `diem`, etc.).

## Folder map

```
docs/
├── README.md              ← you are here
├── glossary.md            EN ↔ VI domain terms
├── architecture/          How the system is wired
├── api/                   Endpoint reference (one file per endpoint)
├── backend/schemas/       Mongoose model reference (one file per schema)
├── frontend/              Page-by-page user reference
├── workflows/             End-to-end process docs
├── guides/                Setup, deployment, testing, backups, troubleshooting
└── decisions/             ADRs
```

## Related, non-`docs/` files

- [`frontend/STYLING_GUIDE.md`](../frontend/STYLING_GUIDE.md) — design-token and component-style reference. The frontend architecture doc links into it instead of duplicating.
