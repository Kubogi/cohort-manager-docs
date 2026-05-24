# Biên chế đại đội tự động — Battalion partition planning tool

**Menu path**: Quản lý sinh viên › Biên chế đại đội tự động
**Roles**: admin · staff *(viewer can open but the backend rejects compute; teacher cannot see "Quản lý sinh viên")*
**Source files**: [`frontend/src/resources/quanLySinhVien/bienCheDaiDoiTuDong.tsx`](../../../../frontend/src/resources/quanLySinhVien/bienCheDaiDoiTuDong.tsx) + companion folder `bienCheDaiDoiTuDong/bienChe/` (BienCheTab, ClassInputPanel, PartitionResultPanel, useBienCheData)
**Related API endpoints**: [`POST /api/bien-che/partition`](../../../api/endpoints/bien-che/partition-post.md), [`POST /api/bien-che/parse-classes`](../../../api/endpoints/bien-che/parse-classes-post.md)

## When to use

Planning an intake before any students are entered: you have a list of classes and their student counts on paper / in Excel and need to figure out how many đại đội to split them into so each đại đội has roughly the same headcount. This page is **a calculator only** — it does not read or modify any student records, đại đội records, or anything else in the database.

## Layout

Single page (no tabs). Two side-by-side panels:

- **Left — Danh sách lớp** (`ClassInputPanel.tsx`): a `Số đại đội` input + `Biên chế` button on top, then the editable class table, then the action row (`Nhập Excel`, `+ Thêm lớp`, `Xóa toàn bộ`) and a live totals summary.
- **Right — Kết quả biên chế** (`PartitionResultPanel.tsx`): the partition the backend computes, one sub-table per đại đội, plus a stat strip (Tổng / Cao nhất / Thấp nhất / Chênh lệch).

> The historical **Chuyển đại đội** tab has been removed from this page. Its code lives untouched under `bienCheDaiDoiTuDong/chuyenDoi/` and can be brought back later under a new entry point if needed.

## Input UX (left panel)

- **`Số đại đội` + `Biên chế` action bar sits above the table** — natural top-down order, and the action stays visible after a long class list.
- Inline editable table with columns: STT, Tên lớp, Số sinh viên, (✕ delete).
- Action row below the table: **Nhập Excel** | **+ Thêm lớp** | **Xóa toàn bộ**.
- **Nhập Excel** opens a hidden `<input type="file">` (`.xls`/`.xlsx`) and posts the chosen file to [`POST /api/bien-che/parse-classes`](../../../api/endpoints/bien-che/parse-classes-post.md). The endpoint reads every sheet, finds the `Lớp` column by **header name** (so K178-style sheets with Lớp in col E and coSoDuLieuSinhVien-style sheets with Lớp in col I both parse cleanly), counts students per class, and sums cross-sheet duplicates into one row. Sheets without a Lớp header (e.g. summary tabs like `SỐ LƯỢNG`) are skipped silently. On success the table rows are **replaced** with the parsed list and the prior partition result is cleared. If the table already has any non-empty cell, a `window.confirm` gate fires first — accept to replace, cancel to keep what's there. The button shows "Đang nhập…" and is disabled while the request is in flight.
- **+ Thêm lớp** appends a blank row. **Xóa toàn bộ** resets the form (including any prior result).
- The ✕ delete button is disabled when only one row remains (otherwise the table would collapse to nothing).
- Footer line under the table tracks `Tổng số lớp` and `Tổng số sinh viên` live as you type — useful as a sanity check before computing.
- `Số sinh viên` and `Số đại đội` are `<Input type="number">`. The hook keeps them as strings internally to preserve normal typing UX (no premature coercion), then validates and converts at submit time.

## Validation (client-side, mirrors the backend)

Before any network call, the hook emits a Vietnamese warning notify and aborts for each of:

- Every row blank → "Hãy thêm ít nhất một lớp."
- A row has `soSinhVien` filled but no `ten` → "Tên lớp không được trống."
- Any `soSinhVien` is not a positive integer → "Số sinh viên phải là số nguyên dương."
- `Số đại đội` empty / non-positive integer → "Số đại đội phải là số nguyên dương."
- `Số đại đội > số lớp` → "Số đại đội không được lớn hơn số lớp."

The backend revalidates and returns 400 with the same messages if these slip through. A backend rejection (network error, 400, etc.) leaves `result` untouched and surfaces a generic "Không thể tính phân bổ. Vui lòng thử lại." error notify.

## Algorithm

The backend uses **LPT (Longest Processing Time first)** seed + **pairwise swap/move refinement** to minimise `max - min` of per-bucket totals. Details + endpoint contract: [`POST /api/bien-che/partition`](../../../api/endpoints/bien-che/partition-post.md). Deterministic — the same input always returns the same partition.

## Result UX (right panel)

- Before the first compute: placeholder "Nhập danh sách lớp và bấm 'Tính toán phân bổ' để xem kết quả."
- After a successful compute: stat strip + one table per đại đội (header "Đại đội N — Tổng: …"), classes within each đại đội sorted by `soSinhVien` desc.
- No export. Result is on-screen only.

## Common tasks

### Plan a 3-đại-đội split for 8 classes

1. Type each class's name + headcount into the table (use **+ Thêm lớp** to add rows).
2. Enter `Số đại đội = 3`.
3. Click **Tính toán phân bổ**. The right panel shows the assignment and `Chênh lệch` (spread) — the smaller, the more balanced.
4. To experiment with a different bucket count, change `Số đại đội` and click again. The old result is replaced.

### Start over

Click **Xóa toàn bộ**. The class table collapses to one blank row, `Số đại đội` clears, and the result panel returns to the placeholder.

## Edge cases / gotchas

- **No database touched.** Computing a partition does not assign students to đại đội. To actually move students, use the bulk-edit / drag-drop flows on the [Cơ sở dữ liệu sinh viên](./co-so-du-lieu-sinh-vien.md) page (or whichever transfer UI is currently active).
- **Duplicate class names are rejected.** Backend returns 400. Class names must be unique (after trim) within one request.
- **Spread isn't always 0.** With integer counts, perfect balance is often impossible (e.g. total 434 ÷ 3 = 144.67); the algorithm reaches the *integer* optimum, which may be 1–10 students apart. The stat strip's `Chênh lệch` is the gap between the busiest and quietest đại đội.
- **Practical scale**: tested at a few dozen classes × ~10 đại đội. Feeding hundreds of classes will still work but the swap pass cost grows roughly with k² · n².
- **Viewer + teacher**: viewer can navigate to the page but `POST /api/bien-che/partition` will 403; teacher doesn't see "Quản lý sinh viên" at all.
