# Authentication

## Token Endpoint

Credentials are passed as HTTP headers — **not** in the URL path.

```
GET /token/{config}
Headers:
  username: {USERNAME}
  password: {PASSWORD}
```

The `{config}` selects which Syteline site/database to authenticate against.

### Config Selection

Do not hardcode a config list in scripts or docs. Config names are environment-specific.

- Default to `DEFAULT_SITE` from `.env`.
- Only discover/confirm sites if `DEFAULT_SITE` is not applicable, or if the user explicitly references another site.
- Use `SLSites` discovery (after authenticating) to list available sites and validate names when needed.

## Getting a Token

All discovery and introspection uses the **agent user**. When testing or executing API calls, use the **automation user** if configured (see [06_GOTCHAS.md — Testing with the Automation User](06_GOTCHAS.md#testing-with-the-automation-user)).

### Bash

```bash
SITE_CONFIG="${DEFAULT_SITE:-Demo_DALS}"
TOKEN=$(curl -s "$SYTELINE_BASE_URL/token/$SITE_CONFIG" \
  -H "username: $SYTELINE_AGENT_USERNAME" \
  -H "password: $SYTELINE_AGENT_PASSWORD" | jq -r '.Token')
```

### PowerShell

```powershell
$SITE_CONFIG = if ($env:DEFAULT_SITE) { $env:DEFAULT_SITE } else { "Demo_DALS" }
$TOKEN = (curl -s "$env:SYTELINE_BASE_URL/token/$SITE_CONFIG" `
  -H "username: $env:SYTELINE_AGENT_USERNAME" `
  -H "password: $env:SYTELINE_AGENT_PASSWORD" | ConvertFrom-Json).Token
```

### Response

```json
{
  "Message": null,
  "Success": true,
  "Token": "b/XdI6IQzCviZOGJ0E+002DoKUFOPm..."
}
```

- `Success: true` — token is valid
- `Success: false` — check `Message` for details (bad credentials, unlicensed user, etc.)

## Using the Token

The token is passed as the `Authorization` header on all subsequent requests. It is **not** prefixed with `Bearer` — use the raw value.

```bash
curl -s -X GET \
  "$SYTELINE_BASE_URL/load/SomeIDO?properties=Col1,Col2" \
  -H "Authorization: $TOKEN"
```

### PowerShell: Always Use `curl.exe`, Not `Invoke-RestMethod`

PowerShell's `Invoke-RestMethod` rejects the Syteline token format — it throws `"The format of value '...' is invalid"` because it tries to parse the `Authorization` header as a standard scheme (`Bearer`, `Basic`, etc.).

**Always use `curl.exe`** (the native Windows binary, not the PowerShell alias) for all Syteline API calls:

```powershell
# Correct — curl.exe passes the header as-is
curl.exe -s "$env:SYTELINE_BASE_URL/load/SomeIDO?properties=Col1,Col2" `
  -H "Authorization: $TOKEN"

# Wrong — Invoke-RestMethod rejects the raw token value
Invoke-RestMethod -Uri "..." -Headers @{ Authorization = $TOKEN }  # DO NOT USE
```

`curl.exe` is present on Windows 10/11 and Windows Server 2019+ by default. On older systems, install it from [curl.se](https://curl.se) or use WSL.

## Multi-Site Workflows

Each site requires its own token. Start with `DEFAULT_SITE` for single-site tasks. Only run discovery and multi-site loops when `DEFAULT_SITE` is not applicable or the user asks for additional sites.

```bash
# Step 1: discover available sites in the current environment
curl -s "$SYTELINE_BASE_URL/load/SLSites?properties=Site,Description&recordcap=0" \
  -H "Authorization: $TOKEN"

# Step 2: ask user which site configs to process, then loop only those
for SITE_CONFIG in "$@"; do
  TOKEN=$(curl -s "$SYTELINE_BASE_URL/token/$SITE_CONFIG" \
    -H "username: $SYTELINE_AGENT_USERNAME" \
    -H "password: $SYTELINE_AGENT_PASSWORD" | jq -r '.Token')
  # ... use $TOKEN for this selected site ...
done
```

## Permissions Model

API sessions check permissions against **IDOs**, not forms. The user account must be granted:

1. **IDO-level authorization** — permission to access the specific IDO (LoadCollection, UpdateCollection, Invoke)
2. **License module** — at least one license module that includes the IDO being accessed

This differs from the Syteline UI, where permissions are checked against forms.

> If you get `"You are not licensed to use the {FormName} form"`, the user needs either form access or a license module that covers the target IDO. For introspection errors, grant the permission to the **agent user**. For business-data errors during testing, grant the permission to the **automation user** only.

## Health Check

To verify the IDO runtime is reachable:

```
GET {SYTELINE_BASE_URL without /ido}/Ping.aspx
```

- `200` — IIS + IDO Request Web Service + IDO Runtime Service all running
- `503` — IDO Runtime Service is down
- `404` — IIS or IDO Request Web Service is unavailable
