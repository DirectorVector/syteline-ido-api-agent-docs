<!-- Use this file to provide workspace-specific custom instructions to Copilot. For more details, visit https://code.visualstudio.com/docs/copilot/copilot-customization#_use-a-githubcopilotinstructionsmd-file -->

## Syteline IDO REST API — Agent Reference Docs

This repository contains reference documentation for the Infor Syteline (CloudSuite Industrial) IDO REST API. The docs in `docs/` are designed to be attached to any AI coding agent's context to teach it how to interact with the Syteline API.

### Repository Structure
- `docs/` — The main documentation set (authentication, IDO concepts, API operations, discovery, gotchas)
- `.env.template` — Template for credential and token configuration

### Key Concepts
- **IDO (Intelligent Data Object):** The Mongoose framework middleware layer between clients and the database
- **Three API verbs:** Load (GET), Invoke (POST), Update (POST) — see `docs/03_CORE_OPERATIONS.md`
- **Self-describing system:** Use meta-IDOs (`IdoCollections`, `IdoProperties`, `IdoMethods`, `IdoMethodParameters`, `IdoTables`, `SqlColumns`) to discover any IDO's structure via the API itself
- **Background tasks:** Long-running operations submitted via `BGTaskDefinitions.BGTaskSubmit` — see `docs/04_BGTASK_SUBMISSION.md`

### Development Guidelines
- Use `curl` for all API examples (universal, easily convertible to any client)
- Pass credentials via HTTP headers, never in URL paths
- Use environment variables (`$SYTELINE_USERNAME`, `$SYTELINE_PASSWORD`) — never hardcode credentials
- Set small `recordcap` values during discovery to avoid overwhelming responses
- Verify API behavior by running actual requests against the test environment
