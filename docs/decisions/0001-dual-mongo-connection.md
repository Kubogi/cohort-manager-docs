# ADR 0001 — Dual MongoDB Connection

**Status**: Accepted (in production since project inception)
**Date**: ~2024 (predates this docs tree; recorded retroactively 2026-05-16)

## Context

This system needs two distinct sets of data:

1. **Operational data** — students, decisions, grades, health records, units, users. Mutated continuously by the app.
2. **Reference data** — partner-school catalog (`schools`) and historical certificate index (`students`). Owned by a separate, smaller project (the [Certificate Management](https://github.com/Kubogi/certificate-management-docs) system) that this app reads but does not write.

The two have different lifecycles: operational data evolves daily with the institution's operations; reference data is updated only when the lookup project ingests a new batch of certificates.

## Decision

Use **two separate MongoDB clusters**, each with its own Mongoose connection:

- **Primary** — `MONGO_URI`, accessed via `mongoose.connect()` and the default global connection. Owns 13 collections.
- **Secondary** — `MONGO_URI2`, accessed via `mongoose.createConnection()` in [`backend/src/db/secondaryConnection.js`](../../backend/src/db/secondaryConnection.js). Owns 2 collections: `schools` (partner organizations) and `students` (certificate lookup). Models bind explicitly to this connection via `secondaryConnection.model(...)`.

Both clusters can run in the same physical Atlas project today; the separation is at the connection/database level, not necessarily the cluster level.

## Consequences

### Positive

- Each project owns its own schema; the Certificate Management project can change its `students` table without coordinating a deploy here, as long as the read-shape stays compatible.
- Operational backups are independent. Restoring the primary cluster doesn't require touching the lookup cluster.
- The lookup cluster is read-mostly here; access patterns are simpler.

### Negative

- **You can't `$lookup` or aggregate across clusters.** Joining data from both means: fetch from one, then query the other in JavaScript with the resulting IDs. Service-layer code does this in a few places.
- **The secondary connection is fail-fast at import time.** [`secondaryConnection.js`](../../backend/src/db/secondaryConnection.js) throws synchronously if `MONGO_URI2` is missing — the process crashes during module loading, before `connectPrimary()` even runs. This means the env var is *effectively* required, despite the `env.js` default being an empty string. There's no graceful degradation if you only want to use part of the app.
- **Backup and restore is two procedures, not one.** See [`/docs/guides/backups.md`](../guides/backups.md).
- **Field-naming inconsistency** — the secondary cluster's fields are snake_case Vietnamese on disk (`ten_truong`, `ho_va_ten`) because they're owned by a different project. Mongoose `alias` masks this for `DonViLienKet` but not `CertificateLookup`. Code that touches the secondary cluster has to deal with both styles.

## Alternatives considered

- **Single cluster with two databases** — same physical Mongo, two databases. Rejected because it doesn't decouple ownership; the lookup project still needs to coordinate schema changes.
- **Single cluster, single database, namespaced collections** — `lookup_schools`, `lookup_students`. Rejected for the same reason plus the convention noise.
- **HTTP API instead of a shared Mongo** — the lookup project would expose REST endpoints. Rejected for cost (extra hop, latency on every certificate lookup) and complexity (no Mongoose models here, hand-rolled HTTP client).

## See also

- [`/docs/architecture/database.md`](../architecture/database.md) — schema-level detail
- [`backend/src/db/secondaryConnection.js`](../../backend/src/db/secondaryConnection.js) — the connection initializer
- [`backend/src/models/DonViLienKet.js`](../../backend/src/models/DonViLienKet.js), [`backend/src/models/CertificateLookup.js`](../../backend/src/models/CertificateLookup.js) — the two models bound to the secondary
