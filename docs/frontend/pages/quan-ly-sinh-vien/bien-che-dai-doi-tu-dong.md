# Biên chế đại đội tự động — Automatic battalion assignment

**Menu path**: Quản lý sinh viên › Biên chế đại đội tự động

**Roles**: admin · staff · viewer *(not visible to teacher)*

**Source files**: [`frontend/src/resources/quanLySinhVien/bienCheDaiDoiTuDong.tsx`](../../../../frontend/src/resources/quanLySinhVien/bienCheDaiDoiTuDong.tsx) + companion folder `bienCheDaiDoiTuDong/`

**Related API endpoints**: [`PATCH /api/sinh-vien/:id`](../../../api/endpoints/sinh-vien/patch.md), [`GET /api/sinh-vien`](../../../api/endpoints/sinh-vien/get-list.md), [`GET /api/don-vi/dai-doi`](../../../api/endpoints/don-vi/dai-doi-get-list.md)

**Related workflows**: [`auto-battalion-assignment.md`](../../../workflows/auto-battalion-assignment.md)

## When to use

After a fresh roster lands (typically right after [Excel import of students](../../../workflows/excel-import-students.md) or manual creation), every new student needs to be placed into a battalion. This page lets staff distribute them in bulk by rule instead of dragging one record at a time. Use it again later if a transfer is needed between battalions.

## Layout

- Tab bar: **Biên chế đại đội** | **Chuyển đại đội**.
- Each tab has its own filter bar and a two-column transfer-list UI.

## Tabs

### Tab 1 — "Biên chế đại đội" (initial assignment)

**What it does:**
- Filter by `khoa` (and optionally `truong`). Loads two pools:
  - **Left**: students in this khoa with no `daiDoi` (or with a deprecated one).
  - **Right**: a list of `DaiDoi` belonging to this khoa, each with current `quanSo` and a capacity hint.
- Apply a distribution rule (e.g. round-robin, fill-first, by gender quota — depends on implementation). The UI computes a preview of who lands where.
- **Lưu thay đổi** writes each `SinhVien.daiDoi` via per-student `PATCH /api/sinh-vien/:id`.

**What it shows:**
- Live count of unassigned students.
- Per-battalion target vs current count.
- Preview of the planned moves before commit.

### Tab 2 — "Chuyển đại đội" (bulk transfer)

**What it does:**
- Filter source `(khoa, daiDoi)` and pick a destination `daiDoi`.
- Students are transferred using **drag-and-drop** — rows have the `draggable` attribute with `onDragStart`, `onDragOver`, and `onDrop` handlers. There are no checkboxes.
- Writes each student's new `daiDoi` via `PATCH /api/sinh-vien/:id`.

**What it shows:**
- Source and destination battalion student counts.
- Confirmation modal listing the affected students before commit.

## Common tasks

### Distribute a freshly imported intake

1. Run the Excel import on Cơ sở dữ liệu sinh viên first.
2. Open this page, **Tab 1**. Pick `khoa K48`.
3. Pick a distribution rule. Click **Biên chế** to compute/preview.
4. Inspect the preview. If it looks right, click **Lưu thay đổi** to save.

### Move 5 students from D1 to D2

1. Open **Tab 2**. Set source `K48 / D1`, destination `K48 / D2`.
2. Drag the 5 students you want moved from the source list to the destination list.
3. Click **Chuyển**. Confirm in the modal.

## Edge cases / gotchas

- **Per-student PATCH means no atomicity across the batch.** If a network blip kills the request mid-loop, some students are moved and some aren't. The page should re-fetch and let you retry on the remainder; verify before walking away.
- **Cross-khoa moves are disallowed.** The `sinhVien` validator (commit `7840d44`) checks `daiDoi.khoa === payload.khoa`. Trying to send a student from `K47 / D1` to `K48 / D2` returns a 400 — the destination has to be in the same khoa unless you change `khoa` in the same payload.
- **DaiDoi's `quanSo` is not auto-updated** on student assignment. It's a hint maintained separately on the DaiDoi document; after a large reassignment, the dispatcher should reconcile `quanSo` per battalion.
- **Teacher cannot see this page** — `App.tsx` hides "Quản lý sinh viên" from teachers.
- **Capacity hints are advisory.** Nothing prevents you from assigning 200 students to a battalion. The UI may show a warning past a threshold but it's not a hard cap.
