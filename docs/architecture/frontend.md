# Frontend Architecture

The frontend is a React 19 + TypeScript single-page app, built with Vite, structured around **react-admin 5.12** as the resource/CRUD shell. The UI is Vietnamese; user-facing labels are in `App.tsx` and inside each resource file.

This document covers the technical scaffolding: the bootstrap order, the data-flow layers (dataProvider + authProvider + HTTP client), the resource-name → API-path mapping, the design-token system, and the hooks that every page leans on. For per-page user-facing documentation see [`docs/frontend/README.md`](../frontend/README.md).

## 1. Bootstrap

```
frontend/src/
├─ main.tsx                  ── ReactDOM.createRoot → <App />
├─ App.tsx                   ── <Admin> shell, custom menu, theme, resource list
├─ LoginPage.tsx             ── Email/password form posted via authProvider.login
├─ dataProvider.tsx          ── React-admin DataProvider (CRUD over fetch)
├─ authProvider.ts           ── React-admin AuthProvider (JWT in localStorage)
├─ api/
│  ├─ client.ts              ── fetch wrapper, Bearer injection, 401 auto-logout
│  └─ unitLookupService.ts   ── thin helpers for /api/don-vi/* (used by useUnitLookups)
├─ hooks/                    ── useUnitLookups, useTeacherScope
├─ components/               ── shared/ (Button, Modal, Input, Card, DataTable, …) + SearchableSelect, AttachmentField, …
├─ resources/                ── one folder per menu module, then per page (see frontend/README.md)
├─ styles/                   ── design tokens + per-component style objects
└─ data/types.ts             ── shared TS types (entity shapes, enums)
```

In production, Vite builds a static bundle into `frontend/dist/`, which Nginx serves at `/`. There is no SSR. All routing is client-side (react-router via react-admin).

## 2. The `<Admin>` shell (`src/App.tsx`)

`App.tsx` does five things:

1. Imports every resource component (one `import` per page).
2. Defines a `MyAppBar` (top bar with Vietnamese title) and a custom `MyMenu` (collapsible Vietnamese sidebar) — these replace react-admin's defaults so the menu can show Vietnamese groupings.
3. Builds a `darkBlueTheme` extending react-admin's `defaultTheme` with token-derived typography and a dark-blue color scheme.
4. Renders `<Admin dataProvider authProvider loginPage layout theme>` and uses the `(permissions) => …` render-prop form to filter the resource list at render time.
5. Provides a `MyLayout = (props) => <Layout {...props} appBar={MyAppBar} menu={MyMenu} />` wrapper.

The resource list inside `<Admin>` registers ~22 `<Resource name=… list=… />` entries — one per page. The names match the keys used in `dataProvider.resourcePathMap` and the route segments under `/#/`.

**Teacher-role filtering.** If `permissions === 'teacher'`, the resource array is filtered to a small allow-list:

```
'lichHuanLuyen', 'soTayGiangVien', 'khoHocLieuSo',
'nhapDiem', 'traCuu', 'thongKeBaoCao',
'giangVienTuDanhGia', 'khaoSatSinhVien', 'ketQuaKhaoSat',
'caiDatTaiKhoan'
```

The menu also hides "Thông tin chung" and "Quản lý sinh viên" sections for teachers, and conditionally renders admin-only items (`quanLyLink`).

## 3. dataProvider — Vietnamese resource → kebab-case path

`src/dataProvider.tsx` implements the react-admin `DataProvider` interface (getList, getOne, getMany, getManyReference, create, update, updateMany, delete, deleteMany) on top of `api/client.ts request()`.

The key piece is `resourcePathMap`, which translates the camelCase Vietnamese resource names used in the SPA into the kebab-case API segments used by the backend:

```ts
const resourcePathMap: Record<string, string> = {
  khoaHoc:              'khoa-hoc',
  giangVien:            'can-bo-quan-ly',   // note: NOT 'giang-vien'
  coSoDuLieuSinhVien:   'sinh-vien',
  quanLyCacQuyetDinh:   'quyet-dinh',
  hoSoSucKhoe:          'ho-so-suc-khoe',
  nhapDiem:              'diem',
  trucQuanSu:           'truc-quan-su',
};

const resolvePath = (resource: string) =>
  resourcePathMap[resource] ?? toKebabCase(resource);
```

Any resource not in the map falls through to a naive camelCase → kebab-case conversion. This means **adding a `<Resource name="foo" />` whose API path is `/api/foo` Just Works** (no map entry needed) — but a name like `myBespoke` would resolve to `/api/my-bespoke`, which may or may not exist on the backend. If you add a resource whose backend path is not a kebab-case form of its name, add it to `resourcePathMap`.

### Id normalization

The backend returns Mongo's `_id`; react-admin requires `id`. The dataProvider normalizes both directions:

```ts
const normalizeItem = (item) => {
  const id = (item as any)?.id ?? (item as any)?._id;
  return { ...item, id };
};
```

`updateMany` and `deleteMany` are implemented as parallel per-id requests (no bulk endpoint exists on the backend).

## 4. authProvider — JWT in localStorage

`src/authProvider.ts` implements the react-admin `AuthProvider` interface and stores three localStorage keys:

| Key | Contents |
|---|---|
| `accessToken` | Short-lived JWT (1 h, signed with `JWT_SECRET`). Sent as `Authorization: Bearer …` on every request. |
| `refreshToken` | Long-lived JWT (7 d, signed with `JWT_REFRESH_SECRET`). **Not currently used by any UI flow** — the frontend logs out on 401 instead of calling `/api/auth/refresh`. |
| `authUser` | Cached `{ id, username, role }` from the login response. Falls back to decoding the access token if missing. |

### Methods

| Method | Behavior |
|---|---|
| `login({username,password})` | `fetch` POSTs to `${API_BASE_URL}auth/login`, stores tokens + user. Vietnamese error messages. |
| `logout()` | Clears the three localStorage keys. |
| `checkAuth()` | Resolves if `accessToken` is present, rejects otherwise. |
| `checkError(error)` | On 401/403, clears tokens and rejects (forces redirect to login). |
| `getPermissions()` | Returns `user.role` from localStorage or decodes from the access token. Used by `<Admin>` render prop and `usePermissions()`. |
| `getIdentity()` | Returns `{ id, fullName }` for the top-bar identity widget. |

Helpers `readRoleFromToken()` and `readTeacherScopeFromToken()` decode the JWT payload client-side (atob + JSON.parse). These power `useTeacherScope()` so the UI can pre-filter dropdowns without a round-trip.

> **Security note.** Storing tokens in `localStorage` makes them readable by any XSS. The codebase relies on React's default escaping for safety. Do not add `dangerouslySetInnerHTML` or eval'd-from-API content without considering this.

## 5. HTTP client — `src/api/client.ts`

A single `request<T>(path, options)` helper wraps `fetch`:

- **Base URL** — `import.meta.env.VITE_API_URL`, defaulting to `http://localhost:3000/api`. Missing protocol is auto-prefixed with `http://` so `.env` lines like `VITE_API_URL=localhost:3000/api` work.
- **Bearer injection** — pulls `accessToken` from localStorage on every call.
- **Body handling** — JSON-encodes objects, passes `FormData` through unchanged (lets the browser set `Content-Type: multipart/form-data; boundary=...`).
- **Query string** — flat key/value, `undefined`/`null`/empty values dropped, arrays joined with `,`.
- **204 handling** — returns `{ data: undefined }` so callers don't try to parse an empty body.
- **Global 401** — clears tokens, redirects to `/login`, throws a Vietnamese-language error. Important: this happens *before* the caller sees the error, so calling code can assume any caught error is non-auth.

The same file exports three typed helper bundles:

| Bundle | Endpoints it wraps |
|---|---|
| `userAPI` | `GET /users`, `GET /users/:id`, `POST /auth/register`, `PATCH /users/:id`, `DELETE /users/:id` |
| `settingsAPI` | `GET/PATCH /settings/training-schedule-link` |
| `khaoSatChatLuongAPI` | `/khao-sat-chat-luong/nam`, `/khao-sat-chat-luong/giang-vien` CRUD |
| `changePassword(old,new)` | `PATCH /auth/change-password` |

Other resources go through the react-admin `dataProvider`, not these helpers. The two coexist because some pages need queries the generic dataProvider can't express (e.g. POST to a custom path, complex form data).

## 6. Shared components

Production-ready components live in `src/components/shared/` and `src/components/`. See [`frontend/STYLING_GUIDE.md`](../../frontend/STYLING_GUIDE.md) for full API + usage; quick summary:

| Component | Purpose |
|---|---|
| `Button` (shared/) | `variant: 'primary'\|'secondary'\|'danger'`, `size: 'small'\|'medium'\|'large'` |
| `Modal` + `ModalFooter` (shared/) | Dialog with overlay click + escape-to-close. `ModalFooter` provides pre-styled Confirm/Cancel pair. |
| `Input` (shared/) | Text/textarea with label, error, helper text |
| `DataTable` (shared/) | Reusable tabular wrapper |
| `DateInput` (shared/) | Date picker field |
| `Card` (shared/) | Container with token-styled padding/shadow |
| `SearchableSelect` | Dropdown with Vietnamese-locale-aware sorting + keyboard nav. Used heavily for unit pickers. |
| `AttachmentField` | File-upload widget for QuyetDinh/HoSoSucKhoe modals; talks to `/api/attachments` |

**Rule:** when adding UI, search `src/components/` and `src/styles/` *before* writing a new style block. Reusing the existing components keeps the dark-blue theme consistent and the token system useful.

## 7. Hooks

| Hook | Purpose |
|---|---|
| `useUnitLookups()` (`hooks/useUnitLookups.ts`) | Loads `khoa`, `daiDoi`, `donViLienKet` from the API once per page; returns `{ khoaMap, daiDoiMap, truongMap, khoaOptions, truongOptions, daiDoiList, refetchDaiDoi(khoaId?), ensureCoreLookupsLoaded(), loading, error }`. Use this anywhere a unit dropdown is needed; do not re-query these endpoints manually. |
| `useTeacherScope()` (`hooks/useTeacherScope.ts`) | Reads `role` + `teacherScope[]` from the JWT and returns `{ isTeacher, scope, allowedKhoa, allowedDaiDoi(khoaId), allowedTruong(), singletonKhoa, singletonDaiDoi(khoaId), singletonTruong() }`. Use the singleton helpers to auto-fill a filter when a teacher has access to exactly one unit on an axis. |

## 8. Design tokens

`src/styles/tokens/` defines all visual primitives. There is **no CSS framework** — components compose token values via inline styles (or, in some cases, MUI `sx`).

```
styles/tokens/
├─ colors.ts         ── colors.primary[50…900], status, semantic
├─ typography.ts     ── fontFamily, fontSize.{xs,sm,base,lg,xl,2xl,3xl,4xl,5xl}, fontWeight, lineHeight
├─ spacing.ts        ── 8-point scale (spacing[0]…spacing[20])
├─ borderRadius.ts
├─ shadows.ts
└─ version.ts        ── changelog for token-system changes
```

Component-level style objects live in `src/styles/components/` (e.g. `button.styles.ts`, `modal.styles.ts`). They reference tokens, never raw values. **Do not hard-code colors, spacing, or font sizes in resource files** — import from tokens. [`frontend/STYLING_GUIDE.md`](../../frontend/STYLING_GUIDE.md) is the long-form reference.

## 9. Routing

react-admin uses hash-based routing by default. The URL `https://host/#/coSoDuLieuSinhVien` renders the `coSoDuLieuSinhVien` resource's `list` component. There are no manually defined routes — every page is reached via `<Resource name=… list=… />`. To add a new page, add a `<Resource>` entry in `App.tsx` and (optionally) a `<Menu.Item to="/<name>" …/>` in `MyMenu`.

Two outliers worth knowing:
- **Login** — `loginPage={LoginPage}` is rendered by `<Admin>` when `checkAuth` rejects.
- **External link** — the "CSDL Chứng chỉ" menu item in `App.tsx` opens the [Certificate Management](https://github.com/Kubogi/certificate-management-docs) system in a new tab via `window.open`, bypassing react-admin's routing entirely.

## 10. Build, dev, env

```bash
cd frontend
npm install
npm run dev          # Vite dev server, default port 5173
npm run build        # tsc -b && vite build → dist/
npm run preview      # serve dist/ locally
npm run lint         # eslint .
npm run test         # vitest (no unit tests exist yet — only setup)
npm run cypress:open # Cypress runner UI (E2E)
npm run cypress:run  # Cypress headless
```

**Env vars read by the frontend** (`vite.config.ts` sets `envDir: '..'` so `.env` is read from the project root):

| Variable | Default | Where it's read |
|---|---|---|
| `VITE_API_URL` | `http://localhost:3000/api` | `src/api/client.ts:8` |

Anything not prefixed `VITE_` is **not** exposed to the bundle.

## 11. Adding a new page — checklist

1. **Create the entry file** under `src/resources/<menuModule>/<pageName>.tsx`. Export a PascalCase component (e.g. `export const FooBar = () => …`).
2. **Companion folder** (if the page is non-trivial) — sibling folder `src/resources/<menuModule>/<pageName>/` with the `useFooBarData.ts` hook, sub-views, modals, etc.
3. **Register the resource** in `App.tsx`: add the import and a `<Resource key="fooBar" name="fooBar" list={FooBar} />` entry.
4. **Add to the menu** in `App.tsx`'s `MyMenu` under the correct Vietnamese section.
5. **Map the resource path** in `dataProvider.tsx` if the backend path is not the auto-derived kebab-case of the name.
6. **Reuse components** — `Button`, `Modal`, `Input`, `SearchableSelect`, etc. Do not write a new modal from scratch.
7. **Reuse hooks** — `useUnitLookups()` for any unit picker, `useTeacherScope()` for any per-unit filter that should auto-apply for teachers.
8. **Tokens for styling** — pull from `@/styles/tokens`; never inline hex colors or pixel values.
9. **Type-check** — `npx tsc --noEmit` in `frontend/` must pass before commit.
10. **Document** — add a `docs/frontend/pages/<menu>/<page>.md` per the template in the doc plan.

## 12. See also

- [`overview.md`](overview.md) — top-of-stack diagram + request flow
- [`auth.md`](auth.md) — JWT issuance, role matrix, scope evaluation
- [`docs/frontend/README.md`](../frontend/README.md) — per-page user reference
- [`frontend/STYLING_GUIDE.md`](../../frontend/STYLING_GUIDE.md) — design tokens + component API
