# Gotchas & Troubleshooting

Proven pitfalls from real testing against the Syteline IDO REST API.

---

## Permissions & Safety

### Two users, two roles

- **Agent user** (`$SYTELINE_AGENT_USERNAME`) — introspection only. Has meta-IDO permissions to discover IDO structures, methods, and parameters. Should **not** have access to real business data.
- **Automation user** (`$SYTELINE_AUTOMATION_USERNAME`) — optional. Used to test and execute the API calls the agent builds. Grant task-specific IDO permissions to this user as needed.

### Exploratory calls are safe

The agent user is scoped to read-only introspection. LoadCollection calls and failed Invoke attempts return clear error messages — they won't break anything. It's okay to try things during discovery.

### Ask before modifying data

Before executing Invoke calls on methods that modify state, or Update calls (Insert/Update/Delete), confirm with the user. Even if the API returns a permission error, it's better to ask first.

### Request missing permissions explicitly

If you need access to an introspection IDO (like `IdoMethodParameters`) and get a permission error, tell the user. These are safe, read-only IDOs and the user's admin can grant access to the **agent user** quickly. Don't silently skip a discovery step because of a permission error.

---

## Testing with the Automation User

Once you have built the necessary API call(s) that solve the user's request, ask the user if they want to test them using the automation user (`$SYTELINE_AUTOMATION_USERNAME`).

### Workflow

1. **Build the API call(s)** using discovery (with the agent user).
2. **Present the call(s) to the user** — show them the curl command(s) you've constructed.
3. **Ask if they want to test.** If `$SYTELINE_AUTOMATION_USERNAME` is configured and the user agrees, authenticate as the automation user and execute the call(s).

```bash
# Authenticate as the automation user for testing
SITE_CONFIG="${DEFAULT_SITE:-Demo_DALS}"
AUTOMATION_TOKEN=$(curl -s "$SYTELINE_BASE_URL/token/$SITE_CONFIG" \
  -H "username: $SYTELINE_AUTOMATION_USERNAME" \
  -H "password: $SYTELINE_AUTOMATION_PASSWORD" | jq -r '.Token')

# Execute the API call with the automation user's token
curl -s -X GET \
  "$SYTELINE_BASE_URL/load/SLItemprices?properties=Item,UnitPrice1&filter=Item%20%3D%20N'WIDGET-A'&recordcap=10" \
  -H "Authorization: $AUTOMATION_TOKEN"
```

### Handling insufficient permissions

If the test returns a permission error (e.g., `"User requires [Read] privilege for SLItemprices"`), ask the user to grant the required permission to the **automation user** (`$SYTELINE_AUTOMATION_USERNAME`) for the specific IDO(s). **Do not** grant these permissions to the **agent user** (`$SYTELINE_AGENT_USERNAME`) — the agent user should remain limited to introspection.

Example message to the user:
> The automation user needs `[Read]` permission on the `SLItemprices` IDO. Please grant this in Syteline's Object Authorizations for the automation user only — not the agent user.

---

## Data Type Errors

### BYTE fields reject booleans

`PrintPreview`, `SchedFreqType`, and other BYTE fields must be `0` or `1` — **not** `false`/`true`.

```
WRONG:  [..., "username", false, ...]
ERROR:  "IDOValueType.GetValue: failed to parse normalized string [false], Type = Byte"

RIGHT:  [..., "username", 0, ...]
```

### Parameter arrays must be flat

The invoke body is a raw JSON array. Do **not** wrap it.

```
WRONG:  { "parameters": ["TaskName", ...] }
RIGHT:  ["TaskName", ...]
```

### All positions must be present

For methods like BGTaskSubmit (20 parameters), every position must exist. Use `null` for unused slots. Short arrays cause parsing errors.

---

## Authentication Errors

### "User is not licensed..."

The user doesn't have IDO-level permissions or a license module covering the IDO.

**Fix:** Grant the user authorization in Syteline's Object Authorizations (at the IDO level, not form level). API sessions check permissions against IDOs, not forms. For introspection IDOs, grant to the **agent user**. For business-data IDOs, grant to the **automation user** only.

### "format is invalid" in PowerShell

Tokens contain special characters (`/`, `+`, `=`) that break `Invoke-RestMethod` header parsing.

**Fix:** Use `curl` or `[System.Net.WebClient]` instead. The bash/PowerShell curl wrapper handles it correctly.

### Token not prefixed with Bearer

Unlike most REST APIs, the token goes directly into the `Authorization` header without a `Bearer ` prefix.

```
WRONG:  -H "Authorization: Bearer b/XdI6IQ..."
RIGHT:  -H "Authorization: b/XdI6IQ..."
```

---

## HTTP Method Errors

### 405 Method Not Allowed

- `/load` is **GET only**
- `/invoke` is **POST only**
- `/update` is **POST only**

Mixing these up returns `405`.

---

## Task Submission Errors

### "Task Name is invalid"

The `TaskName` (index 0) doesn't match any defined task. Task names are case-sensitive.

**Fix:** Query `ActiveBGTasks` to find the exact name:
```bash
curl -s "$SYTELINE_BASE_URL/load/ActiveBGTasks?properties=TaskName&filter=TaskName%20LIKE%20N'%25YourTask%25'&recordcap=10" \
  -H "Authorization: $TOKEN"
```

### ReturnValue "16" — Validation error

The task name exists but the parameters failed validation. Check `Parameters[3]` (Infobar) in the response for the actual error message.

---

## Stored Procedure Conversion Errors

### "You may not call this 'SpName' as it has been converted to custom assembly application method"

This means the stored procedure has been migrated from raw SQL to an extension class (C# DLL). Direct `EXEC SpName` calls from SQL no longer work.

**Fix:** Call it through the IDO REST API instead. See the [Discovery Guide](05_DISCOVERY_GUIDE.md) for how to find the IDO and method name:

```bash
# Find which IDO hosts the converted SP
curl -s "$SYTELINE_BASE_URL/load/IdoMethods?properties=CollectionName,MethodName,StoredProcedure&filter=StoredProcedure%20LIKE%20N'%25SpName%25'&recordcap=0" \
  -H "Authorization: $TOKEN"
```

Then invoke it via `/invoke/{IDO}?method={MethodName}`.

---

## Date/Time Formats

Dates in SETVARVALUES and filter parameters use SQL Server format:
```
2026-02-28 00:00:00.000
```

Date-only comparisons in filters:
```
CompletionDate%20%3E%20'2026-02-01'
```

---

## String Literals in Filters

Always use the `N'...'` Unicode prefix for string values in filters:

```
WRONG:  filter=TaskName%20%3D%20'ChangeCOStatusUtility'
RIGHT:  filter=TaskName%20%3D%20N'ChangeCOStatusUtility'
```

Without `N`, non-ASCII characters and some comparisons may behave unexpectedly.

---

## SETVARVALUES Gotchas

- **NewId/GUID fields** — Generate a fresh GUID per submission. Don't reuse.
- **Parm_Site** — Must match the site for the token you're using.
- **Empty values** — Use `Key=` (key with empty value), not omit the key.
- **Spacing** — Some historic examples have spaces around delimiters; the system appears to trim them, but match what history shows to be safe.

---

## Record Cap Defaults

If you omit `recordcap`, the API returns **200 rows maximum** (`-1` default). To get all rows:

```
recordcap=0
```

During discovery, always set a small `recordcap` (e.g., `10`) to avoid overwhelming responses. Only use `recordcap=0` with a narrow filter.

---

## SQL Table Name Suffixes

Database table names in `SqlColumns` use a `_mst` suffix (e.g., `itemprice_mst`), but the IDO layer references them without it (e.g., `itemprice`). When querying `SqlColumns`, use the `_mst` version. When querying `IdoTables`, use the version without `_mst`.

Check `MoreRowsExist` in the response to know if you're missing data.
