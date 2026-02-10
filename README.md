# Syteline IDO REST API — Agent Reference

Reference documentation for the Infor Syteline (CloudSuite Industrial) IDO REST API, designed to be attached to any AI coding agent's context.

## Prerequisites

A dedicated Syteline automation user with the following configuration:

- **License:** Automation
- **Super User:** unchecked
- **Groups:** none
- **Password:** set (used via HTTP headers — never in URL paths)

### Required IDO Permissions (you can Paste Rows Append these into User Permissions!)

| IDO | Read | Execute | Edit | Insert | Delete | Update |
|---|---|---|---|---|---|---|
| `ActiveBGTasks` | Granted | Granted | Revoked | — | — | Revoked |
| `BGTaskDefinitions` | Granted | Granted | Revoked | — | — | Revoked |
| `BGTaskHistories` | Granted | Granted | Revoked | — | — | Revoked |
| `IdoCollections` | Granted | Granted | Revoked | — | — | Revoked |
| `IdoMethodParameters` | Granted | Granted | Revoked | — | — | Revoked |
| `IdoMethodResultSets` | Granted | Granted | Revoked | — | — | Revoked |
| `IdoMethods` | Granted | Granted | Revoked | — | — | Revoked |
| `IdoProperties` | Granted | Granted | Revoked | — | — | Revoked |
| `IdoPropertyClasses` | Granted | Granted | Revoked | — | — | Revoked |
| `IdoRequestExclusions` | Granted | Granted | Revoked | — | — | Revoked |
| `IdoSubColFilterProperties` | Granted | Granted | Revoked | — | — | Revoked |
| `IdoTables` | Granted | Granted | Revoked | — | — | Revoked |
| `ProcessErrorLogs` | Granted | Granted | Revoked | — | — | Revoked |
| `SchemaTypes` | Granted | Granted | Revoked | — | — | Revoked |
| `SLSites` | Granted | Granted | Revoked | — | — | Revoked |
| `SLSystemTypes` | Granted | Granted | Revoked | — | — | Revoked |
| `SqlColumns` | Granted | Granted | Revoked | — | — | Revoked |
| `SqlTableKeys` | Granted | Granted | Revoked | — | — | Revoked |
| `SqlTables` | Granted | Granted | Revoked | — | — | Revoked |

Additional IDO permissions will be required per task (e.g., `Read` on `SLItemprices` or `SVC_SLItemPrices` to query item pricing data). Grant the minimum necessary permissions for each integration.

---

## Quick Start

1. Copy `.env.template` to `.env` and fill in your credentials
2. Attach the `docs/` folder to your AI agent's context
3. Tell the agent what you need to do in Syteline

## Documentation

See [docs/README.md](docs/README.md) for the full table of contents.

| Doc | Purpose |
|---|---|
| [Authentication](docs/01_AUTHENTICATION.md) | Token acquisition via HTTP headers |
| [IDO Overview](docs/02_IDO_OVERVIEW.md) | What IDOs are and how the metadata system works |
| [Core Operations](docs/03_CORE_OPERATIONS.md) | Load, Invoke, Update — the three API verbs |
| [Background Tasks](docs/04_BGTASK_SUBMISSION.md) | Submitting and monitoring background tasks |
| [Discovery Guide](docs/05_DISCOVERY_GUIDE.md) | How to find any IDO, its properties, methods, and parameters |
| [Gotchas](docs/06_GOTCHAS.md) | Common pitfalls and troubleshooting |
