# Core Operations

The IDO REST API exposes three primary operations. All require a valid token in the `Authorization` header (see [01_AUTHENTICATION.md](01_AUTHENTICATION.md)).

**Base URL:** `$SYTELINE_BASE_URL`

---

## 1. Load Collection (Query Data)

Retrieve rows from an IDO. This is a **GET** request.

```
GET /load/{IDO}?properties={columns}&filter={where}&orderby={sort}&recordcap={limit}
Authorization: {token}
```

| Parameter | Required | Description |
|---|---|---|
| `properties` | No | Comma-delimited property names. Omit to get all properties. |
| `filter` | No | SQL WHERE clause (URL-encoded). See filter syntax below. |
| `orderby` | No | Comma-delimited sort. Append `DESC` per column as needed. |
| `recordcap` | No | `-1` = 200 rows (default), `0` = all rows, `N` = N rows |

> **Tip:** During discovery, omit `properties` to see all available columns, and set `recordcap=10` or smaller to avoid overwhelming the response.

### Response

```json
{
  "Items": [
    { "Property1": "value1", "Property2": "value2" },
    { "Property1": "value3", "Property2": "value4" }
  ],
  "Bookmark": "...",
  "MoreRowsExist": true,
  "Success": true,
  "Message": null
}
```

- `Items` — array of objects, one per row
- `MoreRowsExist` — `true` if the record cap was hit and more data exists
- `Bookmark` — pagination cursor (pass back for next page)

### Example: Query items from a table

```bash
curl -s -X GET \
  "$SYTELINE_BASE_URL/load/SLItems?properties=Item,Description,UnitCost&filter=Item%20LIKE%20N'ABC%25'&recordcap=10" \
  -H "Authorization: $TOKEN"
```

### Filter Syntax

Filters use SQL WHERE syntax, URL-encoded. String literals must be wrapped in `N'...'` (Unicode prefix).

```
# Exact match
filter=TaskName%20%3D%20N'ChangeCOStatusUtility'

# Starts with
filter=TaskName%20LIKE%20N'Change%25'

# Contains
filter=TaskName%20LIKE%20N'%25Journal%25'

# AND
filter=TaskName%20%3D%20N'OrderVerificationReport'%20AND%20RequestingUser%20%3D%20N'npetrie'

# Date comparison
filter=CompletionDate%20%3E%20'2026-02-01'
```

**URL encoding reference:**

| Char | Encoded | Purpose |
|---|---|---|
| space | `%20` | Separates tokens |
| `=` | `%3D` | Comparison operator |
| `%` | `%25` | SQL LIKE wildcard |
| `>` | `%3E` | Greater than |
| `<` | `%3C` | Less than |
| `'` | `%27` | String delimiter (usually OK unencoded) |

### Filters: What's Allowed and Not Allowed

Filters support: property names, comparison operators (`=`, `<>`, `>`, `<`, `>=`, `<=`, `LIKE`), Boolean operators (`AND`, `OR`), `IN`, `NOT`, `IS NULL`, `IS NOT NULL`, and string literals.

**Not allowed** in filters: subqueries, function calls, or anything that could be used for SQL injection. Filters are validated server-side.

---

## 2. Invoke (Execute a Method)

Execute a method defined on an IDO. This is a **POST** request.

```
POST /invoke/{IDO}?method={METHOD}
Authorization: {token}
Content-Type: application/json
Body: [ param1, param2, param3, ... ]
```

The body is a **flat JSON array** of positional parameters. Not wrapped in an object — just the raw array.

### Response

```json
{
  "Message": null,
  "Success": true,
  "Parameters": ["echoed_param1", "echoed_param2", "output_value", ...],
  "ReturnValue": "0"
}
```

- `ReturnValue: "0"` — success
- `ReturnValue: "16"` — validation error (check output parameters for error message)
- `Success: false` + `Message` — API-level error (permissions, parsing, etc.)

### OUTPUT Parameters

Methods can have input, output, or input/output parameters. Output values are returned in the same positional index in `Parameters[]`. Pass `null` for output-only parameters in the request.

### Example: Call a method

```bash
curl -s -X POST \
  "$SYTELINE_BASE_URL/invoke/UserNames?method=GetUserAttributes" \
  -H "Authorization: $TOKEN" \
  -H "Content-Type: application/json" \
  -d '["jdoe",null,null,null,null]'
```

Output parameters come back in their positional indices:
```json
{
  "Parameters": ["jdoe", "2", "1", "", ""],
  "ReturnValue": "0"
}
```

---

## 3. Update Collection (Insert / Update / Delete)

Modify data in an IDO. This is a **POST** request.

```
POST /update/{IDO}
Authorization: {token}
Content-Type: application/json
```

### Insert

```json
{
  "Items": [
    {
      "Action": "Insert",
      "Properties": {
        "PropertyName1": "value1",
        "PropertyName2": "value2"
      }
    }
  ]
}
```

### Update by Key

```json
{
  "Items": [
    {
      "Action": "Update",
      "Properties": {
        "KeyProperty": "key_value",
        "PropertyToChange": "new_value"
      }
    }
  ]
}
```

### Delete by Key

```json
{
  "Items": [
    {
      "Action": "Delete",
      "Properties": {
        "KeyProperty": "key_value"
      }
    }
  ]
}
```

### Important Notes

- To update or delete, you need the **key values** for the record. Find keys using `SqlColumns` with `isPrimaryKey=1` — see [05_DISCOVERY_GUIDE.md](05_DISCOVERY_GUIDE.md#step-6-find-primary-keys).
- Include all non-nullable properties that don't have defaults when inserting.
- The IDO runtime handles the SQL generation — you never write raw INSERT/UPDATE/DELETE statements.
- **Before deleting,** always LoadCollection first with a small `recordcap` to verify the scope of your filter. Many tables have compound keys (e.g., item + currency + date + site), and a filter on one column may match far more records than intended.
- If converting a raw SQL `DELETE FROM` to an API call, see the [Delete conversion walkthrough](05_DISCOVERY_GUIDE.md#practical-example-convert-a-sql-delete-to-an-api-call) for the full discovery process.

---

## HTTP Method Summary

| Operation | HTTP Method | Endpoint Pattern |
|---|---|---|
| Query data | `GET` | `/load/{IDO}?properties=...&filter=...` |
| Execute method | `POST` | `/invoke/{IDO}?method={Method}` |
| Insert/Update/Delete | `POST` | `/update/{IDO}` |

**Common mistake:** Using POST for `/load` or GET for `/invoke` returns `405 Method Not Allowed`.
