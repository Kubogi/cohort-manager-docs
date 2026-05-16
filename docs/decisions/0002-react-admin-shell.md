# ADR 0002 — Use react-admin as the SPA shell

**Status**: Accepted (in production since project inception)
**Date**: ~2024 (recorded retroactively 2026-05-16)

## Context

The app is, in essence, a collection of admin CRUD pages over a REST API — list students, edit a decision, browse a folder of materials. Building each of these from scratch in plain React would mean re-implementing the same patterns: paginated lists with filters, edit modals with validation, role-based menu gating, and a top-bar + sidebar layout.

[react-admin](https://marmelab.com/react-admin/) is a framework explicitly designed for this kind of admin interface, with batteries-included patterns for resources, data providers, auth providers, and a Material-UI-based shell.

## Decision

Build the frontend on top of **react-admin 5.12** as the admin shell, with these specific choices:

- Use `<Admin>` from react-admin as the root component, with a **custom `dataProvider`** (mapping Vietnamese camelCase resource names to kebab-case API paths) and a **custom `authProvider`** (JWT in localStorage).
- Use react-admin's `<Resource>` model: each menu item is a `<Resource name=... list=... />`. The `list` component renders the whole page (we don't use `<List>` / `<Datagrid>` because our UIs are too custom).
- **Replace** react-admin's default sidebar and top bar with custom Vietnamese implementations in `App.tsx` (`MyMenu`, `MyAppBar`).
- Keep react-admin's `usePermissions`, `useDataProvider`, `useNotify` etc. for the convenience hooks they provide.

## Consequences

### Positive

- The `<Admin>` shell handles auth lifecycle (`checkAuth` on every navigation, `getPermissions`, automatic redirect to login on 401), saving a lot of plumbing.
- `useDataProvider` + `dataProvider.getList(...)` produces consistent pagination + sort + filter handling across pages.
- react-admin's hooks integrate with React Query under the hood, so caching and re-fetch invalidation come for free.
- Role-based rendering via `<Admin>(permissions => …)` is clean: the resource list can be filtered server-trustedly by role.

### Negative

- **Heavy dependency.** `react-admin@5.12` pulls in Material-UI v5, React Query, react-router, react-hook-form. The frontend bundle is large; the production `dist/` is several MB compressed. Acceptable for an admin tool used by ~10 people; would be a non-starter for a public consumer app.
- **Custom UI fights the defaults.** Because every page is a custom `list` component rather than react-admin's `<List>` + `<Datagrid>`, we duplicate effort that the framework would have done for us. The trade-off: the UIs are exactly what the Vietnamese operators expect, with vocab/labels they recognize.
- **Custom menu means we re-implement role gating twice** — once in `App.tsx`'s `MyMenu` (which items to show) and once in the resource render-prop (`teacherOnly` allow-list). They have to be kept in sync.
- **Inline styles via tokens instead of `theme.palette`.** We do use react-admin's `theme` for typography and the Material drawer, but most component styling is via `@/styles/tokens` rather than the MUI theme system. The frontend styling guide (`frontend/STYLING_GUIDE.md`) is the source of truth, not MUI.

## Alternatives considered

- **Plain React + react-router + an ad-hoc data layer.** Rejected — the admin-CRUD patterns are exactly what react-admin solves; building them from scratch would have shipped slower and inconsistently.
- **AdminJS** — Rails-style auto-generated admin UI from the Mongoose schema. Rejected because the operator workflows are too domain-specific (decision processing, Excel pipeline, automatic battalion assignment) for a generated UI.
- **Refine** — a more modular alternative to react-admin. Considered briefly but react-admin had more out-of-the-box hooks and a larger Stack Overflow footprint at the time.

## See also

- [`/docs/architecture/frontend.md`](../architecture/frontend.md) — how `<Admin>`, dataProvider, and authProvider fit together
- [`/frontend/src/App.tsx`](../../frontend/src/App.tsx) — the shell composition
- [`/frontend/STYLING_GUIDE.md`](../../frontend/STYLING_GUIDE.md) — design tokens, why we don't use the MUI theme system end-to-end
