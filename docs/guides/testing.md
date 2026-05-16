# Testing Strategy

A short statement of *what* is tested, *what is intentionally not*, and *what tooling lives where*. For command cheat sheets see [`backend/docs/TESTING.md`](../../backend/docs/TESTING.md) and [`frontend/docs/TESTING.md`](../../frontend/docs/TESTING.md).

## The current shape

| Layer | Tool | What's tested | What's not |
|---|---|---|---|
| Backend integration | Jest + supertest + mongodb-memory-server | Every documented endpoint has at least one happy-path test; auth gating; service-layer scope filters. | E2E against a real Mongo cluster. Performance. |
| Backend unit | Jest | A handful of utilities (`utils/permissions.js` is well-covered). | Most services — covered indirectly via integration tests. |
| Frontend E2E | Cypress | Login flow only today (`cypress/e2e/login.cy.ts`). Logout test is skipped pending implementation. | All other pages — Cypress harness exists, no coverage. |
| Frontend unit | Vitest + Testing Library + MSW | Harness configured; **no actual tests committed**. Adding the first unit test is a good starter task. | Everything. |
| Type safety | `tsc --noEmit` | Every TS file in the frontend. | Backend (JS, no types). |
| Lint | ESLint | Frontend. | Backend. |

The bias is heavy toward **backend integration testing**. The reasoning:

1. The dataProvider abstracts most frontend logic to "fetch and render"; the interesting code is in services and middlewares.
2. The integration tests use `mongodb-memory-server`, so a CI run needs no external infrastructure.
3. Manual playbooks fill the gap for UI flows where Cypress coverage is thin.

## Running the tests

```bash
# Backend
cd backend
npm test               # all tests
npm test -- --watch    # watch mode (not run-in-band; risk of flake)
npm test -- sinh-vien  # filter to tests matching the pattern

# Frontend
cd frontend
npm run lint           # ESLint
npx tsc --noEmit       # type check (must be clean before every commit)
npm run test           # Vitest — runs 0 tests today
npm run cypress:open   # Cypress UI runner
npm run cypress:run    # Cypress headless
```

For full command reference see the per-side TESTING.md cheat sheets.

## When to add what

| You're adding… | Write tests of type… |
|---|---|
| A new API endpoint | Integration: happy path + at least one failure mode + auth gating. File: `backend/src/tests/<domain>.test.js`. |
| A new service-layer helper (scope filter, validation) | Unit test in the same `<domain>.test.js` file or a dedicated `<helper>.test.js` |
| A new react-admin resource page | Cypress E2E for the most common user task; defer unit tests until the page stabilizes. |
| A new shared component (`components/shared/`) | Vitest unit test colocated as `<Component>.test.tsx`. Be the first to do this; the harness is empty. |
| A new model | Integration tests cover it. Plus, if you added a `pre('validate')` or other hook, a unit test that exercises the hook directly. |

## Conventions

### Backend tests

- One file per domain (`auth.test.js`, `sinh-vien.test.js`, etc.). The existing files in `backend/src/tests/` are the template.
- Use `supertest` against the in-process `app` instance — no HTTP server boot.
- `mongodb-memory-server` is set up globally; each test file gets a fresh database via `helpers.js`.
- Seed your own data — don't rely on test order or shared state.
- Use `expect(res.status).toBe(...)` and `expect(res.body).toMatchObject({...})`. Don't snapshot full bodies; the `_id` and timestamps make them flaky.

### Frontend tests (when you write them)

- Vitest config in `frontend/vitest.config.ts`, JSDOM environment, `@testing-library/react` for components.
- MSW (Mock Service Worker) is installed — use it to mock `/api/*` responses for unit tests. Don't `mock('axios')` or stub `fetch` directly.
- Cypress E2E tests use `data-testid` attributes for selectors.
- Each Cypress test should start from a clean session — call `cy.clearLocalStorage()` and re-login in a `beforeEach` if needed.

## What's intentionally not tested

- **Real Mongo connection.** Tests use mongodb-memory-server. Local dev still uses the real `MONGO_URI`.
- **Real partner-school cluster.** `MONGO_URI2`-backed models (`DonViLienKet`, `CertificateLookup`) get mock connections in tests.
- **Excel binary correctness.** The export pipeline is exercised by tests up to the point of "did the controller call `workbook.xlsx.write`"; the byte-level correctness of the output file is verified manually.
- **Performance.** No load tests. The system's expected concurrent user count is small (a single training institution's admin staff), and Mongo + Node can absorb that easily.

## Gaps to close (in priority order)

1. **First Vitest unit test.** The harness exists; nothing uses it. A reasonable starter: `components/shared/Button.test.tsx` covering the variant + disabled-state rendering.
2. **Cypress logout test.** Skipped in `login.cy.ts` waiting for the feature to ship. The feature is shipped; the test should be too.
3. **Integration tests for the newer route domains** — `khao-sat-chat-luong`, `truc-quan-su`, `tra-cuu`, `attachments`, `hoc-lieu`, `bieu-mau`. Each one currently has zero coverage.
4. **Performance baseline** — at least one timed Cypress run that records latency for the heaviest page (e.g. opening "Cơ sở dữ liệu sinh viên" with seeded data). Catches regressions cheaply.

## See also

- [`backend/docs/TESTING.md`](../../backend/docs/TESTING.md), [`frontend/docs/TESTING.md`](../../frontend/docs/TESTING.md) — command reference
