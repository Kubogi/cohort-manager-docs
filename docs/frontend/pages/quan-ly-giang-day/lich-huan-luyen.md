# Lịch huấn luyện — Training calendar (STUB)

**Menu path**: Quản lý giảng dạy › Lịch huấn luyện

**Roles**: admin · staff · viewer · teacher

**Source files**: [`frontend/src/resources/quanLyGiangDay/lichHuanLuyen.tsx`](../../../../frontend/src/resources/quanLyGiangDay/lichHuanLuyen.tsx)

**Status**: ⚠ **STUB** — the page is registered as a Resource in `App.tsx` and shows up in the menu, but the component is a placeholder.

## When (not) to use

This page currently renders a "Chức năng đang được phát triển" (Feature under development) message. Don't promise users that it works. The corresponding backend route does not exist either.

## When the feature ships

A training-calendar UI is planned: shareable read-only calendar across instructors, possibly synchronized with [Lịch trình đào tạo](../thong-tin-chung/lich-trinh-dao-tao.md). When implementation begins:

- Decide whether to embed a calendar widget (FullCalendar, Material UI date picker) or build from scratch.
- Decide whether to store events in Mongo or fetch from an external Google Calendar.
- Add `useTeacherScope()` to filter the view per teacher's `khoa` / `daiDoi`.

Until that decision is made, this doc just notes the page is a stub.
