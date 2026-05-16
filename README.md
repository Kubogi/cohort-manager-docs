# Student Cohort Management System — Documentation

A full-stack school administration platform managing the complete operational lifecycle of student cohorts at a military training center.

Vietnamese-language UI; English documentation with Vietnamese domain terms inline.

**This repository contains documentation only. The codebase is closed source.**

## What it manages

- Student records (sinh viên), unit assignments (đại đội), and partner schools (đơn vị liên kết)
- Administrative decisions (quyết định) with cascading student-status updates
- Grades (điểm) with auto-computed averages and Excel gradebook import/export
- Health records (hồ sơ sức khỏe) and hospital tracking
- Learning materials (kho học liệu số) with per-drive storage quotas
- Quality surveys (khảo sát chất lượng) for instructors and students
- Role-scoped accounts (admin / staff / viewer / teacher) with per-unit access control

## System Architecture

```text
Staff / Admin ──► React SPA (react-admin)
                          │
                          ▼
                   Express 5 API
                    │         │
                    ▼         ▼
             MongoDB        MongoDB
            (operational)  (partner schools
                            & cert lookup)
```

Two MongoDB connections are maintained: one for all operational data, and a
secondary read connection for partner-school and certificate lookup data.
Every operation is request-driven and synchronous — no background workers,
queues, or caches.

## Tech Stack

### Frontend
- React 19 + TypeScript
- react-admin 5.12
- Vite
- Custom data provider and Vietnamese-language menu

### Backend
- Node.js 22 + Express 5
- Mongoose 9
- JWT authentication (separate access + refresh tokens)
- Multer (uploads), SheetJS (Excel imports), ExcelJS (Excel exports)

### Database
- MongoDB — two connections (see architecture above)

### Deployment
- Self-hosted Ubuntu VPS
- Nginx reverse proxy
- PM2 process management

## Repo layout

*Layout describes the source repository, not this documentation repo.*

```
backend/      Express API, Mongoose models, validators, services, scripts
frontend/     React-admin SPA, custom dataProvider, Vietnamese menu
docs/         Start here — see docs/README.md
```

## Documentation map

Start with the glossary — domain terms are in Vietnamese and appear throughout.

- [glossary.md](docs/glossary.md) — Vietnamese ↔ English term reference. Read this first.
- [Architecture overview](docs/architecture/overview.md) — components, data flow, deployment topology
- **Backend**
  - [Schemas](docs/backend/schemas/README.md) — one file per Mongoose model
- **API**
  - [API README](docs/api/README.md) — conventions, auth header format, error shape, role matrix
- **Frontend**
  - [Frontend README](docs/frontend/README.md) — page-by-page reference, routing, state
- **Guides**
  - [Deployment](docs/guides/deployment.md)
  - [Workflows](docs/workflows/README.md)

## Known gotchas

A few things that will surprise a new reader:

1. **Dual MongoDB connections** — the secondary connection is read-only and scoped
   to partner-school and certificate data. Writes always go to the primary.
2. **react-admin's dataProvider is fully custom** — the frontend does not use
   a standard provider. See [frontend README](docs/frontend/README.md) before
   assuming any default react-admin behavior.

## Related projects

This system reads partner-school and certificate-lookup data from a separate service:

- [Certificate Management](https://github.com/Kubogi/certificate-management-docs) —
  owns the certificate archive and partner-school catalog. Read-only from this app's
  perspective; populated by its own ingestion pipeline.
