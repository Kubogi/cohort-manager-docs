# POST /api/can-bo-quan-ly/import

**Endpoint**: `POST /api/can-bo-quan-ly/import`  
**Authentication**: ✅ Required  

**Roles**: admin  
**Content-Type**: `multipart/form-data`  

**Last Verified**: 2026-05-16

---

## Description

Bulk-imports management staff from an Excel file (`.xls` or `.xlsx`). **Admin-only operation**.

---

## Request

### Headers
```
Authorization: Bearer <access_token>
Content-Type: multipart/form-data
```

### Form Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| file | File | Yes | Excel file (`.xls` / `.xlsx`), max 20 MB |

---

## Excel Format

- **Worksheet**: `DS CBQL`
- **Data starts**: Row 4 (rows 1–3 are headers)

| Column | Field | Notes |
|--------|-------|-------|
| B | `hoTen` + `capBac` | Combined cell; service parses rank from name |
| C | `capBac` fallback | Used if rank cannot be extracted from column B |
| D | `soDienThoai` | Phone number |
| F | `chucVu` | Position/title |

- **`donViQL`**: Determined by the fill color of column B cells — each color maps to a unit name.

---

## Response

### Success (200 OK)

```json
{
  "data": {
    "inserted": 42,
    "duplicates": 3,
    "errors": [{ "row": 5, "reason": "Tên trùng và số điện thoại trùng" }]
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| inserted | number | Records successfully created |
| duplicates | number | Rows skipped because the combination of `hoTen` + `soDienThoai` already exists |
| errors | object[] | Per-row failures. Each entry is `{ row, reason }`. |

---

## Error Responses

### 400 Bad Request
No file uploaded or unsupported file type.

### 403 Forbidden
Only admin role can import staff.

---

## Notes

- Duplicate detection is by the **combination** of `hoTen` and `soDienThoai` — two staff members with the same name but different phone numbers are NOT considered duplicates.
- The temp file is always deleted regardless of whether the service throws, because the controller wraps cleanup in a `finally` block.

---

## Related

- [POST /api/can-bo-quan-ly](./post.md)
- [GET /api/can-bo-quan-ly](./get-list.md)
- [CanBoQuanLy Schema](../../../backend/schemas/CanBoQuanLy.md)
