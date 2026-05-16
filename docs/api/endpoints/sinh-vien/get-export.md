# GET /api/sinh-vien/export/all

**Endpoint**: `GET /api/sinh-vien/export/all`  

**Authentication**: ✅ Required  

**Roles**: admin  

**Status**: ⚠️ **STUB - NOT IMPLEMENTED**  

**Last Verified**: 2026-05-16

---

## Description

**STUB ENDPOINT**: Currently returns empty array. Export functionality not implemented.

---

## Current Implementation

```javascript
export const exportData = asyncHandler(async (_req, res) => {
  res.json({ data: [], meta: { exported: 0 } });
});
```

---

## Response

### Success (200 OK)

```json
{
  "data": [],
  "meta": {
    "exported": 0
  }
}
```

---

## Notes

- **Not Implemented**: This endpoint is a placeholder
- Always returns empty data array
- Actual export logic needs to be added
- Expected to generate CSV/Excel file for download

---

## Related

- [POST /api/sinh-vien/import](./post-import.md)
- [GET /api/sinh-vien](./get-list.md) - List students
- [SinhVien Schema](../../../backend/schemas/SinhVien.md)
