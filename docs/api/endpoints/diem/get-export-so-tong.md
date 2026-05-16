# GET /api/diem/export-so-tong

**Endpoint**: `GET /api/diem/export-so-tong`
**Authentication**: ✅ Required

**Roles**: admin, staff, viewer, teacher

**Last Verified**: 2026-05-16

**Unit Scoping**: ✅ Applied (staff: `allowedUnits`; teacher: `teacherScope`)

---

## Description

Generates and streams an Excel workbook (`.xlsx`) containing the full grade roster ("sổ tổng") for the specified cohort, school, and education system. The response is a binary file download, not a JSON envelope.

---

## Request

### Headers
```
Authorization: Bearer <access_token>
```

### Query Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| he | string | **Yes** | Education system. Must be `"Đại học"` or `"Cao đẳng"`. |
| khoa | string | **Yes** | Cohort ObjectId. |
| truong | string | **Yes** | Partner school ObjectId. |
| daiDoi | string | No | Battalion ObjectId. Required when `scope=single`. |
| scope | string | No | `"all"` (default when `daiDoi` is absent) or `"single"` (exports only the specified battalion). If `daiDoi` is provided and `scope` is omitted, defaults to `"single"`. |

---

## Response

### Success (200 OK)

Binary `.xlsx` file. Response headers:

```
Content-Type: application/vnd.openxmlformats-officedocument.spreadsheetml.sheet
Content-Disposition: attachment; filename="<encoded-filename>.xlsx"
```

---

## Error Responses

### 400 Bad Request — missing required parameters

```json
{
  "error": {
    "code": "MISSING_PARAMS",
    "message": "Thiếu tham số: he, khoa, truong"
  }
}
```

**When**: `he`, `khoa`, or `truong` is absent.

### 400 Bad Request — invalid education system

```json
{
  "error": {
    "code": "INVALID_HE",
    "message": "Hệ không hợp lệ: ..."
  }
}
```

### 400 Bad Request — invalid scope

```json
{
  "error": {
    "code": "INVALID_SCOPE",
    "message": "scope phải là single hoặc all"
  }
}
```

### 400 Bad Request — missing daiDoi for single-scope

```json
{
  "error": {
    "code": "MISSING_PARAMS",
    "message": "Thiếu tham số daiDoi cho scope=single"
  }
}
```

### 404 Not Found

Returned when the specified `khoa` or `truong` does not exist (`KHOA_NOT_FOUND` / `TRUONG_NOT_FOUND`).

---

## Related

- [GET /api/diem/xep-loai-by-unit](./get-xep-loai-by-unit.md)
- [GET /api/diem/summary](./get-summary.md)
- [Excel pipeline architecture](../../../architecture/excel-pipeline.md)
