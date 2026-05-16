# ADR 0003 — Vietnamese camelCase resource names + a translation map to kebab-case API paths

**Status**: Accepted
**Date**: ~2024 (recorded retroactively 2026-05-16)

## Context

The application is built for Vietnamese-speaking operators. The UI is in Vietnamese. Domain concepts (sinh viên, đại đội, khóa, quyết định, hồ sơ sức khỏe) don't have one-word English equivalents that the operators would recognize. Forcing English identifiers everywhere would mean every developer who touches the codebase has to maintain a mental glossary, AND the API surface would diverge from the language operators actually speak when describing bugs.

But: REST conventions, URL hygiene, and tooling are firmly in the camelCase / kebab-case English-ish world.

## Decision

Use Vietnamese for identifiers, but with **discipline about the style per layer**:

| Layer | Style | Examples |
|---|---|---|
| Mongoose model name + JS variable names | Vietnamese **camelCase** | `sinhVien`, `daiDoi`, `quyetDinh`, `hoSoSucKhoe` |
| Mongo collection name (on disk) | Vietnamese **snake_case** | `sinh_vien`, `dai_doi`, `quyet_dinh`, `ho_so_suc_khoe` |
| REST API path segment | Vietnamese **kebab-case** | `/api/sinh-vien`, `/api/dai-doi`, `/api/quyet-dinh` |
| react-admin Resource `name` | Vietnamese **camelCase** | `coSoDuLieuSinhVien`, `quanLyCacQuyetDinh`, `hoSoSucKhoe` |
| URL route in the SPA | Vietnamese **camelCase** | `#/coSoDuLieuSinhVien`, `#/quanLyCacQuyetDinh` |

The translation between the SPA's camelCase resource name and the backend's kebab-case path is centralized in [`frontend/src/dataProvider.tsx#resourcePathMap`](../../frontend/src/dataProvider.tsx):

```ts
const resourcePathMap: Record<string, string> = {
  khoaHoc:              'khoa-hoc',
  giangVien:            'can-bo-quan-ly',       // NOT 'giang-vien'
  coSoDuLieuSinhVien:   'sinh-vien',
  quanLyCacQuyetDinh:   'quyet-dinh',
  hoSoSucKhoe:          'ho-so-suc-khoe',
  nhapDiem:              'diem',
  trucQuanSu:           'truc-quan-su',
};
```

Resources not in the map fall back to a naive `camelCase → kebab-case` conversion.

## Consequences

### Positive

- Operators and developers speak the same vocabulary. Bug reports like "the đại đội field on the sinh viên modal isn't filtering correctly" map 1:1 onto code locations.
- The glossary ([`/docs/glossary.md`](../glossary.md)) provides a single point of translation; future English-speaking contributors can use it to onboard.
- The translation map is *minimal* — only 7 entries — because most resource names happen to be the same word as the URL segment after kebab-casing.

### Negative

- **Visual collisions.** `khoa` (faculty) and `khóa` (cohort/course) are spelled identically without diacritics. The code uses `khoa` for the unit and `khoaHoc` for the cohort, but in URL paths (`/api/don-vi/khoa` vs `/api/khoa-hoc`) the distinction is only by adjacency. This is documented prominently in `glossary.md` and `architecture/database.md`.
- **The `giangVien` ↔ `can-bo-quan-ly` mapping is non-obvious.** The frontend's display label "Giảng viên" suggests an endpoint called `giang-vien`, but the actual endpoint is `can-bo-quan-ly` (because that's the backend model's name). Two old frontend code paths bug-leveraged the naive kebab-case fallback to call non-existent `/api/giang-vien` endpoints; this map fixes them.
- **English-speaking contributors need the glossary.** There's no avoiding it.
- **Linting/IDE tools don't catch Vietnamese identifier typos** the way they do English. A typo like `sinhVeen` won't get auto-corrected; runtime is when you find out.

## Alternatives considered

- **Translate everything to English** (`students`, `battalions`, `courses`). Rejected because the English translations are awkward and lossy — "battalion" doesn't quite capture đại đội; "cohort" is closer to khóa but doesn't carry the military-school connotation; "decision" loses the bureaucratic specificity of quyết định.
- **English at every layer except the UI labels.** Rejected because the UI label is what's displayed in error messages and search; keeping it disconnected from the model name would lead to cognitive overhead when debugging.
- **Vietnamese with diacritics in identifiers** (e.g. `khóaHọc`, `đạiĐội`). Rejected — many tools (some terminal fonts, some lint rules, some Vietnamese keyboard auto-replace) garble non-ASCII identifiers. Stripping diacritics is the smaller sacrifice.

## See also

- [`/docs/glossary.md`](../glossary.md) — the canonical EN ↔ VI translation
- [`/docs/architecture/frontend.md`](../architecture/frontend.md) — how the resource ↔ path mapping is wired
- [`/frontend/src/dataProvider.tsx`](../../frontend/src/dataProvider.tsx) — the translation map source
