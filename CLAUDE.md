# CLAUDE.md

Dropbox file integration MCP server (v0.1.0) — provides file operations, metadata, search, and sharing via Dropbox API v2 with OAuth 2.0 PKCE authentication.

**Compare with:** `reference/mcp-server-dash` — both integrate Dropbox but serve different purposes. This repo does file CRUD operations; Dash does search and metadata retrieval.

## Build Commands

```bash
npm install
npm run build           # TypeScript compile to build/
npm test                # Jest unit tests
npm test:coverage       # Jest with coverage report
npm test:watch          # Jest in watch mode
npm test:integration    # Live API integration tests (requires valid tokens)
npm run inspector       # MCP Inspector against build/index.js
npm run setup           # Interactive OAuth setup wizard
npm start               # node build/src/index.js
```

## Environment Variables

**Single source of truth: `~/.claude/.env`** (gitignored; referenced from mcp.json via
`${env:VAR}` expansion). The per-server `.env` file has been removed (2026-06-19, Phase 1
standardization — see `.env.bak` for the last values). Do not recreate it; Claude Code
injects all required vars at launch from `~/.claude/.env`.

When running outside Claude Code (e.g. `npm run setup`), export vars manually:

```powershell
# Windows PowerShell
Get-Content "C:\Users\david\.claude\.env" |
    Where-Object { $_ -notmatch '^\s*#' -and $_ -match '=' } |
    ForEach-Object {
        $k, $v = $_ -split '=', 2
        [System.Environment]::SetEnvironmentVariable($k.Trim(), $v.Trim(), 'Process')
    }
npm run setup
```

Required variables (must be set in `~/.claude/.env`):
- `DROPBOX_APP_KEY` — From Dropbox App Console
- `DROPBOX_APP_SECRET` — From Dropbox App Console
- `TOKEN_ENCRYPTION_KEY` — 32+ chars, AES-256-GCM key for `.tokens.json`
- `DROPBOX_SELECT_USER` — Dropbox member ID (`dbmid:...`) for team/Business mode

`DROPBOX_REDIRECT_URI` is hardcoded to `http://localhost` in mcp.json.

Optional:
- `TOKEN_REFRESH_THRESHOLD_MINUTES` (default: 5)
- `MAX_TOKEN_REFRESH_RETRIES` (default: 3)
- `LOG_LEVEL` (default: info)
- `DROPBOX_RECYCLE_BIN_PATH`, `DROPBOX_MAX_DELETES_PER_DAY`, `DROPBOX_RETENTION_DAYS`

See `TEAM_TOKEN.md` for Business/team token documentation.

## MCP Tools Exposed (12 active)

| Tool | Purpose |
|------|---------|
| `list_files` | List directory contents |
| `upload_file` | Upload file (base64 content) |
| `download_file` | Download file |
| `get_file_content` | Read file contents |
| `get_file_metadata` | File/folder metadata |
| `search_file_db` | Search files and folders |
| `create_folder` | Create folder |
| `copy_item` | Copy file or folder |
| `move_item` | Move or rename |
| `safe_delete_item` | Delete with recycle bin support (prefer over legacy delete) |
| `get_sharing_link` | Create sharing links |
| `get_account_info` | Account information (uses `teamGetInfo` in team mode) |

Note: `delete_item` was removed from the advertised tool list (2026-06-19 — it is deprecated;
use `safe_delete_item` instead). The server-side handler is retained for backward
compatibility but the tool will not appear in MCP tool listings.

## Source Structure

```
src/
├── index.ts              # Entry point
├── dbx-server.ts         # MCP server + handler registration
├── dbx-api.ts            # Dropbox SDK wrapper (all API calls)
├── tool-definitions.ts   # MCP tool schemas
├── auth.ts               # OAuth 2.0 PKCE + token refresh
├── security-utils.ts     # Token encryption (AES)
├── config.ts             # Environment config loading
├── interfaces.ts         # TypeScript interfaces
├── setup.ts              # Interactive setup wizard
├── resource-handler.ts   # MCP resource handler
├── resource/             # Resource resolution
├── prompt-*.ts           # MCP prompt schemas + handlers
└── *-tokens.ts           # Token create/reset/exchange utilities
```

## Key Details

- **MCP SDK**: `@modelcontextprotocol/sdk` 0.6.0, **Dropbox SDK**: v10.34.0
- **Auth**: OAuth 2.0 with PKCE. Tokens stored encrypted in `.tokens.json` (gitignored, in server dir). Auto-refresh before expiry.
- **Team/Business mode**: Set `DROPBOX_SELECT_USER` to a `dbmid:...` member ID. See `TEAM_TOKEN.md`.
- **Safe delete**: Moves to `/.recycle_bin` with configurable retention (30 days default, 100 deletes/day max).
- **Logging**: Winston logger, level configurable via `LOG_LEVEL`. Never writes to stdout (protects JSON-RPC channel).
- **Secrets**: Single source = `~/.claude/.env` (no per-server `.env`). See Phase 1 of standardization proposal.
- **Testing**: Jest + Babel (ESM). Unit tests in `tests/dropbox/`, `tests/resource/`. Integration tests in `tests/dbx-operations.test.ts` (live API, requires valid tokens).
- **Phase 3 (planned)**: MCP SDK 0.6.0 → 1.x upgrade + Windows Credential Manager keychain for refresh token. See `docs/proposals/dropbox-standardized-tool-2026-06-19.md`.
