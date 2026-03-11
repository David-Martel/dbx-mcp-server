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

Required in `.env` (see `.env.example`):

```
DROPBOX_APP_KEY=           # From Dropbox App Console
DROPBOX_APP_SECRET=        # From Dropbox App Console
DROPBOX_REDIRECT_URI=      # e.g. http://localhost:3000/callback
TOKEN_ENCRYPTION_KEY=      # 32+ chars, encrypts stored tokens
```

Optional:
- `TOKEN_REFRESH_THRESHOLD_MINUTES` (default: 5)
- `MAX_TOKEN_REFRESH_RETRIES` (default: 3)
- `LOG_LEVEL` (default: info)
- `DBX_RECYCLE_BIN_PATH`, `DBX_MAX_DELETES_PER_DAY`, `DBX_RETENTION_DAYS`

## MCP Tools Exposed

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
| `safe_delete_item` | Delete with recycle bin support |
| `get_sharing_link` | Create sharing links |
| `get_account_info` | Account information |

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
- **Auth**: OAuth 2.0 with PKCE. Tokens stored encrypted in `.tokens.json`. Auto-refresh before expiry.
- **Safe delete**: Moves to `/.recycle_bin` with configurable retention (30 days default, 100 deletes/day max).
- **Logging**: Winston logger, level configurable via `LOG_LEVEL`.
- **Testing**: Jest + Babel (ESM). Unit tests in `tests/dropbox/`, `tests/resource/`. Integration tests in `tests/dbx-operations.test.ts` (live API, requires valid tokens).
