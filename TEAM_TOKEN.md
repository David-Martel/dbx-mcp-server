# Team Token Mode (Dropbox Business / Auricle)

This server supports **Dropbox Business team tokens** in addition to personal OAuth tokens.
Team token mode is enabled by setting the `DROPBOX_SELECT_USER` environment variable.

## How It Works

When `DROPBOX_SELECT_USER` is set, every API request adds the
`Dropbox-API-Select-User` header with the specified team member ID.
This allows a single set of app credentials (app key + app secret + refresh token)
to act on behalf of a specific member of the team.

The `get_account_info` tool detects the presence of `DROPBOX_SELECT_USER` and calls
`teamGetInfo` instead of `usersGetCurrentAccount`, returning team-level account details.

Implementation: `src/dbx-api.ts` lines 48-50 (header injection), lines 751-757
(`get_account_info` branch).

## Configuration for the Auricle Business App

The Auricle account uses the **Auricle Business Dropbox app**
(`david.martel@auricleinc.com`). The relevant environment variables are:

| Variable | Source | Description |
|---|---|---|
| `DROPBOX_APP_KEY` | `~/.claude/.env` (via `${env:DROPBOX_APP_KEY}` in mcp.json) | App key from the Dropbox App Console |
| `DROPBOX_APP_SECRET` | `~/.claude/.env` (via `${env:DROPBOX_APP_SECRET}` in mcp.json) | App secret from the Dropbox App Console |
| `DROPBOX_SELECT_USER` | `~/.claude/.env` (via `${env:DROPBOX_SELECT_USER}` in mcp.json) | Dropbox member ID (format: `dbmid:...`) |
| `TOKEN_ENCRYPTION_KEY` | `~/.claude/.env` (via `${env:TOKEN_ENCRYPTION_KEY}` in mcp.json) | AES-256 key for `.tokens.json` at-rest encryption |

## Differences from Personal Account Mode

| Aspect | Personal mode | Team / Business mode |
|---|---|---|
| `DROPBOX_SELECT_USER` | Not set | Set to `dbmid:...` member ID |
| `get_account_info` | `usersGetCurrentAccount` | `teamGetInfo` |
| Token scope | Single user | Acts on behalf of team member |
| Rate limits | Per-app | Per-team + per-member |

## Finding the Member ID (`dbmid:...`)

Run `npm run setup` and complete the OAuth flow, or call `get_account_info` once
authenticated — the response includes the member ID under `account_id`.
Alternatively, use the Dropbox Business API:
`POST https://api.dropboxapi.com/2/team/members/list/v2`
with a team admin token.

## Secret Source of Truth

All secrets (app key, app secret, select-user ID, encryption key) live exclusively in
`~/.claude/.env`, which is gitignored and never committed. The server-side `.env` in this
repository is intentionally absent (removed 2026-06-19 — see Phase 1 of the Dropbox
standardization proposal). The server receives all required variables from the mcp.json
`env` block via Claude Code's `${env:VAR}` expansion.

If you need to run the server outside Claude Code (e.g. via `npm start` or `npm run setup`),
export the variables in your shell before running:

```powershell
# Windows — source from the canonical file
Get-Content "C:\Users\david\.claude\.env" |
    Where-Object { $_ -notmatch '^\s*#' -and $_ -match '=' } |
    ForEach-Object {
        $k, $v = $_ -split '=', 2
        [System.Environment]::SetEnvironmentVariable($k.Trim(), $v.Trim(), 'Process')
    }
npm run setup
```

```bash
# Bash / Git Bash — source from the canonical file
set -a; source ~/.claude/.env; set +a
npm run setup
```
